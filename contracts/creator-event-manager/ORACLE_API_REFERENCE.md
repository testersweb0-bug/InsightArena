# Oracle API Reference

## Quick Reference

| Function               | Purpose                            | Returns            | Errors                                                    |
| ---------------------- | ---------------------------------- | ------------------ | --------------------------------------------------------- |
| `verify_event_winners` | Identify and store perfect scorers | u32 (winner count) | Paused, EventNotFound, EventCancelled, MatchesNotComplete |
| `get_event_winners`    | Retrieve sorted winners list       | Vec<Winner>        | EventNotFound                                             |
| `get_user_score`       | Calculate user's score             | (u32, u32)         | EventNotFound, Overflow                                   |
| `get_creation_fee`     | Get event creation fee             | i128               | CreationFeeNotSet                                         |

---

## Function Signatures

### verify_event_winners

```rust
pub fn verify_event_winners(
    env: Env,
    caller: Address,
    event_id: u64,
) -> u32
```

**Description**: Verify and record all perfect scorers for an event.

**Parameters**:

- `env`: Soroban environment
- `caller`: Address calling the function (requires auth)
- `event_id`: Event to verify winners for

**Returns**: Number of winners identified

**Panics**:

- `"contract_paused"` - Contract is paused
- `"event_not_found"` - Event doesn't exist
- `"event_cancelled"` - Event is cancelled
- `"matches_not_complete"` - Not all matches resolved
- `"overflow"` - Arithmetic overflow

**Preconditions**:

- Event must exist
- Event must be active (not cancelled)
- All matches must have results submitted
- At least one participant must exist

**Postconditions**:

- Winners are stored in `EventWinners(event_id)`
- `WinnersVerified` event is emitted
- TTL is extended on all accessed storage

**Example**:

```rust
let winner_count = contract.verify_event_winners(caller, 1)?;
assert!(winner_count >= 0);
```

---

### get_event_winners

```rust
pub fn get_event_winners(
    env: Env,
    event_id: u64,
) -> Vec<Winner>
```

**Description**: Retrieve the list of winners for an event, sorted by completion time.

**Parameters**:

- `env`: Soroban environment
- `event_id`: Event to retrieve winners for

**Returns**: Vec<Winner> sorted by completion_time ascending

**Panics**:

- `"event_not_found"` - Event doesn't exist (returns empty Vec instead)

**Sorting**:

- Primary: `completion_time` ascending (earliest first)
- Tiebreaker: None (completion_time is unique per user)

**Example**:

```rust
let winners = contract.get_event_winners(1)?;
for (rank, winner) in winners.iter().enumerate() {
    println!("#{}: {} - {}/{}", rank + 1, winner.user, winner.total_correct, winner.total_matches);
}
```

---

### get_user_score

```rust
pub fn get_user_score(
    env: Env,
    user: Address,
    event_id: u64,
) -> (u32, u32)
```

**Description**: Calculate a user's score (correct predictions) for an event.

**Parameters**:

- `env`: Soroban environment
- `user`: User address to score
- `event_id`: Event to calculate score for

**Returns**: Tuple `(correct_count, total_matches)`

- `correct_count`: Number of correct predictions
- `total_matches`: Total matches in event

**Panics**:

- `"event_not_found"` - Event doesn't exist
- `"overflow"` - Arithmetic overflow

**Behavior**:

- Unresolved predictions are not counted
- Returns (0, total_matches) if user has no predictions
- Returns (0, 0) if event has no matches

**Example**:

```rust
let (correct, total) = contract.get_user_score(user, 1)?;
let accuracy = if total > 0 { (correct * 100) / total } else { 0 };
println!("Score: {}/{} ({}%)", correct, total, accuracy);
```

---

### get_creation_fee

```rust
pub fn get_creation_fee(env: Env) -> i128
```

**Description**: Retrieve the current XLM fee required to create an event.

**Parameters**:

- `env`: Soroban environment

**Returns**: Creation fee in stroops (i128)

