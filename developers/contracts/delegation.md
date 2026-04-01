# Delegation

The Delegation contract is the backbone of Cap's financial guarantee market. It tracks how much slashable collateral Delegators have committed to each Operator across EigenLayer and Symbiotic, routes restaker interest rewards to Delegators, and executes slashing when an Operator is liquidated. Each Operator is assigned a network — either the EigenLayer AVS or the Symbiotic network — and their borrowing capacity is determined by the USD value of collateral their Delegators have staked on that network.

## Mechanics

### Coverage and Borrowing Capacity

Each Operator's borrowing capacity is derived from their **coverage** — the minimum of several collateral snapshots designed to prevent gaming:

```
coverage = min(
    currentEpochCoverage,       // collateral at the current epoch boundary
    slashableCollateral,        // collateral at slashTimestamp (most conservative)
    currentDelegation,          // live delegation value
    lastBorrowMinusOneDelegation, // delegation at (lastBorrow - 1 second)
    coverageCap                 // admin-set USD cap
)
```

All USD values use 8 decimals. Coverage is queried from the network middleware contract (Symbiotic or EigenLayer), which aggregates across all Delegators backing that Operator.

The Lender computes `maxBorrowable = coverage × ltv / 1e27`. The **LTV buffer** (default 5%) ensures `liquidationThreshold ≥ ltv + ltvBuffer`, maintaining a gap between the borrow limit and the liquidation trigger.

### Slash Timestamp

The slash timestamp is used to determine which snapshot of collateral is slashable during a liquidation. It is the most recent of:
- `(epoch - 1) × epochDuration` — the previous epoch boundary
- `lastBorrow - 1 second` — one second before the Operator's most recent borrow

This conservative snapshot prevents Delegators from front-running liquidations by rapidly removing collateral after an Operator borrows.

### Reward Distribution

When an Operator repays restaker interest, the Lender transfers the asset to the Delegation contract and calls `distributeRewards`. The contract forwards the full balance to the Operator's network middleware, which distributes proportionally to each Delegator based on their collateral share. If `coverage == 0` (no active Delegators), rewards go to the `feeRecipient` fallback address.

### Slash Execution

When the Lender liquidates an Operator, it calls `slash(_agent, _liquidator, _amount)`:

1. The `slashTimestamp` is computed for the Operator.
2. The network middleware's `slashableCollateral` is queried at that timestamp.
3. A `slashShare` ratio is computed: `_amount / networkSlashableCollateral` (capped at 100%).
4. The network middleware's `slash` function is called with the share — which propagates to the underlying EigenLayer/Symbiotic slashing mechanisms.

### Epochs

The epoch system provides a time window during which collateral state is considered stable for slashing purposes. `epochDuration` is set at initialization. The current epoch is `block.timestamp / epochDuration`. Symbiotic vaults use epoch-based accounting for slashing; the Delegation contract aligns with this by using the previous epoch boundary as one of the slash timestamp candidates.

---

## Architecture

### Delegation (core)

#### `slash(address _agent, address _liquidator, uint256 _amount)`

```solidity
function slash(address _agent, address _liquidator, uint256 _amount) external;
```

Access-controlled (Lender only). Executes a slash against the Operator's network. Reverts with `NoSlashableCollateral` if no slashable collateral exists at the slash timestamp.

| Parameter    | Type      | Description                                              |
|--------------|-----------|----------------------------------------------------------|
| `_agent`     | `address` | Operator being liquidated                                |
| `_liquidator`| `address` | Address receiving the slashed collateral value           |
| `_amount`    | `uint256` | USD value (8 decimals) to slash                          |

---

#### `distributeRewards(address _agent, address _asset)`

```solidity
function distributeRewards(address _agent, address _asset) external;
```

Permissionless (called by the Lender). Transfers the contract's full balance of `_asset` to the Operator's network for distribution to Delegators. Falls back to `feeRecipient` if coverage is zero.

---

#### `addAgent(address _agent, address _network, uint256 _ltv, uint256 _liquidationThreshold)`

```solidity
function addAgent(address _agent, address _network, uint256 _ltv, uint256 _liquidationThreshold) external;
```

