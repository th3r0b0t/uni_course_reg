# Pros and Cons (v4, consolidated)

**Project type:** New software, deployed as an alternative path alongside the existing website, growing from regional pilot to national scale.

**Architecture and procedure assumptions behind this version**
- Decoupled microservices, hosted in rented enterprise data centers managed centrally by the capital's IT department, with services replicated across several data centers.
- Branch-local registration: students see and register for their own branch's courses. Data syncs to headquarters with eventual consistency, and each student has one home partition for profile-wide checks.
- Your multi-phase procedure replaces one-shot registration: pre-registration, head decisions, auto-finalization and replacement, then a completion round.
- Oversubscribed courses are allocated in a batch (random order within priority tiers). This is my suggestion and not yet confirmed by you.
- Profiles are consistent by the time a semester begins (your guarantee).

## Pros

**1. Fault isolation**
- **Contained failures:** If trading, waitlists, or notifications fail, viewing and registering keep working.
- **Circuit breakers and timeouts:** A slow service is cut off instead of dragging down its callers.
- **Graceful degradation:** Non-critical features switch off under extreme load while core registration stays up.
- **Buffered writes:** Message queues absorb spikes and hold requests if a downstream service is briefly down.
- **Branch-level isolation:** Each branch's registration is its own partition, so an overload or fault in one branch does not affect the others.
- **Multi-data-center redundancy:** Services are replicated across data centers, so losing one site does not stop the system.
- **Old system as fallback** during the pilot.

**2. Security isolation** *(motivated by the previous breach)*
- **Limited blast radius:** A compromised service (e.g., the Telegram gateway) cannot automatically reach registration or student records.
- **Separate databases and credentials per service:** Stolen credentials expose one service's data only.
- **Least privilege and network segmentation:** Internal services are not publicly reachable.
- **Single hardened entry point:** Authentication, rate limiting, input validation, and a WAF are enforced in one place.
- **Sensitive data separation:** Personal and academic data live in a tightly controlled service, apart from public-facing components.
- **Per-service audit logs:** Suspicious activity is easier to trace.
- **Patching in small units:** A fix touches one service and deploys quickly.
- **Centralized administration:** One IT team means one patch process, one audit scope, and fewer misconfigurations than per-branch hosting.

**3. Scalability and performance**
- **Designed for national scale:** Semester-start spikes are handled by independent scaling of the hot services (registration, allocation, ingest).
- **Horizontal scaling and gradual growth** by adding instances in the data centers.
- **Branch partitioning:** Students compete only within their own branch, which shrinks contention and scales per branch.
- **Caching and async processing:** Reads are cached and writes are queued.
- **Same architecture from pilot to national rollout,** with no rewrite.

**4. Infrastructure and cost**
- **Reuse of already-rented servers:** Services are placed on existing capacity, so there is no new hardware purchase.
- **Right-sized placement:** Light services go on small instances and heavy ones on larger instances.
- **No spend on branch-level infrastructure,** since hosting stays central.

**5. Fairness**
- **No race in the main phase:** Pre-registration has no seat limits, so network quality and server load stop affecting outcomes.
- **Fairness defined as equal opportunity within a time window,** not millisecond ordering.
- **Server-side timestamping:** A lightweight, over-provisioned ingest layer stamps requests, never the client.
- **Time buckets and random ordering** within a window neutralize latency differences and exact-millisecond bots.
- **Processing decoupled from ingestion:** Server load cannot change anyone's position.
- **Batch allocation for oversubscribed courses:** Random order within priority tiers makes bots gain nothing from speed.

**6. Registration functionality**
- **Pre-registration without seat limits,** with fee payment up front.
- **Ranked priorities per schedule:** Students may pick several courses for one schedule, ranked, and a lower rank is promoted automatically if a higher one is cancelled.
- **Substitute requests:** Students can request courses that are not offered, as a subsidiary of their top-priority course in a schedule. A request is auto-replaced only if it causes no conflict, so nobody keeps courses they don't want.
- **Limited requests:** Each student gets a request cap and at most one requested course per schedule, which keeps the data clean and reduces gaming.
- **Completion round:** Students resolve what is left: conflicting requested courses and cancelled classes, filled from free seats.
- **Course trading and waitlist queuing** (fair, ordered by ingest stamp, auto-enrollment when a seat opens).
- **Rule-based editing windows** enforced in the backend.

