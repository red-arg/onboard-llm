# Framework Design Standard for AI, Software and Product

Oct 30, 2026 · @Zhi Feng

Use this standard to design any AI, software or product framework, such as governance, assurance, testing, delivery or launch readiness. A compliant framework follows the eight principles, contains the required sections and passes the release gate.

## How to apply

- **Must** is required. **Should** is required unless you record why not.

- Scale depth to risk. Every Must applies, but a low-risk framework may meet one in a line, while a high-risk one needs the full artefact.

- Work through the build sequence, then check the result against the release gate.

## The eight principles

Each principle lists its rules, then common failures to avoid.

### 1. Subject definition and scope

- **Must** model the subject: components, interfaces, actors, and where controls and logs sit. Map every requirement to a component.

- **Must** name the two or three risk dimensions with the most effect on risk, such as impact, reversibility, data sensitivity, autonomy or external exposure. Describe archetypes along them and say which are out of scope.

- **Must** state scope, exclusions (with where to go instead) and audiences.

- **Must** state what carries over from existing frameworks and what is new, with a crosswalk.

- *Avoid:* rigid definitions that become outdated; forcing mixed cases into one archetype.

### 2. Risk assessment

- **Must** use one risk taxonomy across the framework and its companions: sources by component, five to seven harm types, accidental versus adversarial, and system-level effects.

- **Must** gate suitability: rate likelihood and impact on named factors, then weigh against benefit. "Do not proceed" and "use something simpler" are valid outcomes.

- **Must** turn each risk into concrete scenarios covering outcome and process, including an adaptive attacker. Trace each scenario to at least one control and one test.

- **Must** have residual risk accepted by a named owner, with a rationale and a review date.

- *Avoid:* risk lists not mapped to testable scenarios.

### 3. Risk-based tiering and phased rollout

- **Must** tier obligations by risk — approvals, testing depth, monitoring and change review — and keep lowest-tier requirements minimal.

- **Must** phase any expansion of users, scope or autonomy behind entry criteria: evidence gathered and next-phase controls in place.

- **Should** record each heavy control's friction and cost, and prefer a lighter control that gives equal assurance.

- *Avoid:* uniform process regardless of risk.

### 4. Control design

- **Must** use the strongest feasible control, in this order: remove the hazard, enforce structurally (permissions, code, architecture), check automatically, monitor and escalate to people, instruct.

- **Must** give every high-risk action at least one control that is not an instruction.

- **Must** apply least privilege: default deny, scoped and expiring grants, delegation never beyond the delegator's authority, every grant recorded.

- **Must** give every actor an identity, an owner, an inventory entry and tamper-resistant logs, and retire what is no longer needed.

- **Must** design for failure: a stated blast radius, fail-closed defaults, a shutdown path and a manual fallback.

- **Must** treat interfaces — between components, services, organisations and people — as control points with defined contracts.

- *Avoid:* instruction-only controls for high-risk actions; a single actor with broad privileges.

### 5. Accountability and human oversight

- **Must** map the value chain and assign responsibilities by role and lifecycle stage, including supplier obligations for transparency, logging, security and data. The deploying organisation stays accountable.

- **Must** specify each human checkpoint: its trigger (high-stakes, irreversible, atypical or user-set), the information shown, the decision options and the expertise required.

- **Must** measure oversight through override rates, review times and decision accuracy, and test it, for example with seeded errors.

- **Must** tailor information by audience: transparency at the point of use for those affected; training for operators and reviewers.

- **Should** plan for automation bias, fatigue and skill erosion, keep manual fallbacks practised, and name who keeps the framework current.

- *Avoid:* human approval steps that are never measured or tested; disclosures only in separate documents.

### 6. Verification and testing

- **Must** verify actual effects and the process taken, not outputs or the system's own account.

- **Must** pair every control with a test that tries to break it, including adversarially.

- **Must** test in realistic but contained environments, at component and system level, repeated enough to capture variable behaviour, with configurations recorded.

- **Must** set a metric, pass gate, evaluator and baseline for each scenario. Automate evaluation where possible, keep humans owning ground truth, and weight errors by consequence.

- **Must** design logging backward from the scenarios: the signal that shows each one, where it is captured and who can alter it.

- *Avoid:* assuming a configured control works; accepting a single passing run as evidence of reliability.

### 7. Lifecycle operations

- **Must** assign obligations to every stage: design, pre-release, rollout and operation. Pre-release assurance alone is not enough.

- **Must** keep a monitoring playbook: signal → threshold → severity → response → owner.

- **Must** define change triggers (technical, environmental, performance, regulatory) and scale review depth to the change. Re-test after material change and grow a regression set from incidents.