**Behavior**:

- Returns 0 if fee is not set (should not happen after init)
- No panics - always returns a value

**Example**:

```rust
let fee_stroops = contract.get_creation_fee();
let fee_xlm = fee_stroops as f64 / 10_000_000.0;
println!("Fee: {} XLM", fee_xlm);
```

---

## Data Structures

### Winner

```rust
pub struct Winner {
    pub user: Address,              // Wallet address
    pub event_id: u64,              // Event ID
    pub total_correct: u32,         // Correct predictions
    pub total_matches: u32,         // Total matches
    pub completion_time: u64,       // Last prediction timestamp
    pub verified_at: u64,           // Verification timestamp
}
```

**Methods**:

- `new(user, event_id, total_correct, total_matches, completion_time, verified_at) -> Winner`
- `get_accuracy_percentage() -> u32` - Returns 0-100
- `outranks(other) -> bool` - Leaderboard comparison

---

## Error Codes

| Code | Name               | Description              |
| ---- | ------------------ | ------------------------ |
| 1    | Paused             | Contract is paused       |
| 2    | EventNotFound      | Event doesn't exist      |
| 3    | EventCancelled     | Event is cancelled       |
| 4    | MatchesNotComplete | Not all matches resolved |
| 5    | CreationFeeNotSet  | Fee not initialized      |
| 6    | Overflow           | Arithmetic overflow      |

---

## Events

### WinnersVerified

Emitted when `verify_event_winners` completes successfully.

```rust
env.events().publish(
    (Symbol::new(env, "event"), Symbol::new(env, "winners_verified")),
    (event_id, winner_count),
);
```

**Fields**:

- `event_id`: u64 - Event ID
- `winner_count`: u32 - Number of winners

---

## Storage Keys

| Key                               | Type        | TTL    | Description           |
| --------------------------------- | ----------- | ------ | --------------------- |
| `EventWinners(event_id)`          | Vec<Winner> | 1 year | Winners for event     |
| `Event(event_id)`                 | Event       | 1 year | Event metadata        |
| `Match(match_id)`                 | Match       | 1 year | Match data            |
| `Prediction(prediction_id)`       | Prediction  | 1 year | User prediction       |
| `UserPredictions(user, event_id)` | Vec<u64>    | 1 year | User's prediction IDs |
| `EventMatches(event_id)`          | Vec<u64>    | 1 year | Event's match IDs     |

---

## Usage Patterns

### Pattern 1: Verify Winners After Event Completion

```rust
// 1. Ensure all matches are resolved
let event = contract.get_event(event_id)?;
assert!(event.match_count > 0);

// 2. Verify winners
let winner_count = contract.verify_event_winners(caller, event_id)?;

// 3. Retrieve and display winners
let winners = contract.get_event_winners(event_id)?;
for winner in winners.iter() {
    println!("Winner: {} ({}/{})", winner.user, winner.total_correct, winner.total_matches);
}
```

### Pattern 2: Display User Score

```rust
let (correct, total) = contract.get_user_score(user, event_id)?;
let accuracy = if total > 0 { (correct * 100) / total } else { 0 };
println!("Your score: {}/{} ({}%)", correct, total, accuracy);
```

### Pattern 3: Leaderboard Display

```rust
let winners = contract.get_event_winners(event_id)?;
println!("Leaderboard for Event {}", event_id);
println!("Rank | User | Score | Accuracy | Time");
for (rank, winner) in winners.iter().enumerate() {
    let accuracy = winner.get_accuracy_percentage();
    println!(
        "#{} | {} | {}/{} | {}% | {}",
        rank + 1,
        winner.user,
        winner.total_correct,
        winner.total_matches,
        accuracy,
        winner.completion_time
    );
}
```

### Pattern 4: Check Event Creation Cost

```rust
let fee = contract.get_creation_fee();
let fee_xlm = fee as f64 / 10_000_000.0;
println!("Creating an event costs {} XLM", fee_xlm);
```

