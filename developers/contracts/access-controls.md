# Access Controls

Cap uses a two-contract access control system. The `AccessControl` contract is the single authority that maps `(function selector, contract address)` pairs to addresses that are allowed to call them. The `Access` abstract contract is a thin mixin inherited by every Cap contract — it provides the `checkAccess` modifier that calls out to `AccessControl` to validate each privileged operation.

This architecture gives Cap fine-grained, per-function permissioning across the entire protocol with a single upgradeable registry.

## Mechanics

### Role Derivation

Every permission in the system is represented as an OpenZeppelin role — a `bytes32` identifier. The role for a given `(selector, contract)` pair is computed as:

```solidity
roleId = bytes32(selector) | bytes32(uint256(uint160(contract)));
```

The selector occupies the top 4 bytes; the contract address fills the lower 20 bytes. This means two functions with the same selector on different contracts get distinct roles automatically.

### Granting and Revoking Access

`grantAccess` and `revokeAccess` are themselves access-controlled — only addresses holding the `role(grantAccess.selector, AccessControl)` role can modify permissions. The deployer (initial admin) receives:
- `DEFAULT_ADMIN_ROLE` (OpenZeppelin admin role)
- The role to call `grantAccess` on `AccessControl`
- The role to call `revokeAccess` on `AccessControl`

Self-revocation of the `revokeAccess` role is blocked — an admin cannot accidentally lock themselves out by revoking their own revocation privilege.

### `checkAccess` Modifier

Every privileged function in Cap contracts uses the `checkAccess` modifier:

```solidity
modifier checkAccess(bytes4 _selector) {
    _checkAccess(_selector);
    _;
}
```

`_checkAccess` calls `IAccessControl.checkAccess(_selector, address(this), msg.sender)` on the stored `accessControl` address. If the caller does not hold the required role, the call reverts with `AccessDenied`. There are no owner or operator patterns — all access is role-based.

The `bytes4(0)` selector is conventionally used for UUPS upgrade authorization (`_authorizeUpgrade`) across all upgradeable Cap contracts.

---

## Architecture

### AccessControl

```solidity
contract AccessControl is IAccessControl, UUPSUpgradeable, AccessControlEnumerableUpgradeable
```

The central permissions registry. UUPS upgradeable; upgrade requires `DEFAULT_ADMIN_ROLE`.

#### `initialize(address _admin)`

```solidity
function initialize(address _admin) external initializer;
```

Bootstraps the contract. Grants `DEFAULT_ADMIN_ROLE` and both `grantAccess` / `revokeAccess` roles to `_admin`.

---

#### `grantAccess(bytes4 _selector, address _contract, address _address)`

```solidity
function grantAccess(bytes4 _selector, address _contract, address _address) external;
```

Grants `_address` permission to call `_selector` on `_contract`. Requires the caller to hold the `grantAccess` role on this `AccessControl` contract.

| Parameter | Type | Description |
|-----------|------|-------------|
| `_selector` | `bytes4` | Function selector being permissioned |
| `_contract` | `address` | Contract on which the function lives |
| `_address` | `address` | Address being granted access |

---

#### `revokeAccess(bytes4 _selector, address _contract, address _address)`

```solidity
function revokeAccess(bytes4 _selector, address _contract, address _address) external;
```

Revokes `_address`'s permission to call `_selector` on `_contract`. Cannot be used to revoke the caller's own `revokeAccess` role (reverts with `CannotRevokeSelf`).

---

#### `checkAccess(bytes4 _selector, address _contract, address _caller)`

```solidity
function checkAccess(bytes4 _selector, address _contract, address _caller)
    external view returns (bool hasAccess);
```

Verifies that `_caller` holds the role for `(_selector, _contract)`. Reverts (via OpenZeppelin `_checkRole`) if they do not. Returns `true` if access is granted.

---

#### `role(bytes4 _selector, address _contract)`

```solidity
function role(bytes4 _selector, address _contract) public pure returns (bytes32 roleId);
```

Computes the role identifier for a `(selector, contract)` pair. Pure function — can be called off-chain to compute role IDs for `grantAccess` / `revokeAccess`.

---

### Access (mixin)

```solidity
abstract contract Access is IAccess, Initializable, AccessStorageUtils
```

Inherited by every Cap contract that has privileged functions. Stores the `accessControl` address and exposes the `checkAccess` modifier.

#### `__Access_init(address _accessControl)`

Called from each contract's `initialize`. Stores the `AccessControl` contract address in ERC-7201 namespaced storage.

---

## Usage Examples

### 1. Granting a keeper permission to call `realizeInterest`

```solidity
interface IAccessControl {
    function grantAccess(bytes4 _selector, address _contract, address _address) external;
    function role(bytes4 _selector, address _contract) external pure returns (bytes32);
}

address accessControl = 0x...;
address lender        = 0x...;
address keeper        = 0x...;

bytes4 selector = ILender.realizeInterest.selector;

// Grant the keeper permission to call realizeInterest on the Lender
IAccessControl(accessControl).grantAccess(selector, lender, keeper);
```

---

### 2. Revoking a deprecated admin address

```solidity
interface IAccessControl {
    function revokeAccess(bytes4 _selector, address _contract, address _address) external;
    function role(bytes4 _selector, address _contract) external pure returns (bytes32);
}

address accessControl = 0x...;
address delegation    = 0x...;
address oldAdmin      = 0x...;

// Revoke permission to call addAgent on the Delegation contract
bytes4 selector = IDelegation.addAgent.selector;
IAccessControl(accessControl).revokeAccess(selector, delegation, oldAdmin);
```

---

### 3. Checking access off-chain (role ID calculation)

```solidity
interface IAccessControl {
    function role(bytes4 _selector, address _contract) external pure returns (bytes32 roleId);
    function hasRole(bytes32 role, address account) external view returns (bool);
}

address accessControl = 0x...;
address vault         = 0x...;
address addr          = 0x...;

bytes4 selector = IVault.addAsset.selector;
bytes32 roleId  = IAccessControl(accessControl).role(selector, vault);

bool canAddAsset = IAccessControl(accessControl).hasRole(roleId, addr);
```