Access-controlled. Registers a new Operator with their network, LTV, and liquidation threshold. Both values are in ray format (`1e27 = 100%`). Enforces `liquidationThreshold ≥ ltv + ltvBuffer` and `liquidationThreshold ≤ 1e27`.

---

#### `modifyAgent(address _agent, uint256 _ltv, uint256 _liquidationThreshold)`

```solidity
function modifyAgent(address _agent, uint256 _ltv, uint256 _liquidationThreshold) external;
```

Access-controlled. Updates an existing Operator's LTV and liquidation threshold. Same validation as `addAgent`.

---

#### `registerNetwork(address _network)`

```solidity
function registerNetwork(address _network) external;
```

Access-controlled. Registers a new network middleware contract (EigenLayer AVS or Symbiotic network). An Operator can only be assigned to a registered network.

---

#### View Functions

| Function | Returns | Description |
|----------|---------|-------------|
| `coverage(address _agent)` | `uint256` (USD, 8 dec) | Conservative collateral coverage used for borrowing capacity |
| `slashableCollateral(address _agent)` | `uint256` (USD, 8 dec) | Collateral slashable at the current slash timestamp |
| `ltv(address _agent)` | `uint256` (ray) | Maximum LTV ratio for the Operator |
| `liquidationThreshold(address _agent)` | `uint256` (ray) | Health ratio below which liquidation can be triggered |
| `slashTimestamp(address _agent)` | `uint48` | Most conservative timestamp for collateral snapshots |
| `epoch()` | `uint256` | Current epoch index (`block.timestamp / epochDuration`) |
| `epochDuration()` | `uint256` | Epoch length in seconds |
| `ltvBuffer()` | `uint256` (ray) | Minimum gap between LTV and liquidation threshold |
| `coverageCap(address _agent)` | `uint256` (USD, 8 dec) | Admin-set USD cap on an Operator's coverage |
| `collateral(address _agent)` | `address` | Collateral token address for the Operator's network |
| `networks(address _agent)` | `address` | Network middleware contract for the Operator |
| `agents()` | `address[]` | All registered Operator addresses |

---

#### Admin Functions

| Function | Description |
|----------|-------------|
| `setLastBorrow(address)` | Records borrow timestamp (Lender only) |
| `setLtvBuffer(uint256)` | Update the LTV safety buffer (ray, must be > 1%) |
| `setFeeRecipient(address)` | Update the fallback reward recipient |
| `setCoverageCap(address, uint256)` | Set a USD cap on an Operator's coverage |

---

### Data Structures

#### `DelegationStorage`

```solidity
struct DelegationStorage {
    EnumerableSet.AddressSet agents;
    mapping(address => AgentData) agentData;
    EnumerableSet.AddressSet networks;
    address oracle;
    uint256 epochDuration;
    uint256 ltvBuffer;
    address feeRecipient;
    mapping(address => uint256) coverageCap;
}
```

| Field          | Description                                                               |
|----------------|---------------------------------------------------------------------------|
| `agents`       | Set of all registered Operators                                           |
| `agentData`    | Per-Operator config: network, LTV, liquidation threshold, last borrow     |
| `networks`     | Set of all registered network middleware contracts                        |
| `oracle`       | Oracle contract used for price feeds (passed to network middleware)       |
| `epochDuration`| Epoch length in seconds — aligns with Symbiotic's slash window            |
| `ltvBuffer`    | Minimum gap between LTV and liquidation threshold (ray, default 5%)      |
| `feeRecipient` | Receives rewards when an Operator has no active coverage                  |
| `coverageCap`  | Per-Operator USD cap on coverage                                          |

#### `AgentData`

```solidity
struct AgentData {
    address network;
    uint256 ltv;
    uint256 liquidationThreshold;
    uint256 lastBorrow;
}
```

| Field                 | Description                                                  |
|-----------------------|--------------------------------------------------------------|
| `network`             | Network middleware contract (EigenLayer or Symbiotic)        |
| `ltv`                 | Maximum loan-to-value ratio in ray (`1e27 = 100%`)           |
| `liquidationThreshold`| Health threshold triggering liquidation in ray               |
| `lastBorrow`          | `block.timestamp` of the Operator's most recent borrow       |

---

### EigenLayer Integration

