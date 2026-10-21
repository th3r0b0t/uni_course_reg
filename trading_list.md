# Course Trading Subsystem: Pros and Cons (V1)

**Scope:** The seat-exchange feature used during the add/drop period of the new registration system. It is called "seat exchange" instead of "auction" because there is no bidding and no money.

## Design assumptions behind this version
- **Add/drop stays.** It is a university tradition and rule, and heads keep adding seats where needed.
- **Students list a course for exchange** and name one or more courses they want in return (ranked), or simply drop it and lose all claims to it.
- **Conditional release:** The student keeps the current seat until a wanted seat is secured. The swap then executes atomically.
- **The list is hidden from students** so they cannot coordinate timing. Heads see aggregates per section (how many want out, how many want in), with individual views access-logged. *(Proposed, not yet confirmed.)*
- **Matching runs in batch rounds,** not first-listed-first-served. Contested seats use the same tiers and retained lottery number as registration (v6). *(Proposed, not yet confirmed.)*
- **Caps:** Active listings and completed swaps are capped per student per term. Outright drops are uncapped but cannot push a student below minimum units.
- **Eligibility is checked twice,** at listing and at execution (conflicts, unit caps, prerequisites, fees).
- **No price or message fields.** A seat gained by exchange cannot be re-listed in the same period.

## Pros

**1. Student protection**
- **No drop-and-hope:** Students never lose the course they hold unless a wanted seat is secured.
- **Atomic swaps:** Either everyone in a swap gets their new seat or nothing changes.
- **Section changes without risk:** Students who dislike a lecturer can move to another section of the same course safely.

**2. Fairness and abuse resistance**
- **Hidden list:** Students cannot coordinate timing to place each other's courses.
- **Batch rounds:** Listing earlier or later gives no advantage, so bots and network quality gain nothing.
- **Consistent rules:** The same tiers and lottery as registration, so students learn one system.
- **Named wishes and caps:** Students cannot hoard seats just to trade them.
- **Brokering deterrents:** No price field, no re-listing of exchanged seats, and flags for repeated reciprocal swaps.
- **Pre-registration:** The main phase already reduces unwanted course picks.

**3. Better matching**
- **Cycle matching (optional extension):** Three-way and longer swaps (A wants B's seat, B wants C's, C wants A's) fill more requests than pairs alone.
- **Free seats are used:** Students can move into open seats through the same process.

**4. Decision support for heads**
- **Real demand per section:** Heads see how many want in and out of each section, instead of guessing from free seats.
- **Fixes the "free seats at a bad time" problem:** A section with free seats at a conflicting time no longer looks fine, because the wishes show where demand is.
- **"One seat away" report:** Shows which added seats would unlock the most moves.
- **Aggregate views by default:** Individual names are access-logged.

**5. Process and timeline**
- **Keeps the add/drop tradition** while replacing its weakest part.
- **Earlier class starts:** Because most students get their chosen courses in pre-registration, few change classes during add/drop, so lecturers can start teaching sooner.
- **Audit trail:** Every listing, match, and swap is logged.

## Cons

**1. Technical complexity**
- **Atomic execution across services:** A swap touches registration, profile, and seat data, and needs a saga or idempotent steps.
- **Cycle matching is harder to build and test** than pair swaps.
- **Re-checks at execution:** Profile or seat data may change between listing and the batch run.
- **Partial-failure recovery:** If one participant fails a check, the cycle must be recomputed.

**2. Fairness and abuse**
- **Brokering is not preventable:** Friends can still swap for private reasons, and payment outside the system cannot be detected.
- **Speculative listing:** Students may list wishes they do not really want. Caps limit this but do not eliminate it.
- **Priority conflicts:** Rules are needed for how freed seats are split between exchanges and the waitlist.
- **Disputes over tiers and results:** An appeals path is required.
- **Hidden list reduces transparency:** Students may suspect the process is unfair because they cannot see it.

**3. Student experience**
- **Slower than instant matching:** Batch rounds mean waiting for a result.
- **Caps frustrate legitimate swaps** for students who need more than the allowed number.
- **Rule complexity:** Conditional release, ranked wishes, and caps are harder to explain than "drop and re-add".

**4. Workload and organization**
- **Heads' workload:** Heads must read aggregates, add seats, and handle exceptions during a short period.
- **Section imbalance:** Many students leaving a disliked lecturer's section can empty it, and cancellation after classes start is disruptive.
- **Policy approval:** The university must approve a new add/drop procedure, even though add/drop itself is kept.

**5. Scope limits**
- **Same-branch only:** Cross-branch exchanges are not supported in this version.
- **Data dependence:** Wrong profile or fee data produces wrong eligibility results.

**6. Security**
- **Exchange API abuse:** Bots could spam listings. Rate limits and per-student caps apply.
- **Privacy of the hidden list:** Wishes reveal student preferences, so access must be restricted and logged.

## Mitigations
| Risk | Mitigation |
|---|---|
| Atomic execution failures | Saga or idempotent steps, all-or-nothing execution, recompute after removal |
| Cycle matching complexity | Pair swaps and free-seat moves in the MVP, cycles added later |
| Stale data at execution | Re-check eligibility at execution, remove failing participants |
| Brokering | No price or message fields, no re-listing of exchanged seats, flags for repeated reciprocal swaps, sanctions |
| Speculative listing | Listing caps, named ranked wishes, one active list per student |
| Priority conflicts with waitlist | Documented rule for splitting freed seats between exchanges and waitlist |
| Disputes | Published rules, appeals path with deadlines, logged decisions |
| Slow batch results | Several rounds per period, with results sent via website and Telegram |
| Heads' workload | Aggregate dashboards, "one seat away" report, defaults for common cases |
| Section imbalance | Heads see per-section outflow and can cap moves out of a section |
| Exchange API abuse | Rate limits, per-student caps, authentication, audit logs |
| Privacy of wishes | Heads see aggregates, individual views access-logged |

## Open items
- Cap values: active listings and completed swaps per term.
- Number and timing of batch rounds in the add/drop period.
- MVP scope: pair swaps only, or cycles from the start.
- Are cross-course exchanges allowed, or only section changes within the same course?
- Do heads see aggregates only, or names with access logging?
- How are freed seats split between exchanges and the waitlist?
- Penalties for abuse.
- Cross-branch exchanges: out of scope for now, or planned later?
