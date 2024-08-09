Cuddly Inky Rat

High

# Unauthorized Burning of MRT tokens can lead loss of funds of a user and totalsupply

## Summary
The burn function in the LockstakeMkr contract has potential vulnerabilities due to missing explicit authorization checks. This can lead to unauthorized addresses being able to burn tokens, resulting in a significant loss of funds for token holders.
https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeMkr.sol#L124
## Vulnerability Detail
The burn function is designed to decrease the token balance of a specified address and reduce the total token supply. However, the function lacks an explicit authorization check to ensure that only authorized addresses can call it, just like the [mint](https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/main/lockstake/src/LockstakeMkr.sol#L114) function.

This omission can be exploited by malicious actors to burn tokens without the token holder's consent.

Alice has 100 tokens.
Bob has an allowance to spend 50 tokens on behalf of Alice.
If the burn function lacks a proper authorization check (assuming requires auth is not enforced), Bob or any other address could call the burn function directly.
Bob calls burn(Alice, 50), reducing Alice's balance by 50 tokens and the total supply by 50 tokens.

Since there is no authorization check, Bob could repeatedly call this function to burn tokens from Alice's balance without her consent, resulting in a significant loss of funds for Alice and also reducing the totalsupply significantly.

This can happen multiple times, affecting others, oneself, and the total supply of the contract in general.

## Impact
Unauthorized token burning can lead to unexpected loss of tokens for individual holders, decreasing their token balance and overall stake in the protocol.

The total token supply can be manipulated, affecting the overall market dynamics and trust in the token's stability.

## Code Snippet
```solidity
function burn(address from, uint256 value) external {
    // requires auth (assuming this is enforced elsewhere in the contract)
    uint256 balance = balanceOf[from];
    require(balance >= value, "LockstakeMkr/insufficient-balance");

    if (from != msg.sender) {
        uint256 allowed = allowance[from][msg.sender];
        if (allowed != type(uint256).max) {
            require(allowed >= value, "LockstakeMkr/insufficient-allowance");

            unchecked {
                allowance[from][msg.sender] = allowed - value;
            }
        }
    }

    unchecked {
        balanceOf[from] = balance - value;
        totalSupply     = totalSupply - value;
    }

    emit Transfer(from, address(0), value);
}
```

## Tool used
Manual Review

## Recommendation
Ensure that only authorized addresses can call the burn function as instructed in the [Docs](https://docs.makerdao.com/smart-contract-modules/mkr-module)
```solidity
function burn(address from, uint256 value) external auth{// requires auth 
        uint256 balance = balanceOf[from];
        require(balance >= value, "LockstakeMkr/insufficient-balance");

        if (from != msg.sender) {
            uint256 allowed = allowance[from][msg.sender];
            if (allowed != type(uint256).max) {
                require(allowed >= value, "LockstakeMkr/insufficient-allowance");

                unchecked {
                    allowance[from][msg.sender] = allowed - value;
                }
            }
        }

        unchecked {
            balanceOf[from] = balance - value; // note: we don't need overflow checks b/c require(balance >= value) and balance <= totalSupply
            totalSupply     = totalSupply - value;
        }

        emit Transfer(from, address(0), value);
    }
}

```