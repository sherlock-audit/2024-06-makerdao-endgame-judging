Fantastic Crepe Llama

Medium

# Any User with Auth Access Can Grant or Revoke Admin Access to/from Any Other User

## Summary
SDAO.sol:150
Any user with `auth` access can use the `rely` and `deny` functions to manage admin access for other users. The `wards[msg.sender] = 1` in the constructor only sets the deployer as an admin when the contract is first deployed. It does not restrict how admin access is managed after deployment.
## Vulnerability Detail
There's a security flaw in the access control system of the smart contract. Any user with admin (auth) access can revoke or deny admin privileges for other users. This could lead to unauthorized denial of service or the misuse of admin privileges
## Impact
An admin with malicious intent could revoke admin access from other legitimate admins, disrupting their ability to manage and control the contract.  Even though there is an initial admin set in the constructor, the original issue still persists. The initial admin can grant or revoke admin access to any other user, potentially leading to unauthorized control
## Code Snippet
     
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;

        wards[msg.sender] = 1;
        emit Rely(msg.sender);

        deploymentChainId = block.chainid;
        _DOMAIN_SEPARATOR = _calculateDomainSeparator(block.chainid);
    }

    * @notice Grants `usr` admin access to this contract.
     * @param usr The user address.
     */
    function rely(address usr) external auth {
        wards[usr] = 1;
        emit Rely(usr);
    }

    /**
     * @notice Revokes `usr` admin access from this contract.
     * @param usr The user address.
     */
    function deny(address usr) external auth {
        wards[usr] = 0;
        emit Deny(usr);
    }

Alice deploys the contract and automatically becomes an admin.
Alice grants admin access to Bob and Charlie.
Bob, a trusted admin, decides to act maliciously. He uses his admin privileges to grant admin access to Eve, a malicious actor.
Bob and Eve revoke Alice and Charlie’s admin access, leaving themselves as the only admins with full control over the contract.

Alice can indeed lose her admin control if another admin (with auth access) decides to revoke her privileges. This is because the original implementation allows any admin to grant or revoke admin rights for any other user, including the initial admin set in the constructor.

https://github.com/sherlock-audit/2024-06-makerdao-endgame/blob/dba30d7a676c20dfed3bda8c52fd6702e2e85bb1/endgame-toolkit/src/SDAO.sol#L153-L165
## Tool used

Manual Review

## Recommendation
 Use a multi-signature approach to require multiple approvals for critical changes.