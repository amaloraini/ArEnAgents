# ArEnAgents

A parallel Arabic–English benchmark for multi-agent LLM coordination, released with the paper
**“Multilingual Multi-Agent LLM Systems: Measuring, Diagnosing, and Mitigating the Arabic–English Coordination Gap”** (IEEE Access, under review).

ArEnAgents is a benchmark for evaluating teams of LLM agents that must coordinate through natural-language messages. Every scenario is generated from language-neutral identifiers and rendered in both **Modern Standard Arabic** and **English** from parallel templates, with no machine translation. Outcomes are scored programmatically against gold answers that are unique by construction, so Arabic and English teams can be compared on identical tasks.

This repository contains the **benchmark data** (scenarios, rendered prompts and gold answers), the **prompt templates and bilingual lexicon** for every experimental arm, and the **frozen-evidence adaptation of the MARBLE database-diagnosis task** used in the paper. The evaluation harness, scenario generator and analysis notebook are not part of this release; the scoring rules are documented below so that results can be reproduced with an independent implementation.

## Contents

```
data/
  arenagents/
    arenagents_train.jsonl   600 scenarios (150 per task)   seeds 300000–300149
    arenagents_dev.jsonl     120 scenarios ( 30 per task)   seeds 200000–200029
    arenagents_test.jsonl    240 scenarios ( 60 per task)   seeds 400000–400059
  marble_db/
    marble_db_evidence.json  100 frozen-evidence tasks adapted from MARBLE
    marble_db_prompts.jsonl  rendered expert prompts (EN and AR) for the 100 tasks
prompts/
  templates_p6.json          all prompt templates and the bilingual lexicon (frozen version p6)
  README.md                  how prompts are assembled for every experimental arm
DATASHEET.md                 datasheet for the dataset
CITATION.cff                 how to cite
LICENSE                      CC BY 4.0 (ArEnAgents data)
LICENSE-MARBLE               MIT notice for MARBLE-derived material
SHA256SUMS                   checksums of the data files
```

Seed ranges of the three splits are disjoint. The pre-registered confirmatory evaluation in the paper uses the **test** split only; the train and dev splits were used for development, prompt freezing and the exploratory fine-tuning experiments.

## Tasks

| Task | Agents | Goal | Gold answer |
|---|---|---|---|
| `meeting` | 3 | Find the single time slot (out of 10) at which all three colleagues are free; each knows only its own busy slots | one slot |
| `plan` | 3 | Order five conference-preparation steps; five precedence rules are split among the organisers (two, two and one) | the full order of five steps |
| `incident` | 3 | Identify the one compromised host and the one attack type; the host is the only one flagged by every analyst's sensor, and only one analyst knows the attack type | `host, attack type` |
| `negotiation` | 2 | Seller and buyer bargain over a used car with private reservation prices | no gold answer; scored by outcome |

Meeting, plan and incident scenarios are generated so that **no single agent's private view determines the answer** (`single_agent_solvable` is empty for every record). Each released prompt is one agent's system prompt in the base team condition; it contains only that agent's private information.

## Record format (`arenagents_*.jsonl`)

One JSON object per line:

| Field | Type | Meaning |
|---|---|---|
| `task` | string | `meeting`, `plan`, `incident` or `negotiation` |
| `seed` | int | scenario seed; `(task, seed)` is the scenario identifier |
| `split` | string | `train`, `dev` or `test` |
| `generator_version`, `prompt_version` | string | `g2` / `p6` for every record (the frozen, pre-registered versions) |
| `n_agents` | int | 3 (cooperative tasks) or 2 (negotiation) |
| `data` | object | the language-neutral scenario (see below) |
| `solution` | varies | the gold answer in identifier form |
| `gold_label` | object | `{"en": ..., "ar": ...}` — the gold answer as natural-language text, exactly as a valid final answer would read; `null` for negotiation |
| `single_agent_solvable` | list | always empty: no agent's private view alone determines the answer |
| `prompts` | object | `{"en": [...], "ar": [...]}` — one system prompt per agent, in agent order, in each language |

### `data` by task