---

## Constraints & Limits

| Constraint                 | Value    | Notes                                   |
| -------------------------- | -------- | --------------------------------------- |
| Max participants per event | u32::MAX | Limited by participant_count field      |
| Max matches per event      | u32::MAX | Limited by match_count field            |
| Max winners per event      | u32::MAX | Limited by Vec capacity                 |
| Accuracy percentage        | 0-100    | Clamped to valid range                  |
| Completion time            | u64      | Unix timestamp in seconds               |
| Creation fee               | i128     | In stroops (1 XLM = 10,000,000 stroops) |

---

## Performance Characteristics

| Function             | Time Complexity | Space Complexity | Notes                                                  |
| -------------------- | --------------- | ---------------- | ------------------------------------------------------ |
| verify_event_winners | O(P × M × Pred) | O(W)             | P=participants, M=matches, Pred=predictions, W=winners |
| get_event_winners    | O(W² log W)     | O(W)             | Insertion sort on winners                              |
| get_user_score       | O(Pred × M)     | O(1)             | Pred=user predictions, M=matches                       |
| get_creation_fee     | O(1)            | O(1)             | Direct storage lookup                                  |

---

## Authorization

| Function             | Requires Auth | Caller Restrictions  |
| -------------------- | ------------- | -------------------- |
| verify_event_winners | Yes           | Any address (public) |
| get_event_winners    | No            | None (view function) |
| get_user_score       | No            | None (view function) |
| get_creation_fee     | No            | None (view function) |

---

## State Mutations

| Function             | Mutates State | Storage Keys Modified     |
| -------------------- | ------------- | ------------------------- |
| verify_event_winners | Yes           | EventWinners(event_id)    |
| get_event_winners    | No            | None (TTL extension only) |
| get_user_score       | No            | None (TTL extension only) |
| get_creation_fee     | No            | None                      |

---

## Pause Behavior

| Function             | Paused Behavior               |
| -------------------- | ----------------------------- |
| verify_event_winners | Panics with "contract_paused" |
| get_event_winners    | Allowed (view function)       |
| get_user_score       | Allowed (view function)       |
| get_creation_fee     | Allowed (view function)       |

---

## Integration Checklist

- [ ] Import oracle module in contract
- [ ] Add oracle functions to contract impl
- [ ] Test verify_event_winners with sample data
- [ ] Test get_event_winners sorting
- [ ] Test get_user_score calculation
- [ ] Test get_creation_fee retrieval
- [ ] Verify error handling
- [ ] Check storage TTL management
- [ ] Validate event emission
- [ ] Test with paused contract
- [ ] Performance test with large datasets
- [ ] Deploy to testnet
- [ ] Deploy to mainnet

---

## Troubleshooting

### Issue: "event_not_found" error

**Cause**: Event ID doesn't exist in storage

**Solution**:

1. Verify event_id is correct
2. Check event was created successfully
3. Ensure event hasn't been deleted

### Issue: "matches_not_complete" error

**Cause**: Not all matches have results submitted

**Solution**:

1. Verify all matches have been resolved
2. Check match result submission status
3. Wait for oracle to submit remaining results

### Issue: No winners returned

**Cause**: No participants predicted all matches correctly

**Solution**:

1. Check participant predictions
2. Verify match results are correct
3. Review prediction accuracy

### Issue: Incorrect user score

**Cause**: Unresolved predictions or incorrect match results

**Solution**:

1. Verify all matches are resolved
2. Check match result values
3. Review user predictions

---

## Version History

| Version | Date       | Changes                |
| ------- | ---------- | ---------------------- |
| 1.0.0   | 2026-05-29 | Initial implementation |

---

## Related Documentation

- [ORACLE_IMPLEMENTATION.md](./ORACLE_IMPLEMENTATION.md) - Detailed implementation guide
- [STORAGE_SCHEMA.md](./STORAGE_SCHEMA.md) - Storage key definitions
- [README.md](./README.md) - Contract overview
