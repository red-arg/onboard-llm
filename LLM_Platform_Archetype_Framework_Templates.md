# LLM Platform: Archetype Framework Templates

Version 0.15 · Nov 13, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

The templates for the records the [archetype framework](LLM_Platform_Archetype_Framework.md) asks for: the profile (Step 1), the definition (Step 4), the step record, the acceptance record and the release notice. They are versioned with the framework.

**Profile**

```yaml
product: <product>
owner: <Business Owner>
use_case: <what the product asks the model to do, in one line>
status: live                          # live | planned, with the quarter
task: answer
output: prose
alias: rag-v1                         # the version the product calls
latency: interactive                  # real-time (first token within 300 ms) | interactive (1.5 s) | batch
lengths: { input_p95: 9000, output_p95: 600 }             # tokens
volume: { requests_per_day: 5000, peak_per_minute: 40 }
languages: [en]
untrusted_content: yes                # adds injection samples to the acceptance set
acceptance: { metric: answer_correctness, floor: 0.85, max_critical_errors: 0.03 }
samples: <at least 100 labeled samples, kept by the product>
cost_target_per_1k: <amount, if known>
accepted: <Business Owner, date>      # the archetype's conditions and residual risk; renewed yearly
```

**Definition** (filled in for the [text-to-SQL example](LLM_Platform_Archetype_Framework.md#example-text-to-sql))

```yaml
id: text-to-sql                       # lowercase, names the capability
owner: <archetype owner>
status: active                        # proposed | pilot | active | retired
tasks: [code]
outputs: [query]
scope: Write one read-only SQL query that answers a question over a supplied schema.
out_of_scope: [running the query, general code (code), answers from documents (rag)]
offer:
  latency: interactive
  context_length: 32k
  languages: [en, de]
  cost_target_per_1k: 20
reference:
  suite: text-to-sql/v1
  datasets:                           # public, used as published; one score over all of them
    - { dataset: <public dataset, en>, revision: <revision> }
    - { dataset: <public dataset, de>, revision: <revision> }
  other_slices: [regression, injection (held out)]
  metric: execution_accuracy
  safety_additions: [destructive statements, injections in column comments and row values]
versions:
  - { alias: text-to-sql-v1, model: <registry id>@<hash>, status: current, ga: 2027-01-18, end_date: null }
products: [analytics-assistant, finance-reporting]
conditions:
  - Queries run with the end user's own database permissions, read-only
blast_radius: 2 products, about 6,500 requests a day, both on v1; the model serves no other alias
risks:
  - risk: Instructions injected into column comments or row values, reworded when blocked
    control: Held-out injection set; yearly red-team round
    test: Red-team success no higher than in the last round
    owner: Platform data science team
  - risk: A query returns the expected shape but wrong values
    control: The G5 policy's critical-error criterion; each product's own maximum
    test: Critical-error rate on the reference set and in each acceptance record
    owner: Archetype owner
residual_risk:
  risks: [wrong values within each product's critical-error maximum]
  rationale: Each Business Owner sets the maximum for the product's use and decides who checks the results.
```

**Step record**

```yaml
archetype: text-to-sql
step: 3                               # 2 fit check | 3 capability test | 4 approval | 5 go active
decided_by: <user>                    # not the proposer
decided_at: 2026-11-20
inputs: [<profiles@commit>, <suite version>, <every run, failed runs included>]
outcome: create an archetype
suitability: { likelihood: medium, decision: proceed, business_owners_told: yes }
conditions: []
reason: <a few sentences>
```

**Acceptance record**

```yaml
profile: <profile@commit>
alias: text-to-sql-v2
harness: text-to-sql/v1               # the suite version
samples: { count: 100, injection: 20, commit: <commit> }
prompt: <product prompt@commit>
result: { execution_accuracy: 0.90, critical_errors: 0.01 }
required: { floor: 0.85, max_critical_errors: 0.03 }
outcome: pass                         # pass | pass within a tie | fail
signed_by: <Business Owner, date>
```

**Release notice**

```yaml
archetype: text-to-sql
new: text-to-sql-v2
replaces: text-to-sql-v1
reason: better on the reference eval  # better | ties at lower cost or latency | forced replacement
reference_scores:                     # same suite version for both
  text-to-sql-v1: { reference_set: 0.87, critical_errors: 0.02 }
  text-to-sql-v2: { reference_set: 0.93, critical_errors: 0.01 }
known_differences:                    # prompt format, output format, refusals
  - v2 wraps the query in a code fence unless the prompt forbids it
reference_prompt: <path@commit>
ga: 2027-08-03
end_date: 2027-11-01                  # 90 days after GA
sent_to: [<every Business Owner on the archetype>]
```