**7. Decision support for field heads** *(shown only after the pre-registration deadline)*
- **Schedule popularity:** Demand per slot, per course and across all courses in that time, with top-priority and any-priority counts separated.
- **Requested-course preview:** For each requested course, heads see its preferred schedules, how many requesters would be auto-replaced, and how many have a conflict for each potential schedule, along with their open schedules. Only requesters are counted, so the preview is cheap to compute.
- **Eligibility view:** Per course, "needs it" (from the chart), "is eligible," and "requested it."
- **Off-semester demand:** Students who still need courses from the other semester parity, split into already requested (real) and not yet requested (potential).
- **Chart-based planning:** Each semester's chart courses are scheduled conflict-free for on-track cohorts by design.
- **Safe dashboards:** Because data is shown only after the deadline, the numbers are final and not misread as live.
- **Demand-driven timetable:** Fewer cancelled courses, fewer conflicts, and a measure of how many students are behind their chart.

**8. Maintainability and development**
- **Interface independence:** One API serves the website, Telegram bot, and future apps.
- **Lower maintenance and feature cost:** Changes touch one small service.
- **Technology freedom and parallel development** across teams.

**9. Deployment and risk**
- **Safe rollout and real-world testing** before the old system is retired.
- **Incremental delivery:** Features go live as they are ready.

**10. User experience**
- Modern UI/UX, multiple channels (e.g., Telegram alerts when a seat opens), and real-time feedback.

## Cons

**1. Development cost and time**
- **High initial effort,** and the scope has grown with pre-registration, allocation, simulation, and dashboards.
- **Skilled personnel needed** in distributed systems and security.
- **Time to first value:** Trading and queues depend on the core being in place, and the new procedure cannot be shown half-built.

**2. Architectural complexity**
- **Distributed-system challenges:** Latency, partial failures, inter-service communication.
- **Data consistency:** Trading, replacement, and allocation span services and need sagas or idempotent operations.
- **Harder debugging:** Centralized logging and tracing are required.
- **Premature complexity at pilot stage:** Many tiny services may be more than a regional pilot needs. Start coarse-grained and split as load grows.

**3. Fault isolation limits and high availability**
- **Cascading failures if poorly designed,** without timeouts and circuit breakers.
- **Shared components:** The gateway, broker, and central profile service are potential single points of failure. This is a generic HA concern, addressed by multi-data-center redundancy.
- **HA has its own cost:** Replication, failover testing, and cross-data-center consistency add complexity.
- **Partial-failure recovery:** Cases such as a seat reserved but not confirmed need explicit logic.

**4. Security risks**
- **Larger attack surface:** More services, public APIs, and a Telegram bot.
- **Isolation is not automatic:** Misconfigured network rules or broad credentials erase the benefit.
- **Secrets management:** Many services mean many keys to store and rotate.
- **Insecure internal traffic:** Service-to-service communication needs encryption and authentication.
- **High-value, centralized target:** A national system with all data in one managed place attracts attackers, and the university has been breached before.
- **Central admin as a risk:** One team means one point of high privilege, so a compromised admin account has wide impact.
- **DDoS risk:** Registration windows are predictable.
- **Third-party dependency:** Telegram is outside university control and raises privacy concerns.
- **Identity linking:** Securely binding Telegram accounts to student identities is nontrivial.

**5. Infrastructure and operations**
- **Rental constraints:** Capacity is bound by existing rental contracts, and peaks may require extra paid capacity.
- **Cross-data-center replication traffic** and inter-site latency.
- **Monitoring and operations burden** across many services and sites.