**meeting** — `busy`: list of three lists, the busy slots of each agent. Slot identifiers: `sun_am, sun_pm, mon_am, mon_pm, tue_am, tue_pm, wed_am, wed_pm, thu_am, thu_pm`. `solution` is the unique slot free for all three agents.

**plan** — `steps`: the five step identifiers in the scenario; `rules`: list of three lists (one per agent) of `[before, after]` pairs. The five rules (2 + 2 + 1) consist of four rules that determine the hidden order and one redundant rule. `solution` is the list of the five step identifiers in the correct order; `gold_label` joins the rendered step names with `>`.

**incident** — `scope`: the hosts in scope; `flags`: list of three lists, the hosts flagged by each analyst's sensor; `type_holder`: index of the agent whose evidence reveals the attack type; `benign`: per-agent index of a benign distractor observation (`null` for none). `solution` is `[host, attack_type_id]`; `gold_label` renders it as `host, attack type`.

**negotiation** — `list`: listed price; `rs`: the seller's reservation (minimum) price; `rb`: the buyer's reservation (maximum) price; `roles`: `["seller", "buyer"]` in agent order. `solution` is `{"zopa": true|false}`, with `zopa = (rs <= rb)`. A ZOPA exists in 50/60 test, 24/30 dev and 103/150 train scenarios. Prices are in riyals.

## Interaction protocol and scoring rules

These rules describe how the paper's harness runs and scores an episode. An independent implementation that follows them should reproduce the paper's success criteria.

**Turns.** Agents speak in round-robin order on a shared channel for at most *R* rounds (*R* = 4 for cooperative tasks, 5 for negotiation). On each turn an agent receives its system prompt (its communication language, its private information only) and the transcript of all previous public messages. It must reply with a private `<notes>` block (≤ 60 words, never shown to others) and a public `<message>` block (≤ 120 words; 80 in negotiation). Everything outside a closed `<notes>` block is public.

**Language conditions.** *EN*: all agents prompted in English. *AR*: all agents prompted in Arabic. *MIX*: agents are assigned different languages and read teammates' messages as written. The released `prompts` fields are the base team prompts; the paper's other arms (single agent with all information, shared briefings, notes-language 2×2, JSON protocol, labelled-text control) are assembled from the same templates as described in `prompts/README.md`, and the per-turn user messages (round counter, transcript, last-round reminder) are there too.

**Final answers (cooperative tasks).** A final answer is a line beginning with the marker `FINAL:` (English) or `النهائي:` (Arabic) inside the public message. The marker counts only at a sentence boundary; it is ignored when introduced by a negation or an example ("Do not submit this answer: FINAL: …"); `FINAL: None` is not an answer. Misspelled markers and phrases outside the marker set are not accepted in team episodes.