The EigenLayer integration consists of three contracts:

**`EigenAgentManager`** — The per-Operator contract deployed when an Operator registers on EigenLayer. It tracks which EigenLayer operator address backs this Cap Operator and queries the AVS's allocation system for slashable collateral.

**`EigenOperator`** — Manages an EigenLayer operator's registration into the Cap AVS (opt-in to the AVS, register operator sets, manage allocation delays).

**`EigenServiceManager`** — The EigenLayer AVS entry point. Registered as the `ServiceManager` on EigenLayer. Handles slash execution by interacting with EigenLayer's `AllocationManager`.

Key EigenLayer concepts:
- Operators opt into the Cap AVS via `EigenServiceManager`
- Delegators stake into EigenLayer strategies (e.g. stETH, EIGEN) and delegate to an EigenLayer operator
- Slashable collateral is the USD value of delegated stake that Cap can slash via the `AllocationManager`

---

### Symbiotic Integration

The Symbiotic integration consists of five contracts:

**`SymbioticAgentManager`** — Tracks collateral for Operators backed by Symbiotic Delegators. Queries Symbiotic vault/delegator contracts for slashable amounts.

**`SymbioticOperator`** — Handles Symbiotic operator registration, opt-in to vaults and networks.

**`SymbioticNetwork`** — The Symbiotic network contract. Cap is registered as a Symbiotic network; this contract manages network-level configuration.

**`SymbioticNetworkMiddleware`** — The primary integration point. Implements `coverage`, `slashableCollateral`, `slash`, and `distributeRewards` for the Symbiotic path. Aggregates collateral across all Symbiotic vaults that have allocated to Cap.

**`CapSymbioticVaultFactory`** — Deploys new Symbiotic collateral vaults compatible with Cap's network. Delegators use these vaults to stake collateral that backs Cap Operators.

---

## Usage Examples

### 1. Querying an Operator's borrowing position

```solidity
interface IDelegation {
    function coverage(address _agent) external view returns (uint256);
    function slashableCollateral(address _agent) external view returns (uint256);
    function ltv(address _agent) external view returns (uint256);
    function liquidationThreshold(address _agent) external view returns (uint256);
}

address delegation = 0x...;
address operator   = 0x...;

uint256 coverageUsd  = IDelegation(delegation).coverage(operator);
// coverageUsd in USD with 8 decimals, e.g. 1_000_000e8 = $1M

uint256 maxBorrowUsd = coverageUsd * IDelegation(delegation).ltv(operator) / 1e27;
// maxBorrowUsd = how much USD value the Operator can borrow in total
```

### 2. Admin: onboarding a new Operator

```solidity
interface IDelegation {
    function registerNetwork(address _network) external;
    function addAgent(address _agent, address _network, uint256 _ltv, uint256 _liquidationThreshold) external;
    function setCoverageCap(address _agent, uint256 _coverageCap) external;
}

address delegation = 0x...;
address network    = 0x...; // SymbioticNetworkMiddleware or EigenAgentManager
address operator   = 0x...;

// 1. Register the network if not already registered
IDelegation(delegation).registerNetwork(network);

// 2. Add the Operator: 80% LTV, 90% liquidation threshold, both in ray
IDelegation(delegation).addAgent(operator, network, 0.8e27, 0.9e27);

// 3. Set a $10M USD coverage cap
IDelegation(delegation).setCoverageCap(operator, 10_000_000e8);
```

### 3. Checking slash eligibility before liquidation

```solidity
interface IDelegation {
    function slashableCollateral(address _agent) external view returns (uint256);
    function slashTimestamp(address _agent) external view returns (uint48);
}

interface ILender {
    function agent(address _agent) external view returns (
        uint256, uint256 slashable, uint256 debt, uint256, uint256, uint256 health
    );
}

address delegation = 0x...;
address lender     = 0x...;
address operator   = 0x...;

(, uint256 slashable, uint256 debt,,, uint256 health) = ILender(lender).agent(operator);
uint256 slashableRaw = IDelegation(delegation).slashableCollateral(operator);

// health < 1e27 = undercollateralised, liquidation can be opened
// slashableRaw > 0 = collateral exists to cover the liquidation
```