**6. Branch-local design**
- **Cross-branch rules:** Credit limits, schedule conflicts, and prerequisites depend on the whole profile, so each student needs one authoritative home partition. Eventual consistency can allow stale-data checks.
- **Cross-branch courses:** Courses taken at another branch, shared or online courses, and transfers need a defined seat owner.
- **Uneven branches:** Small branches may have thin demand, so many courses fall below minimum enrollment.

**7. Process and organizational change**
- **Policy approval:** The university must approve a new registration procedure, which is a bigger risk than the software.
- **Deadlines and workload:** The education department and field heads must hit deadlines, and heads take on more decision work.
- **More phases, roles, and workflow states** to build and support.
- **Longer registration timeline,** since heads decide only after the deadline.
- **Complex schedule model:** Alternating-week 1-hour classes require a biweekly-aware conflict check.
- **Curriculum versions:** Charts vary by entry year, so each student needs a linked chart version.
- **Conflicts beyond the chart:** Lecturer and room conflicts, behind-schedule students, and cross-field electives are not covered by chart-based planning.
- **Eligibility rules correctness:** Wrong prerequisite or fee data makes the dashboards and auto-finalization wrong.
- **Privacy:** Heads should see aggregates by default, with small counts suppressed or access-logged.

**8. Running two systems in parallel**
- **Shared data conflicts:** Both systems could modify the same seats, so double-booking is possible. This is the biggest parallel-run risk.
- **Procedure mismatch:** The old system has no pre-registration, so students of one branch or field cannot be split across paths. The pilot must be defined per branch or field group, not per student.
- **Integration with university systems:** Student records, SSO, and finance may have limited or undocumented APIs.
- **Double maintenance and user confusion** until the old system is retired.
- **Weaker proof at scale:** A small pilot may not exercise real peak load.

**9. National-scale risks**
- **Data residency and privacy compliance.**
- **Branch differences:** Calendars, rules, and legacy data vary by branch.
- **Higher stakes:** An outage or breach affects far more students.
- **Governance:** Someone must own operations, security, and policy.

**10. Fairness, abuse, and user experience**
- **Trading abuse:** Students might sell seats or collude.
- **Queue disputes:** Time conflicts, withdrawn interest, and priority questions.
- **Gaming of requests:** Padding lists with unwanted courses to block others, or strategic requests. The unit and request caps limit this but don't eliminate it.
- **Multiple or sold accounts.**
- **Ingest saturation:** The ingest layer can still be overwhelmed by an attack or extreme load.
- **Perceived unfairness:** Random allocation can feel unfair to senior students unless priority tiers are used.
- **Interface complexity:** Ranked priorities per schedule plus substitute rules are harder to explain than "pick courses," so students may misuse them.
- **Digital divide:** Less tech-savvy students may be disadvantaged, though the old path partly mitigates this.

**11. Adoption and sustainability**
- **Training and change resistance** for students and staff.
- **Unproven performance:** Gains are estimates until tested under real load.
- **Requirements uncertainty:** Scope may shift with real usage.
- **Ongoing support and knowledge concentration:** On-call coverage during registration periods and dependence on a few key developers.

## Mitigations

| Risk | Mitigation |
|---|---|
| Cascading failures | Timeouts, circuit breakers, bulkheads, fault-injection testing |
| Shared SPOFs and HA complexity | Redundant gateway, replicated database and broker across data centers, regular failover drills |
| Compromised service spreading | Network segmentation, per-service credentials, least privilege, mutual TLS |
| Entry-point attacks and DDoS | Hardened gateway with WAF, per-student rate limits, DDoS protection |
| Central admin compromise | MFA, privileged access management, audit logging, separation of duties |
| Another breach | Independent security audit and penetration test before pilot, incident response plan |
| Secrets sprawl | Central secrets manager with rotation |
| Cross-branch rules | One home partition per student for profile-wide checks, defined seat owner for cross-branch courses |
| Double-booking across systems | Pilot by whole branch or field group, with one authoritative seat source per course |
| Ingest saturation and bots | Over-provisioned stateless ingest, time buckets, CAPTCHA/MFA at login, one active list per student |
| Request gaming | Unit cap, request cap, one requested course per schedule |
| Perceived unfairness | Priority tiers plus random tie-breaking, published allocation rules |
| Biweekly and chart gaps | Schedule model with week-parity, lecturer and room checks in the timetable tool |
| Curriculum versions | Chart version linked to each student's entry year |
| Privacy of dashboards | Aggregate-only views, small-count suppression, access logs |
| Policy and organizational risk | Early approval from the head of education, published deadlines, training for field heads |
| Pilot complexity | A few coarse-grained services with clear boundaries, split later |
| Data residency | Regional data storage if required, data minimization, legal review |
| Telegram privacy | Minimal data exposure, website stays the primary channel |
| Trading abuse | Admin approval, audit logs, no payment features |
| Cost and time | Phased MVP: view and register first, then pre-registration, then trading and queues |

