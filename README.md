# Canton Patterns

Reusable Daml contract patterns for Canton Network developers. Think "OpenZeppelin for Daml" — production-ready building blocks for common smart contract use cases.

## Patterns

### AccessControl

Role-based access control with Admin, Operator, and Observer roles.

- **RoleAssignment** — links a party to a role, governed by an admin
- **RoleRequest** — party requests a role; admin approves or rejects
- **RoleGrant** — admin offers a role; grantee accepts or it's withdrawn

```daml
import Patterns.AccessControl

-- Admin grants Operator role to alice
grantCid <- submit admin do
  createCmd RoleGrant with admin; grantee = alice; role = Operator

-- Alice accepts
submit alice do exerciseCmd grantCid Accept
```

### Escrow

Two-party escrow with deposit, release, dispute, and timeout.

- **EscrowProposal** — payer proposes; receiver accepts to create the escrow
- **Escrow** — state machine: AwaitingDeposit → Funded → Released/Disputed

```daml
import Patterns.Escrow

proposalCid <- submit payer do
  createCmd EscrowProposal with payer; receiver; amount = 1000.0; asset = "USDC"; deadline

escrowCid <- submit receiver do exerciseCmd proposalCid AcceptProposal
fundedCid <- submit payer do exerciseCmd escrowCid Deposit
submit receiver do exerciseCmd fundedCid Release
```

### Multisig

M-of-N multisig approval — collect signatures until a threshold, then execute.

- **MultisigProposal** — proposer creates; signers sign; execute when threshold met
- **MultisigExecution** — receipt of a completed multisig action

```daml
import Patterns.Multisig

proposalCid <- submit proposer do
  createCmd MultisigProposal with
    proposer; signers = [alice, bob, charlie]; minSigners = 2
    description = "Transfer funds"; approvals = []

signed1 <- submit alice do exerciseCmd proposalCid Sign with signer = alice
signed2 <- submit bob do exerciseCmd signed1 Sign with signer = bob
submit proposer do exerciseCmd signed2 Execute
```

### Vesting

Token vesting with cliff and linear unlock schedule.

- **VestingProposal** — grantor proposes; beneficiary accepts
- **VestingSchedule** — tracks vesting with claim and revoke capabilities

```daml
import Patterns.Vesting

proposalCid <- submit grantor do
  createCmd VestingProposal with
    grantor; beneficiary; totalAmount = 10000.0
    startTime; cliffTime; endTime

scheduleCid <- submit beneficiary do exerciseCmd proposalCid AcceptVesting
submit beneficiary do exerciseCmd scheduleCid Claim with claimAmount = 2500.0; currentTime
```

### Timelock

Time-locked actions that can only execute after a deadline.

- **TimelockProposal** — owner proposes; beneficiary accepts
- **TimelockAction** — executable only after unlock time
- **TimelockReceipt** — proof of execution

```daml
import Patterns.Timelock

timelockCid <- submit owner do
  createCmd TimelockAction with
    owner; beneficiary; description = "Release payment"
    payload = "1000 USDC"; unlockTime

-- After unlockTime has passed:
submit beneficiary do exerciseCmd timelockCid ExecuteAction
```

### Voting

Simple proposal + vote + tally pattern.

- **Ballot** — proposer creates with eligible voter list and deadline
- **BallotResult** — outcome after tallying (yes/no/abstain counts, passed flag)

```daml
import Patterns.Voting

ballotCid <- submit proposer do
  createCmd Ballot with
    proposer; description = "Upgrade to v2"
    voters = [alice, bob, charlie]; votes = []; deadline

voted1 <- submit alice do exerciseCmd ballotCid CastVote with voter = alice; vote = Yes
voted2 <- submit bob do exerciseCmd voted1 CastVote with voter = bob; vote = Yes
-- After deadline:
submit proposer do exerciseCmd voted2 Tally
```

## Project Structure

```
canton-patterns/
├── daml.yaml
├── src/
│   └── Patterns/
│       ├── AccessControl.daml
│       ├── Escrow.daml
│       ├── Multisig.daml
│       ├── Vesting.daml
│       ├── Timelock.daml
│       └── Voting.daml
├── test/
│   ├── TestAccessControl.daml
│   ├── TestEscrow.daml
│   ├── TestMultisig.daml
│   ├── TestVesting.daml
│   ├── TestTimelock.daml
│   └── TestVoting.daml
└── .github/workflows/ci.yml
```

## Getting Started

### Prerequisites

- [Daml SDK 2.9.0](https://docs.daml.com/getting-started/installation.html)

### Build

```bash
daml build
```

### Test

```bash
daml test
```

## License

BSD-0-Clause — see [LICENSE](LICENSE).
