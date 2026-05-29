# Oracle Functions - Quick Start Guide

## What Was Implemented

Four new oracle functions for the CreatorEventManager contract:

1. **verify_event_winners** - Identify perfect scorers
2. **get_event_winners** - Retrieve winners list
3. **get_user_score** - Calculate user's score
4. **get_creation_fee** - Get event creation fee

## Quick Reference

### Function Signatures

```rust
// Verify and record all perfect scorers for an event
pub fn verify_event_winners(env: Env, caller: Address, event_id: u64) -> u32

// Retrieve the list of winners for an event
pub fn get_event_winners(env: Env, event_id: u64) -> Vec<Winner>

// Calculate a user's score for an event
pub fn get_user_score(env: Env, user: Address, event_id: u64) -> (u32, u32)

// Retrieve the current creation fee
pub fn get_creation_fee(env: Env) -> i128
```

## Usage Examples

### 1. Verify Winners After Event Resolution

```rust
// After all matches are resolved
let winner_count = contract.verify_event_winners(caller, event_id)?;
println!("Event {} has {} winners", event_id, winner_count);
```

**When to use**: After all matches in an event have been resolved
**Returns**: Number of perfect scorers identified
**Errors**: Paused, EventNotFound, EventCancelled, MatchesNotComplete

---

### 2. Display Leaderboard

```rust
// Retrieve sorted winners for display
let winners = contract.get_event_winners(event_id)?;
println!("Leaderboard for Event {}", event_id);
println!("Rank | User | Score | Accuracy");

for (rank, winner) in winners.iter().enumerate() {
    let accuracy = winner.get_accuracy_percentage();
    println!(
        "#{} | {} | {}/{} | {}%",
        rank + 1,
        winner.user,
        winner.total_correct,
        winner.total_matches,
        accuracy
    );
}
```

**When to use**: Display leaderboard or rankings
**Returns**: Vec<Winner> sorted by completion_time (earliest first)
**Sorting**: Earlier completion times rank higher

---

### 3. Check User Score

```rust
// Get user's current score
let (correct, total) = contract.get_user_score(user, event_id)?;
let accuracy = if total > 0 { (correct * 100) / total } else { 0 };
println!("User score: {}/{} ({}%)", correct, total, accuracy);
```

**When to use**: Display user's partial score during or after event
**Returns**: Tuple (correct_count, total_matches)
**Note**: Unresolved predictions are not counted

---

### 4. Display Creation Fee

```rust
// Show fee to user
let fee_stroops = contract.get_creation_fee();
let fee_xlm = fee_stroops as f64 / 10_000_000.0;
println!("Event creation fee: {} XLM", fee_xlm);
```

**When to use**: Display cost before event creation
**Returns**: Fee in stroops (i128)
**Note**: Returns 0 if not set (should not happen after init)

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

- `get_accuracy_percentage() -> u32` - Returns 0-100
- `outranks(other) -> bool` - Leaderboard comparison

---

## Error Handling

### Common Errors

| Error                | Cause                    | Solution                            |
| -------------------- | ------------------------ | ----------------------------------- |
| `Paused`             | Contract is paused       | Wait for admin to unpause           |
| `EventNotFound`      | Event doesn't exist      | Verify event_id is correct          |
| `EventCancelled`     | Event is cancelled       | Cannot process cancelled events     |
| `MatchesNotComplete` | Not all matches resolved | Wait for all matches to be resolved |
| `CreationFeeNotSet`  | Fee not initialized      | Should not happen after init        |
| `Overflow`           | Arithmetic overflow      | Contact support                     |

### Error Handling Pattern

```rust
match contract.verify_event_winners(caller, event_id) {
    Ok(count) => println!("Winners: {}", count),
    Err(e) => match e {
        OracleError::Paused => println!("Contract is paused"),
        OracleError::EventNotFound => println!("Event not found"),
        OracleError::EventCancelled => println!("Event is cancelled"),
        OracleError::MatchesNotComplete => println!("Not all matches resolved"),
        _ => println!("Error: {:?}", e),
    }
}
```

---

## Workflow Examples

### Complete Event Resolution Workflow

```rust
// 1. Create event
let (event_id, invite_code) = contract.create_event(
    creator,
    "World Cup 2026",
    "Predict match winners",
    100,
)?;

// 2. Add matches
for match_data in matches {
    contract.add_match(creator, event_id, match_data)?;
}

// 3. Users join and predict
contract.join_event(user1, invite_code)?;
contract.submit_prediction(user1, match_id_1, "TEAM_A")?;
contract.submit_prediction(user1, match_id_2, "TEAM_B")?;

// 4. Resolve matches
contract.submit_match_result(oracle, match_id_1, "TEAM_A")?;
contract.submit_match_result(oracle, match_id_2, "TEAM_B")?;

// 5. Verify winners
let winner_count = contract.verify_event_winners(caller, event_id)?;
println!("Found {} winners", winner_count);

// 6. Display leaderboard
let winners = contract.get_event_winners(event_id)?;
for (rank, winner) in winners.iter().enumerate() {
    println!("#{}: {} - {}/{}", rank + 1, winner.user, winner.total_correct, winner.total_matches);
}
```