- **Must** route findings from tests, monitoring, incidents and users back into risks, controls, tests and the framework itself.

- *Avoid:* ending monitoring and testing at release.

### 8. Framework packaging and maintenance

- **Must** pair each principle with a worked example and a reusable template, and include one end-to-end walkthrough on a realistic system.

- **Must** separate requirements from adaptable guidance, and mark example lists as illustrative.

- **Must** share one vocabulary — subject model, taxonomy, terms — across companion documents, with one owner per definition.

- **Must** carry a version, changelog, known gaps, feedback channel and review date.

- **Should** pilot with practitioners on real systems before release, and credit contributors.

- *Avoid:* principles without examples or templates; undocumented limitations.

## Build sequence

Nine steps, each producing named artefacts; a finding at any step reopens the earliest step it affects. Numbers in brackets point to the principle.

1.  **Frame** — subject model, risk dimensions, archetypes, scope, audiences, crosswalk. (1)

2.  **Assess** — risk taxonomy, suitability decision, scenario register. (2)

3.  **Tier** — tier table; phase plan with entry criteria. (3)

4.  **Control** — control catalogue mapped to scenarios; permission, identity and failure design; interface contracts. (4)

5.  **Assign** — responsibility matrix, checkpoint specifications, audience information plan. (5)

6.  **Assure** — test plan with break tests, metrics and gates, evidence design. (6)

7.  **Operate** — lifecycle matrix, monitoring playbook, change taxonomy, feedback routes. (7)

8.  **Package** — the document in the required structure, with examples, templates and a walkthrough. (8)

9.  **Validate and release** — pilot, independent review, residual-risk sign-off, version 1.0. (2, 6, 8)

## Required structure

Every framework document contains these sections. Sections 7–9 may sit inside the pillars if every element is covered.

| \#  | Section        | Contains                                                                                             |
|-----|----------------|------------------------------------------------------------------------------------------------------|
| 1   | Header         | Title, version, date, owner                                                                          |
| 2   | Summary        | Purpose, why now, the pillars in brief                                                               |
| 3   | What's new     | Changes by section and where they came from, from the second version on                              |
| 4   | Scope          | In and out of scope, audiences, related frameworks, crosswalk                                        |
| 5   | Subject model  | Components, interfaces, risk dimensions, archetypes                                                  |
| 6   | Risks          | Taxonomy, suitability gate, scenario method, residual-risk acceptance                                |
| 7   | Framework      | Three to five pillars or steps, each with rules, tiered requirements, controls, tests and an example |
| 8   | Accountability | Value chain, responsibilities, checkpoints, audience information                                     |
| 9   | Operations     | Lifecycle obligations, monitoring playbook, change triggers, feedback routes                         |
| 10  | Known gaps     | Open questions, interim practices, planned work                                                      |
| 11  | Annexes        | Templates, end-to-end walkthrough, resources, feedback channel, contributors                         |

## Release gate

Publish a framework only when all seven conditions hold.

1.  Every Must is met, at a depth matched to the subject's risk.

2.  Every scenario traces to a control, a test and an owner.

3.  Every control has a recorded break-test result.

4.  Residual risk is accepted by a named owner.

5.  The framework has been piloted on at least one real system, and the findings applied.

6.  An independent reviewer has checked it against its sources and this standard.

7.  Version, changelog, known gaps, feedback channel and review date are set.

## Terms

Definitions used in this standard and in every framework built with it.

| Term           | Meaning                                                                    |
|----------------|----------------------------------------------------------------------------|
| Subject        | The system, product or process a framework governs                         |
| Actor          | Anything that acts: a person, service, model or agent                      |
| Risk dimension | A dimension that changes the consequence of failure, such as reversibility |
| Archetype      | A reference configuration defined by the risk dimensions; not exhaustive   |
| Scenario       | A specific failure or attack sequence: actor, trigger, steps and impact    |
| Tier           | A risk level with its own obligations                                      |
| Gate           | A condition that must hold before work proceeds                            |
| Break test     | A test that tries to defeat a control                                      |
| Blast radius   | Maximum impact of a single failure or compromise                           |
| Residual risk  | Risk left after controls, accepted by a named owner                        |

## About this standard

Version 1.0, derived from [Designing Frameworks](https://claude.ai/artifact/9uKyx4DNR7WGNQi1dFpZoB), an analysis of IMDA's *Model AI Governance Framework for Agentic AI* (v1.5) and *Starter Kit for Testing Agentic Systems for Safety and Reliability* (v1.0). Revise it when a framework built with it exposes a gap, and record each change here.
