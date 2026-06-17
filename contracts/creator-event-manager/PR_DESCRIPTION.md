# Exact Scoreline Predictions with Points Grading (#966)

## Overview

Implements exact scoreline predictions with automatic points grading. Users can now predict exact match scores (e.g., "Arsenal 2 – Chelsea 1") instead of just the 1X2 result, earning bonus points for accuracy.

## Scoring System

- **0 points**: Wrong 1X2 result
- **1 point**: Correct 1X2 result, wrong scoreline
- **4 points**: Exact scoreline (1 for result + 3 for exact score)

## Changes

### Storage Types (`src/storage_types.rs`)

- Added `home_score: Option<u32>` and `away_score: Option<u32>` to Match struct
- Added `predicted_home_score: u32` and `predicted_away_score: u32` to Prediction struct
- Added `points_earned: Option<u32>` and `is_correct: Option<bool>` to Prediction struct
- Defined scoring constants: `POINTS_CORRECT_RESULT=1`, `POINTS_EXACT_SCORE=3`
- Implemented `MatchResult::from_scores(home, away)` helper to derive 1X2 result from scoreline
- Implemented `Prediction::grade(actual_home, actual_away)` for automatic grading

### Prediction Submission (`src/prediction.rs`)

- Changed `submit_prediction()` signature: `(predictor, match_id, predicted_home_score, predicted_away_score)`
- Automatically derives `predicted_outcome` from scores (no user input)
- Removed outcome validation check (now compile-time safe)

### Oracle Match Results (`src/oracle.rs`)

- Changed `submit_match_result()` signature: `(caller, match_id, home_score, away_score)`
- Automatically derives `winning_team` from scores
- Grades all predictions immediately after result submission
- Updated `get_user_score()` return: `(total_points, correct_results, exact_scores, total_matches)`

### Contract Interface (`src/lib.rs`)

- Updated `submit_prediction()` contract method
- Updated `submit_match_result()` contract method
- Updated `get_user_score()` documentation

## Testing

- All 537+ tests passing
- 15 new scoreline-specific integration tests in `submit_match_result_contract_tests.rs`
- Tests validate:
  - Exact score predictions award 4 points
  - Correct result/wrong score awards 1 point
  - Wrong result awards 0 points
  - All outcomes (Team A win, Team B win, Draw) work correctly
  - Points aggregate correctly across multiple matches
  - `get_user_score()` returns accurate statistics

## Acceptance Criteria ✓

- [x] `submit_prediction()` takes scoreline (home_score, away_score)
- [x] `submit_match_result()` takes final scoreline (home_score, away_score)
- [x] Grading awards 0/1/4 points per specification
- [x] `get_user_score()` returns (total_points, correct_results, exact_scores, total_matches)
- [x] `Prediction.points_earned` is None until graded, then Some(0|1|4)
- [x] Comprehensive test coverage

## Migration Notes

This is a breaking change to the prediction API:

- Old: `submit_prediction(predictor, match_id, outcome_symbol)`
- New: `submit_prediction(predictor, match_id, home_score, away_score)`

Callers must be updated to pass scores instead of outcome symbols.

## Files Changed

- `src/storage_types.rs` - Data structure updates
- `src/prediction.rs` - Prediction submission logic
- `src/oracle.rs` - Oracle result submission and grading
- `src/lib.rs` - Contract interface
- `tests/*.rs` - Test suite updates (7 test files)

## Commits

1. `3ec12b49` - Storage types: Add scoreline fields and scoring constants
2. `b1d8d26d` - Prediction: Update to scoreline-based submission
3. `027234b1` - Oracle: Implement scoreline-based results and grading
4. `fc312140` - Contract interface: Update exposed functions
5. `cc60a4c2` - Tests: Update suite for new API