### Partial Score Display During Event

```rust
// During event (before all matches resolved)
let (correct, total) = contract.get_user_score(user, event_id)?;
println!("Current score: {}/{}", correct, total);

// After event (all matches resolved)
let (correct, total) = contract.get_user_score(user, event_id)?;
let accuracy = (correct * 100) / total;
println!("Final score: {}/{} ({}%)", correct, total, accuracy);
```

---

## Performance Tips

### For Large Events (1000+ participants)

1. **Verify Winners in Batches**
   - Consider implementing batch verification
   - Process winners in chunks to avoid timeout

2. **Cache Leaderboard**
   - Store leaderboard snapshot after verification
   - Reduces repeated sorting operations

3. **Optimize Score Queries**
   - Cache user scores during event
   - Update only when new predictions submitted

### Storage Optimization

```rust
// Good: Single query for all winners
let winners = contract.get_event_winners(event_id)?;

// Avoid: Multiple queries for individual winners
for winner_id in winner_ids {
    let winner = contract.get_winner(event_id, winner_id)?; // ❌ Inefficient
}
```

---

## Testing

### Manual Testing Checklist

- [ ] Create event successfully
- [ ] Add matches to event
- [ ] Users can join event
- [ ] Users can submit predictions
- [ ] Verify winners after resolution
- [ ] Get event winners returns sorted list
- [ ] Get user score calculates correctly
- [ ] Get creation fee returns correct value
- [ ] Error handling works for all error cases
- [ ] Pause mechanism blocks verify_event_winners

### Test Commands

```bash
# Build contract
cargo build --release

# Run all tests
cargo test

# Run specific test
cargo test oracle

# Check compilation
cargo check
```

---

## Integration Steps

### 1. Add to Your Contract

The oracle module is already integrated in `src/lib.rs`:

```rust
pub mod oracle;
```

### 2. Call from Frontend

```javascript
// Using Stellar SDK
const result = await contract.invoke({
  method: "verify_event_winners",
  args: [caller, eventId],
});

const winners = await contract.invoke({
  method: "get_event_winners",
  args: [eventId],
});

const [correct, total] = await contract.invoke({
  method: "get_user_score",
  args: [userAddress, eventId],
});

const fee = await contract.invoke({
  method: "get_creation_fee",
  args: [],
});
```

### 3. Handle Results

```javascript
// Verify winners
if (result.ok) {
  console.log(`Found ${result.value} winners`);
} else {
  console.error(`Error: ${result.error}`);
}

// Display leaderboard
winners.forEach((winner, rank) => {
  console.log(
    `#${rank + 1}: ${winner.user} - ${winner.total_correct}/${winner.total_matches}`,
  );
});

// Show user score
console.log(`Score: ${correct}/${total}`);

// Display fee
console.log(`Fee: ${fee / 10_000_000} XLM`);
```

---

## Troubleshooting

### Issue: "event_not_found"

**Cause**: Event ID doesn't exist
**Solution**: Verify event_id is correct and event was created

### Issue: "matches_not_complete"

**Cause**: Not all matches have been resolved
**Solution**: Wait for oracle to submit all match results

### Issue: No winners returned

**Cause**: No participants predicted all matches correctly
**Solution**: Check participant predictions and match results

### Issue: Incorrect user score

**Cause**: Unresolved predictions or incorrect match results
**Solution**: Verify all matches are resolved and results are correct

---

## Best Practices

✅ **DO**:

- Verify winners after all matches are resolved
- Cache leaderboard results for performance
- Handle all error cases gracefully
- Use get_user_score for partial scoring
- Display creation fee before event creation

❌ **DON'T**:

- Call verify_event_winners multiple times for same event
- Assume winners list is unsorted
- Ignore error codes
- Use unresolved predictions in scoring
- Modify Winner structs directly

---

## API Endpoints Summary

| Endpoint             | Method | Parameters       | Returns     |
| -------------------- | ------ | ---------------- | ----------- |
| verify_event_winners | POST   | caller, event_id | u32         |
| get_event_winners    | GET    | event_id         | Vec<Winner> |
| get_user_score       | GET    | user, event_id   | (u32, u32)  |
| get_creation_fee     | GET    | -                | i128        |

---

## Additional Resources

- **ORACLE_IMPLEMENTATION.md** - Detailed implementation guide
- **ORACLE_API_REFERENCE.md** - Complete API reference
- **IMPLEMENTATION_SUMMARY.md** - Implementation overview

---

## Support

For issues or questions:

1. Check the troubleshooting section above
2. Review ORACLE_API_REFERENCE.md for detailed documentation
3. Check error codes and their meanings
4. Contact development team with error details

---

**Last Updated**: May 29, 2026
**Version**: 1.0.0
