# Learning Practice: Black-Box Testing, Part 2

**Application:** North Vacation Booking  
**Topics:** Decision table testing and state transition testing

# Overview

In this activity, I designed tests from the North Vacation Booking requirements without looking at application code. I used a decision table to check when reservations should be accepted or rejected, then used state transition testing to check how reservations move through their life cycle. These are **planned test cases**, not results from running the application.

# Step 1: Identify Conditions and Actions

# Five conditions

1. **Dates are valid:** Check-in is before checkout.
2. **Stay length is valid:** The difference between checkout and check-in is 1–31 nights, excluding checkout day.
3. **Guest count is valid:** There are 1–12 guests.
4. **Dates are available:** Every requested night is available.
5. **Extra guests:** The reservation includes more than five guests.

# Five actions

1. **Create reservation:** Accept valid information and start the reservation in `Pending`.
2. **Calculate standard price:** Charge $200 per night plus a one-time $50 cleaning fee.
3. **Add extra guest fee:** Charge $25 per night for each guest above five.
4. **Raise `ValueError`:** Reject invalid reservation information.
5. **Raise `UnavailableError`:** Reject a request if any requested night is unavailable.

**Price formula for valid reservations:** `($200 × nights) + $50 + ($25 × max(guests − 5, 0) × nights)`.

# Step 2: Decision Table

`T` = true, `F` = false, `–` = does not affect this rule, and `X` = expected action. A blank action cell means that action does not occur.

| Conditions and actions | R1 | R2 | R3 | R4 | R5 | R6 |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Check-in is before checkout | T | T | F | T | T | T |
| Stay is 1–31 nights | T | T | F | F | T | T |
| Guest count is 1–12 | T | T | – | – | F | T |
| All requested nights available | T | T | – | – | – | F |
| More than five guests | F | T | – | – | – | – |
| **Create reservation (`Pending`)** | X | X | | | | |
| **Calculate standard price** | X | X | | | | |
| **Add extra guest fee** | | X | | | | |
| **Raise `ValueError`** | | | X | X | X | |
| **Raise `UnavailableError`** | | | | | | X |

**Rule explanations:** R1 accepts a standard booking; R2 accepts a booking with extra guests; R3 rejects checkout on or before check-in; R4 rejects a stay longer than 31 nights; R5 rejects an invalid guest count; and R6 rejects an unavailable night. In R3, the stay-length check is marked `F` because a non-positive difference is outside 1–31 nights. Invalid-input rules use `–` for conditions that cannot change the expected `ValueError` in that test. The rules intentionally avoid combining multiple independent errors because the requirements do not specify which error takes priority.

# Step 3: Decision Table Test Cases

For every case, the dates are in **2026**. An available date range means every reserved night, from check-in through the night before checkout, is available. No application code is provided, so the following results are expectations.

| Test ID | Rule / conditions | Input values | Expected action | Expected price or exception |
|---|---|---|---|---|
| DT01 | R1: valid dates, nights, guests and availability; no extra guests | Check-in Oct 1; checkout Oct 4; 3 guests; all nights available | Create reservation in `Pending`; calculate standard price | **$650** = (3 × $200) + $50 |
| DT02 | R2: valid dates, nights, guests and availability; extra guests | Check-in Oct 1; checkout Oct 4; 7 guests; all nights available | Create reservation in `Pending`; calculate standard price and extra guest fee | **$800** = $600 + $50 + (2 × $25 × 3) |
| DT03 | R3: dates invalid | Check-in Oct 4; checkout Oct 4; 3 guests; availability not relevant | Reject reservation | **`ValueError`**; no price |
| DT04 | R4: dates in order but stay too long | Check-in Oct 1; checkout Nov 2; 3 guests; availability not relevant (32 nights) | Reject reservation | **`ValueError`**; no price |
| DT05 | R5: guest count too high | Check-in Oct 1; checkout Oct 4; 13 guests; availability not relevant | Reject reservation | **`ValueError`**; no price |
| DT06 | R6: valid inputs but at least one night unavailable | Check-in Oct 1; checkout Oct 4; 3 guests; Oct 2 unavailable | Reject reservation | **`UnavailableError`**; no price |

**Calculation check:** Oct 1 check-in to Oct 4 checkout is **three** nights (Oct 1, 2 and 3). Checkout day is not charged. The $50 cleaning fee is charged only once on a valid booking.

# Step 4: Identify Reservation States and Events

# Five states

| State | Meaning |
|---|---|
| `Pending` | A valid reservation has been created and is awaiting payment. |
| `Confirmed` | Payment succeeded and the reservation is confirmed. |
| `Checked In` | The guest checked in on the scheduled check-in date. |
| `Completed` | The checked-in reservation has passed through checkout. |
| `Cancelled` | The reservation was cancelled or payment failed. |

# Five events