## What changed from the last list
- **Removed:** Weak and unpatched hardware, heterogeneous environment, cross-site seat conflicts, profile-sync timing, over-trusting dashboards, and the what-if interaction between accepted requests. Each is now handled by your decisions or guarantees.
- **Reduced:** Network bottleneck (now only cross-data-center replication) and governance (one owner).
- **Added:** Branch-local design, process and organizational change, procedure mismatch in the parallel run, interface complexity, and the HA cost.

---

# v5 changes (appended): final manual seat-allocation phase

**Context:** In the completion round, when candidates exceed free seats, no mechanism is perfectly fair. Agreed approach: add capacity first; if still oversubscribed, heads choose a published, rule-based policy (priority tiers, then lottery) and the system executes it. Heads set the rule, not individual winners.

## Modified from v4
- **Time buckets (Pros 5, Mitigations "Ingest saturation and bots"):** For contested seats, time buckets are replaced by a lottery over everyone who applied within an open window (a few hours, wide enough to be open to all). Exact click time is irrelevant. Ingest-stamp ordering remains only for non-contested cases.
- **Waitlist (Pros 6):** Waitlist order after a contested round is the random-lottery order, not the ingest-stamp order.
- **Batch allocation (Pros 5):** Now confirmed: priority tiers by need, random order within a tier.
- **Perceived unfairness (Cons 10):** Reframed: fairness means a consistent, published, auditable process, not equal outcomes.

