---
title: Arbitrum Inbox
description: Exploiting reinitialization vulnerability in bridge upgrade process
type: Report
network: [ethereum]
date: 2022-09-19
loss_usd: 400000
returned_usd: 0
tags: [access control, bridge, reinitialization]
subcategory: []
vulnerable_contracts:
  - "0x3e2198a77fc6b266082b92859092170763548730"
  - "0x4Dbd4fc535Ac27206064B68FfCf827b0A60BAB3f"
tokens_lost:
  - ETH
attacker_addresses: []
malicious_token: []
attack_block: []
reproduction_command: forge test --match-contract Report_ArbitrumInbox -vvv
attack_txs: []
sources:
  - title: Riptide Writeup
    url: https://medium.com/@0xriptide/hackers-in-arbitrums-inbox-ca23272641a2
---

## Detailed Description

The Inbox is part of the Arbitrum Bridge between ETH and Arbitrum. The Inbox takes some messages and forwards them to the Bridge contract.

To do this, it takes a reference to the bridge address in its `initialize`. As hinted by this method, the whole contract is behind an [Universal Upgradable Proxy](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable).

The problem can be found in the initialization of the implementation contract. While the `initialize` method is correctly protected by a `initializer` guard, which makes sure that this method can only be called once, it does so by [using flags which are in position `0x00` and `0x01` in the storage](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/proxy/utils/Initializable.sol). But the `postUpgradeInit` method, called after initialization, deletes the first three slots!

This results in the contract being marked as not-initialized and deleting its reference to the `_sequencerInbox`. This last variable is not used, so it's not actually a problem. But now that the contract is marked as not-initialized, anyone can call `initialize` again with their own `_bridge` address!

```solidity
    function initialize(IBridge _bridge, ISequencerInbox _sequencerInbox)
        external
        initializer
        onlyDelegated
    {
        bridge = _bridge;
        sequencerInbox = _sequencerInbox;
        allowListEnabled = false;
        __Pausable_init();
    }

    /// @dev function to be called one time during the inbox upgrade process
    /// this is used to fix the storage slots
    function postUpgradeInit(IBridge _bridge) external onlyDelegated onlyProxyOwner {
        uint8 slotsToWipe = 3;
        for (uint8 i = 0; i < slotsToWipe; i++) {
            assembly {
                sstore(i, 0)
            }
        }
        allowListEnabled = false;
        bridge = _bridge;
    }
```

An attacker can quite easily exploit this by taking advantage of a call the Inbox makes to the Bridge which sends value, specifically to the method `enqueueDelayedMessage()` (follow `depositEth` in the vulnerable contract for the full path). An attacker could have forwarded all ETH deposits from the inbox to their own evil contract.

Maybe more interesting than the exploit itself is how the vulnerability came to be. Two different commits where needed to break the contract:

1. [c33765fa66d74733ab740c0f0cbdf27a05d1d985](https://github.com/OffchainLabs/nitro/commit/c33765fa66d74733ab740c0f0cbdf27a05d1d985) on Feb 18, 2022 introduced the wiping of the slots. This nevertheless was not vulnerable: even though the slots where wiped, they were _replaced_ by another flag in the `initialize` method: `if(address(bridge) != address(0)) revert AlreadyInit();`. This explains why it was safe to delete these slots, as they are not needed anymore.
2. [2631e1e0a4767ef95898ccdca727d61fa1353031](https://github.com/OffchainLabs/nitro/commit/2631e1e0a4767ef95898ccdca727d61fa1353031#diff-de26d64a8be62f56073b95f0590061da9411001beaa20cc71ebdb2316303430cR58) on Aug 1, 2022. 6 months after the commit that removed the slots, it was likely forgotten that the `address(bridge)` check replaced the `initialize` flags, and the check was removed; making the contract vulnerable.

## Possible mitigations

- Be careful when wiping up slots.
- Be careful when removing "useless" checks.
- Test deploy conditions, like `should not be able to reinitialize contract`
