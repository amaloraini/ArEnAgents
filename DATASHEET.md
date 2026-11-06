# Datasheet for ArEnAgents

Following Gebru et al. (2021), *Datasheets for Datasets*.

## Motivation

**For what purpose was the dataset created?** To measure whether teams of LLM agents coordinate as well in Modern Standard Arabic (MSA) as in English on identical, verifiable tasks, and to study mixed-language teams in which agents read each other's messages without translation. Existing multi-agent benchmarks are predominantly English and, where multilingual, rely on machine translation of agent messages.

**Who created the dataset?** The authors of the accompanying paper (see `CITATION.cff`).

**Who funded it?** No external funding.

## Composition

**What do the instances represent?** Each ArEnAgents instance is one coordination scenario: a language-neutral specification (identifiers only), one system prompt per agent in English and in Arabic rendered from parallel templates, and the gold answer. Each MARBLE-DB instance is one database-diagnosis task with frozen diagnostic evidence and its gold root causes.

**How many instances?** ArEnAgents: 960 scenarios in four tasks (meeting, plan, incident, negotiation) — 150 train, 30 dev and 60 test seeds per task, with disjoint seed ranges. MARBLE-DB: 100 tasks, none quarantined.

**Is it a sample or the whole population?** Scenarios are generated from seeds; the released splits are the frozen, pre-registered sets (generator `g2`, prompts `p6`) used in the paper. Any number of further scenarios could be generated, but the released seeds are the ones on which the reported results were obtained.

**What data does each instance consist of?** See `README.md` for the field-level description. Prompts are plain text; there are no images, audio or personal data.

**Is there a label or target?** Yes. Cooperative tasks have a unique gold answer (verified by brute force for the plan task; the free slot and the commonly flagged host are unique by construction). Negotiation has a ZOPA flag and the two reservation prices; outcomes are scored by rationality and constraint adherence, not against a gold answer. MARBLE-DB keeps MARBLE's gold root causes (50 tasks with one cause, 50 with two).

**Information requirement.** Meeting, plan and incident scenarios are generated so that no single agent's private view determines the answer (`single_agent_solvable` is empty for every record).

**Are there recommended data splits?** Yes: train / dev / test. The paper's confirmatory results use the test split only, with the generator and prompt versions frozen and their digests recorded in the pre-registration before the run.

**Errors, noise, redundancies.** The plan task includes one deliberately redundant rule per scenario. The MARBLE-DB evidence contains generator artefacts (statement text and case, index names) that permit surface shortcuts; a pooled-evidence single-agent baseline should be reported alongside team results.

**Is the dataset self-contained?** Yes; it also includes the frozen prompt templates and lexicon (`prompts/templates_p6.json`) from which every released prompt is rendered. MARBLE-DB material is derived from MultiAgentBench/MARBLE (Zhu et al., ACL 2025; MIT licence; commit `8d60fa17`) and is redistributed under the MIT notice in `LICENSE-MARBLE`.

**Confidential or sensitive content?** None. All content is synthetic: fictional colleague names, host identifiers, car prices and PostgreSQL statistics from a throw-away database.

## Collection process

**How was the data generated?** By a seeded generator that produces language-neutral scenarios (time slots, step and host identifiers, attack-type codes, reservation prices) and renders them through parallel English and Arabic templates. No machine translation is used at any stage. Arabic templates were written and reviewed by a native Arabic speaker among the authors. Every gold answer, in both languages, round-trips through the paper's answer parser back to the original solution (1,640 of 1,640: 240 per cooperative task and language, and 100 per language for MARBLE-DB).

**MARBLE-DB evidence** was captured on 4 October 2026 by re-running MARBLE's anomaly workloads at a reduced scale on a fresh PostgreSQL 16.15 instance per task and recording the output of MARBLE's recommended diagnostic queries for each hypothesis. Workload tables are created in a separate schema (`wl`) so that upstream tasks defining their own `orders` table do not break the fetch workload; every workload is validated for operation counts, errors, row counts and created indexes, and each task carries a provenance fingerprint over the upstream commit, generator version, effective load, PostgreSQL version, and the initialisation and diagnostic SQL.

**Who was involved?** The authors only. No crowd workers or human annotators.

**Ethical review.** Not applicable; no human subjects and no personal data.

## Preprocessing

Scenario records are released as generated. The MARBLE evidence file is released as captured, pretty-printed. Answer matching in the paper normalises Arabic orthography (hamza forms, tā' marbūṭa, diacritics) and Arabic-Indic digits; this normalisation is applied at scoring time only and is not reflected in the released text.

## Uses

**Has the dataset been used already?** Yes, in the accompanying paper, for a pre-registered comparison of English, Arabic and mixed-language teams of five open-weight 7–9B models, plus exploratory analyses.

**What other uses are there?** Evaluation of other models and agent frameworks; study of cross-lingual communication between agents; the train split may be used for fine-tuning or self-play.

**What should it not be used for?** Claims about human speakers of either language; dialectal Arabic (MSA only); comparison with the MARBLE leaderboard, because the adaptation replaces live SQL tools with frozen evidence and reduces the workload scale.

## Distribution

**How is it distributed?** As this repository. ArEnAgents data: CC BY 4.0. MARBLE-derived material: MIT.

**Export controls or other restrictions?** None.

## Maintenance

**Who maintains it?** The authors; contact via the corresponding author in `CITATION.cff`.

**Will it be updated?** Corrections, if any, will be released as tagged versions; the released seeds and the `g2`/`p6` versions will not change, so that results remain comparable with the paper.

**Can others contribute?** Issues and pull requests are welcome for documentation and additional language renderings; changes to the scenario content will be versioned separately.
