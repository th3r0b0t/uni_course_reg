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