## New Pros
- **Capacity first:** Adding seats or sections during add/drop resolves most oversubscription at low cost.
- **Need-based priority tiers:** Objective criteria (course on the student's current chart semester or near graduation; previously failed or withdrawn; short of minimum units) shrink the contested pool before any randomness.
- **Lottery within tiers:** No timing advantage, so bots and network quality gain nothing.
- **Heads set policy, system executes:** Reduces favoritism, pressure, and heads' workload compared to hand-picking winners.
- **Transparency and auditability:** Rules published before each round, random seed logged and published afterward, full audit log.
- **Soft landing for losers:** Randomly ordered waitlist, alternatives shown (other sections, other branches if allowed), optional priority next semester.
- **Documented exceptions:** Limited overrides (e.g., disability accommodations) with a recorded reason.

## New Cons
- **Heads' extra workload:** Choosing tiers, publishing rules, and handling exceptions each round.
- **Tier-definition disputes:** Priority criteria always create arguments about who qualifies.
- **Lottery feels arbitrary** to students who need one specific course.
- **Appeals burden:** An appeals process needs staff, time limits, and rules.
- **Override abuse:** Exceptions can become a favoritism channel without strict logging and review.
- **Randomness verification:** Seed generation and logging must be trustworthy, or the lottery is not credible.
- **Policy approval:** Tier rules and the lottery need approval from the university, which adds to the organizational risk in Cons 7.

## New Mitigations
| Risk | Mitigation |
|---|---|
| Heads' workload | Policy templates, rules reused across rounds, add capacity first |
| Tier-definition disputes | Objective, data-derived criteria; rules published before the round |
| Lottery feels arbitrary | Randomly ordered waitlist, shown alternatives, next-round priority for losers |
| Appeals burden | Defined appeals path with deadlines, decisions logged |
| Override abuse | Limited override categories, mandatory recorded reason, periodic audit |
| Randomness credibility | Logged seed, published after the round, reproducible allocation run |
| Policy approval | Early sign-off from head of education on tiers and lottery |

## Open items
- Who sets default tier definitions: the university or each field head?
- Are cross-branch seats allowed as alternatives for losers?



---

# v6 changes (appended): seat-allocation priority rules

**Context:** Defines the priority tiers and lottery rules used in the contested-seat round introduced in v5. Tier thresholds below are proposed defaults and can be tuned by the university or field heads.

**Terms**
- **In-parity course:** A course whose chart semester matches the current term's parity (e.g., an odd-semester course in an odd term).
- **Off-parity course:** A course opened in a term of the other parity (e.g., an odd-semester course held in an even term).
- **Near graduation:** A student whose remaining required units or terms are at or below a set threshold (open item below).

## Assessment of the proposed rules

| # | Rule | Verdict | Adjustment |
|---|---|---|---|
| 1 | Near-graduation students get priority for off-parity courses they need | Fair and useful. A lost seat costs them a whole extra year, and on-track students don't need off-parity courses. | Applies to required courses only, not electives. Needs an objective, data-derived definition of "near graduation". |
| 2 | Repeaters get lower priority in-parity and higher priority off-parity | Fair. First-timers have no other chance this year, while repeaters get the course again next year. Off-parity terms mainly exist for repeaters. | Demotion, not exclusion. Near-graduation students are exempt from the demotion, or a failed final-term course could delay graduation by a year. |
| 3 | Students who lost a lottery for a course they never enrolled in get high priority next time | Fair and useful. It stops the same student losing repeatedly and rewards waiting. | Must be earned only by actually applying and losing. Flag, not accumulating weight. Expires or is consumed on enrollment. |
| 4 | Lottery numbers retained for the waitlist | Fair and simple. No re-draw, and it is reproducible from the logged seed. | Numbers are drawn per student, per course and per round, never one global number per student. |

**Trade-off to note:** In-parity, bonus holders rank above current on-track first-timers. This favors the longest-waiting student, but it can push on-track students into next year's bonus pool. Capacity-first (adding seats) keeps this rare.

## Final rules

1. **Near-graduation priority (off-parity):** Near-graduation students who need the course (required, not elective) are top priority for off-parity courses.
2. **Repeater adjustment:** Students who failed a course are demoted one tier for it when it is in-parity, and promoted when it is off-parity. Near-graduation students are exempt from the in-parity demotion.
3. **Lottery-loser bonus:** A student who applied within the window and lost a contested draw for a course they have never enrolled in receives a priority bonus for that course next time. It is a flag, not an accumulating weight. Holders may be ordered by number of prior losses.
4. **Retained numbers:** The random number from the lottery is stored per (student, course, round) and reused for the waitlist order.
5. **Committed randomness (new):** The seed or a hash of it is committed before the round and published afterward, so the draw can be reproduced.
6. **Bonus integrity (new):**
   - Earned only by applying within the window while eligible and then losing.
   - Consumed when the student takes a seat in that course.
   - Forfeited if the student wins a seat and drops it after the add/drop deadline.
   - Expires after a set number of terms (open item).
7. **Waitlist handling (new):**
   - Order is tier first, then retained number.
   - Students who join after the window are appended in ingest-stamp order.
   - Each offer has a response deadline. No response forfeits the offer and moves the student to the end.
   - Students skipped because of a conflict or unit cap are notified with the reason.
8. **Data-derived tiers (new):** Tiers are computed from the student profile (grades, chart, remaining units), never self-declared. Overrides keep the v5 rule of a recorded reason.

### Tier order

**In-parity course**
| Tier | Who |
|---|---|
| 1 | Lottery-loser bonus holders |
| 2 | Students for whom it is on the current chart semester (first attempt), students short of minimum units, near-graduation students who need it (including repeaters) |
| 3 | Others (electives, students ahead of schedule) |
| 4 | Repeaters, unless near graduation |

**Off-parity course**
| Tier | Who |
|---|---|
| 1 | Near-graduation students who need it |
| 2 | Repeaters and lottery-loser bonus holders |
| 3 | Others |

Within each tier, order is by the retained lottery number.

## Modified from previous versions
- **v5, "Need-based priority tiers":** The criterion "previously failed or withdrawn" is superseded. Failed students are now demoted in-parity and promoted off-parity. Treatment of withdrawn students is an open item.
- **v5, "Soft landing for losers":** "Optional priority next semester" is now rule 3.
- **v5, waitlist:** Rule 4 confirms retained numbers. Rule 7 defines late joiners and offer deadlines.
- **v5, open items:** "Who sets default tier definitions" now has a proposal: the university approves the tier framework, and heads tune thresholds within set bounds. Awaiting your confirmation.
- **v4, assumptions:** "Batch allocation ... not yet confirmed" was confirmed in v5. The tiers are now defined here.
- **v4, Pros 5, last bullet:** "Random order within priority tiers" now refers to the tier tables above.
- **v4, Cons 10, "Perceived unfairness":** Tiers now exist, but rule complexity adds a new risk (see New Cons).
- **v4, Cons 7, "Eligibility rules correctness":** Grades, graduation status, and chart data now also drive priorities, so errors affect allocation directly.
- **v4, Cons 6, "Cross-branch rules":** The bonus flag and repeater history live in the student's home partition so they carry across branches.

## New Pros
- **Rewards waiting:** Lottery losers are not stuck losing repeatedly.
- **Protects graduation timelines:** Near-graduation students get first access to off-parity courses and are exempt from repeater demotion.
- **Matches seats to real need:** Off-parity terms serve the students who need them.
- **Independent draws per course:** One unlucky number cannot sink a student across all their courses.
- **Verifiable randomness:** Committed seed plus retained numbers make every outcome reproducible and auditable.
- **Predictable waitlists:** Students see their position and the reason for it.

## New Cons
- **Rule complexity:** Four tier orderings and a bonus are harder to explain, and students may misread them.
- **Definition disputes:** "Near graduation" and "needs it" thresholds will be contested.
- **Repeater starvation:** Repeaters can lose in-parity repeatedly, and may face the same demotion again.
- **Bonus displacement:** Bonus holders can push on-track students into next year's bonus pool.
- **Bonus gaming:** Students may apply to popular courses to build bonuses, or win and drop late.
- **Rule interaction bugs:** Overlapping rules (repeater and near-graduation, bonus and repeater) need exhaustive testing.
- **Bonus bookkeeping:** Storing and expiring bonuses across terms, branches, and transfers adds data and audit work.
- **Dependence on data quality:** Wrong grades or graduation status silently change who gets seats.

## New Mitigations
| Risk | Mitigation |
|---|---|
| Rule complexity | One-page published explanation, a "why is my position X" view for each student |
| Definition disputes | Objective thresholds approved in advance, derived from profile data only |
| Repeater starvation | Demotion by one tier only, near-graduation exemption, capacity-first, optional cap on consecutive demotions |
| Bonus displacement | Capacity-first, monitoring of bonus-pool size per course |
| Bonus gaming | Earned only by applying while eligible, forfeited on late drop, consumed on enrollment, expiry |
| Rule interaction bugs | Written tier specification, test cases for every rule combination, reproducible runs from logged seeds |
| Bonus bookkeeping | Store flags in the home partition, include them in audit logs and transfer procedures |
| Data quality | Validation and reconciliation of grades before each round, correction path through appeals |

## Open items
- How is "near graduation" defined (remaining units, remaining terms, or both), and who sets the threshold?
- How long does the lottery-loser bonus last before expiring?
- How are withdrawn (not failed) students treated?
- Should repeaters get a cap on consecutive in-parity demotions?
- Do bonuses carry across branches and on transfer?
- Confirm: the university approves the tier framework and heads tune thresholds.

---
# V7 changes

## New pros
- **Earlier class starts:** Because most students get their chosen courses in pre-registration, few change classes during add/drop. Lecturers can start teaching in the first week instead of waiting for the add/drop period to end.