**Strict validation.** The answer text must name exactly the required entities — one slot; one host and one attack type; all five steps, each exactly once; — every entity must belong to the scenario, and any line with negation or exclusion cues in either language is rejected. Invalid answers are never guessed; they are logged with a reason code (missing, negation, incomplete, ambiguous, wrong number, duplicate, out of scope) and reported separately from valid-but-wrong answers. A later invalid answer does not cancel an earlier valid one. Matching normalises Arabic orthography (hamza forms, tā' marbūṭa, diacritics) and Arabic-Indic digits; agents never see normalised text.

**Episode outcome (cooperative tasks).** *Consensus* = all agents hold the same valid final answer. The episode ends at consensus or at the round limit. *Success* = consensus on the gold answer.

**Negotiation.** The last line of each message must be exactly one action: `OFFER: <price>` / `عرض: <السعر>`, `ACCEPT` / `قبول`, or `WALK AWAY` / `انسحاب`. A line listing several actions is logged as ambiguous and not executed. If an offer matches or crosses the counterpart's standing offer (a seller asks no more than the buyer's offer, or a buyer bids at least the seller's offer), it counts as implicit acceptance and the deal closes at the counterpart's price. The episode ends with acceptance, walk-away or the round limit. An outcome is *rational* if a valid deal is reached when a ZOPA exists, or no deal is reached when it does not. Also reported: deal rate, reservation violations (a closed deal outside a party's reservation value, or any offer that breaks the offering agent's own reservation value), and the seller's surplus share (p − rs)/(rb − rs).

**Statistics.** The scenario `(task, seed)` is the unit of analysis; every scenario is run under every condition, so comparisons between conditions are paired.

## MARBLE-DB (`marble_db_evidence.json`)

A **frozen-evidence bilingual adaptation** — not a reproduction — of the database-diagnosis task from MultiAgentBench / MARBLE (Zhu et al., ACL 2025; MIT licence; upstream commit `8d60fa17b5596b44458a52d4296061b9fc13d6f2`). MARBLE's 100 tasks, anomaly configurations, gold root causes, expert roles, the "choose two" rule and the score |Ŷ ∩ Y| / |Y| (called recall@2 in the paper) are kept. Instead of live SQL tools, each expert receives the frozen output of the diagnostic queries MARBLE recommends for its hypothesis, captured after re-running MARBLE's anomaly workload on a fresh PostgreSQL 16 database for each task.

```
{
  "meta": { marble_commit, generator_version, scale, scale_params, postgres, workload_schema, generated, note, n_tasks, n_quarantined },
  "quarantined": {},            # tasks that failed validation (none)
  "tasks": {
    "1": {
      "root_causes":      ["INSERT_LARGE_DATA"],              # gold set; 50 tasks have one cause, 50 have two
      "evidence":         { "INSERT_LARGE_DATA": "...", "LOCK_CONTENTION": "...", "VACUUM": "...",
                            "REDUNDANT_INDEX": "...", "FETCH_LARGE_DATA": "..." },   # one frozen query transcript per expert
      "tables":           [...],                              # tables created by the task's init SQL
      "source_anomalies": [...],                              # upstream anomaly configuration
      "workloads":        [...],                              # parameters actually executed and load statistics
      "init":             { "init_statements": 23 },
      "fingerprint":      "<sha256>",                         # provenance: upstream commit, generator, effective load, PostgreSQL version, init and diagnostic SQL
      "captured":         "2026-10-04T13:28:10+0000"
    }, ...
  }
}
```

Each `evidence` value is a transcript of `SQL> <query>` lines followed by the result rows, as shown to the expert holding that hypothesis. Root-cause labels: `INSERT_LARGE_DATA`, `FETCH_LARGE_DATA`, `LOCK_CONTENTION`, `REDUNDANT_INDEX`, `VACUUM` (30 occurrences each).

Deviations from upstream, and why: (1) workloads run at a **reduced scale** (`meta.scale_params`), with the effective parameters recorded per task; (2) synthetic workload tables live in a separate schema (`wl`) because 20 upstream tasks define their own `orders` table, which silently broke MARBLE's fetch workload; statement text is unchanged via `search_path`; (3) MARBLE's VACUUM query reports execution time in addition to planning time; (4) evidence is retrospective (captured after the workload completes). Known limitation: generator artefacts (statement text and case, index names) allow surface shortcuts, so the paper reports a pooled-evidence single-agent baseline next to team results. `marble_db_prompts.jsonl` contains, for each task, the five expert prompts in English and in Arabic (`prompts.en[i]`, `prompts.ar[i]`, expert order `INSERT_LARGE_DATA, LOCK_CONTENTION, VACUUM, REDUNDANT_INDEX, FETCH_LARGE_DATA`), together with `root_causes` and the task `fingerprint`; the evidence inside them is language-neutral SQL output. Scoring: the team's strict-plurality two-cause answer (ties = no decision) is scored by recall@2 and exact-set accuracy; agents speak for up to three rounds.

MARBLE-derived material keeps its MIT notice (`LICENSE-MARBLE`).

## Intended use and limitations

- Intended for evaluating LLM agent teams in MSA and English on verifiable coordination tasks, and for studying language effects in multi-agent settings.
- Not intended for claims about human speakers, for dialectal Arabic (MSA only), or for comparison with the MARBLE leaderboard (the adaptation changes the tool interface and the workload scale).
- The scenarios are synthetic; names, hosts and prices are fictional.

## Licence

ArEnAgents data: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (`LICENSE`). MARBLE-derived material: MIT (`LICENSE-MARBLE`).

## Citation

See `CITATION.cff`.