1. **Payment succeeds** — `Pending` → `Confirmed`.
2. **Payment fails** — `Pending` → `Cancelled`.
3. **Guest cancels** — `Pending` or `Confirmed` → `Cancelled`.
4. **Guest checks in on the check-in date** — `Confirmed` → `Checked In`.
5. **Guest checks out** — `Checked In` → `Completed` after checkout.

# Step 5: State Transition Table

The first six rows are valid transitions and the last five are invalid examples. For invalid transitions, the reservation stays in its current state and raises `InvalidStateError`.

| ID | Current state | Event | Expected state | Expected result | Type |
|---|---|---|---|---|---|
| TR01 | `Pending` | Payment succeeds | `Confirmed` | Reservation is confirmed. | Valid |
| TR02 | `Pending` | Payment fails | `Cancelled` | Reservation is cancelled. | Valid |
| TR03 | `Pending` | Guest cancels | `Cancelled` | Pending reservation is cancelled. | Valid |
| TR04 | `Confirmed` | Guest checks in on check-in date | `Checked In` | Guest is checked in. | Valid |
| TR05 | `Confirmed` | Guest cancels | `Cancelled` | Confirmed reservation is cancelled. | Valid |
| TR06 | `Checked In` | Guest checks out after checkout | `Completed` | Stay is completed. | Valid |
| TR07 | `Cancelled` | Guest attempts to check in | `Cancelled` | Raise `InvalidStateError`. | Invalid |
| TR08 | `Completed` | Guest attempts to cancel | `Completed` | Raise `InvalidStateError`. | Invalid |
| TR09 | `Pending` | Guest attempts to check in | `Pending` | Raise `InvalidStateError`; payment has not confirmed the booking. | Invalid |
| TR10 | `Confirmed` | Guest attempts to check out without checking in | `Confirmed` | Raise `InvalidStateError`. | Invalid |
| TR11 | `Cancelled` | Payment succeeds | `Cancelled` | Raise `InvalidStateError`; cancelled is final. | Invalid |

**Assumption for invalid-event tests:** Events without a permitted transition in the stated requirements are treated as invalid. The requirements explicitly say that invalid state transitions raise `InvalidStateError`. The requirements do **not** specify refund processing or releasing dates, so those behaviors are not assumed here.

# Step 6: State Transition Test Cases

Each test starts with a reservation already in the listed state, except sequence tests, which begin with a newly created valid reservation in `Pending`.

| Test ID | Starting state | Event / test input | Expected state | Expected result | Type |
|---|---|---|---|---|---|
| ST01 | `Pending` | Successful payment | `Confirmed` | Payment confirms the reservation. | Valid |
| ST02 | `Pending` | Failed payment | `Cancelled` | Failed payment cancels the reservation. | Valid |
| ST03 | `Pending` | Guest requests cancellation | `Cancelled` | Pending reservation is cancelled. | Valid |
| ST04 | `Confirmed` | Guest checks in on the scheduled date | `Checked In` | Check-in succeeds. | Valid |
| ST05 | `Confirmed` | Guest requests cancellation | `Cancelled` | Confirmed reservation is cancelled. | Valid |
| ST06 | `Checked In` | Checkout is completed after checkout | `Completed` | Reservation is completed. | Valid |
| ST07 | `Cancelled` | Guest attempts check-in | `Cancelled` | Raise `InvalidStateError`; state does not change. | Invalid |
| ST08 | `Completed` | Guest attempts cancellation | `Completed` | Raise `InvalidStateError`; state does not change. | Invalid |
| ST09 | `Pending` | Guest attempts check-in before payment succeeds | `Pending` | Raise `InvalidStateError`; state does not change. | Invalid |
| ST10 | `Pending` | **Complete sequence:** valid booking → successful payment → check-in on check-in date → checkout after checkout | `Completed` | States progress `Pending` → `Confirmed` → `Checked In` → `Completed`, without errors. | Valid sequence |
| ST11 | `Pending` | **Cancellation sequence:** valid booking → successful payment → guest cancels | `Cancelled` | States progress `Pending` → `Confirmed` → `Cancelled`, without errors. | Valid sequence |

**Coverage check:** ST01–ST06 cover all **six valid transitions**. ST07–ST09 cover **three invalid transitions**. ST10 covers a complete reservation, and ST11 covers a cancellation sequence, giving **11 state transition test cases** in total. TR10 and TR11 add two more invalid scenarios to the transition table for extra coverage.

# What I Learned

Decision tables helped me organize different booking conditions and show exactly when the system should accept a reservation, calculate fees, or raise an error. State transition testing helped me check that a reservation can move through allowed states but cannot move out of a cancelled or completed state. Together, these techniques make the expected behavior easier to test without needing the source code.

# AI Transparency

I used AI assistance to help organize the decision table, draft the test cases, check calculations, and improve the writing. I reviewed the work against the assignment requirements.

# References

- [Decision Table Testing — GeeksforGeeks](https://www.geeksforgeeks.org/software-engineering/decision-table-based-testing-in-software-testing/)
- [State Transition Testing — GeeksforGeeks](https://www.geeksforgeeks.org/software-engineering/state-transition-testing/)
