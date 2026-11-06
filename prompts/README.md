# Prompt templates (version p6)

`templates_p6.json` contains every prompt template and the bilingual lexicon used to render the benchmark, exactly as frozen for the pre-registered run. All released prompts in `data/arenagents/*.jsonl` and `data/marble_db/marble_db_prompts.jsonl` are reproduced byte-for-byte from these templates (verified for all 5,280 ArEnAgents prompts).

```json
{
  "prompt_version": "p6",
  "templates": { "<key>": { "en": "...", "ar": "..." }, ... },
  "lexicon":   { "AGENT_NAMES": ..., "DAYS": ..., "PERIODS": ..., "SLOTS": ..., "HOSTS": ..., "ATTACKS": ...,
                 "ATTACK_OBS": ..., "BENIGN_OBS": ..., "STEPS": ..., "RC_INFO": ..., "NEG_EN": ..., "NEG_AR": ... }
}
```

Every template exists in English (`en`) and Modern Standard Arabic (`ar`); the two are parallel renderings of the same content, not translations of each other's output.

## Template keys

| Key | Role |
|---|---|
| `meeting`, `plan`, `incident`, `nego_seller`, `nego_buyer`, `marble_db` | task description + the agent's private information (the *body*) |
| `plan_rule`, `inc_flag`, `inc_obs`, `inc_benign` | one line of private information |
| `team_lang` | language instruction: "Teammates may write in English or Arabic. Always write your message in {C}." |
| `fmt_free` | reply format for the free-text protocol (notes block + message block + `FINAL:` line) |
| `fmt_json` | reply format for the JSON protocol (message is a JSON object with keys facts / proposal / final; Arabic keys المعلومات / الاقتراح / النهائي) |
| `fmt_fields` | reply format for the labelled-text control (Facts: / Proposal: / FINAL: lines) |
| `fmt_nego` | reply format for negotiation (OFFER / ACCEPT / WALK AWAY actions) |
| `shared_briefings`, `central_head` | wrapper and header used when an agent also receives teammates' briefings |
| `central`, `fmt_central`, `central_turn` | the single-agent arm: one analyst receives all briefings and answers alone |
| `turn_first`, `turn`, `last_round`, `last_round_nego` | per-turn user messages (round counter, transcript, last-round reminder) |

Placeholders: `{name}`, `{n}` (team size), `{C}` (communication language name), `{R}` (notes language name), `{L}` (language name for the single agent), `{R}`/`{r}` in turn templates (round limit / current round), `{transcript}`, `{briefings}`, and task-specific fields (`{slots}`, `{busy}`, `{steps}`, `{facts}`, `{hosts}`, `{types}`, `{list}`, `{rs}`, `{rb}`, `{tables}`, `{rcs}`, `{hyp}`, `{evidence}`).

## Lexicon

- `AGENT_NAMES` — agent *i* is named `AGENT_NAMES[lang][i]` (Sami/سامي, Lina/لينا, Omar/عمر, Huda/هدى, Yusuf/يوسف).
- `DAYS`, `PERIODS`, `SLOTS` — the ten meeting slots; slot `tue_am` renders as "Tuesday morning" / "الثلاثاء صباحًا".
- `STEPS` — plan step id → (EN label, AR label, EN match keys, AR match keys); the match keys are what the paper's parser uses to recognise a step in a final answer.
- `HOSTS`, `ATTACKS`, `ATTACK_OBS`, `BENIGN_OBS` — incident entities; attack observations deliberately avoid the attack-type keywords so the type must be inferred.
- `RC_INFO` — MARBLE-DB root-cause code → (EN gloss, AR gloss, match keys); codes stay as technical identifiers in both languages.
- `NEG_EN`, `NEG_AR` — negation/exclusion cues; a final-answer line containing any of them (after removing recognised entity labels) is rejected as invalid rather than interpreted. Arabic cues are in normalised form (hamza forms unified, tā' marbūṭa → hā', diacritics removed).

## How a system prompt is assembled

For agent *i* with communication language `lc` and notes language `lr` (equal to `lc` except in the 2×2 arm):

```
body                                   # task template filled with the agent's private facts, in lc
<blank line>
team_lang[lc].format(C = LANG_NAME[lc][lc])
<blank line>
fmt[lc].format(R = LANG_NAME[lc][lr], C = LANG_NAME[lc][lc])     # fmt = fmt_free | fmt_json | fmt_fields | fmt_nego
```

Lists are joined with `LIST_SEP[lc]` (", " in English, "، " in Arabic); prices are formatted with thousands separators (`50,000`).

Arms used in the paper:

- **Team (EN / AR / MIX)** — the assembly above with `fmt_free`; in MIX each agent gets its own `lc` and reads teammates' messages as written. This is what the released `prompts` fields contain.
- **Shared briefings** — the team prompt followed by `shared_briefings[lc]`, whose `{briefings}` is the concatenation, for every other agent *k*, of `central_head[lc]` with agent *k*'s name plus agent *k*'s body rendered in `lc`.
- **Single agent (central)** — `central[lang]` with `{briefings}` = all agents' bodies (each under a `central_head`), followed by `fmt_central[lang]`; the user turn is `central_turn[lang]`.
- **Notes-language 2×2** — the team prompt with `lr ≠ lc`, so `{R}` names a different language from `{C}`.
- **JSON protocol / labelled-text control** — the team prompt with `fmt_json` / `fmt_fields` in place of `fmt_free`.
- **MARBLE-DB** — five experts in the fixed order `INSERT_LARGE_DATA, LOCK_CONTENTION, VACUUM, REDUNDANT_INDEX, FETCH_LARGE_DATA`; expert *i* investigates hypothesis *i* and receives `evidence[hyp]` from `marble_db_evidence.json`. `{rcs}` lists each code with its gloss, e.g. `VACUUM (improper VACUUM operations)`. The rendered prompts are in `data/marble_db/marble_db_prompts.jsonl`.

Per-turn user messages: round 1 uses `turn_first`; later rounds use `turn` with the public transcript so far; in the final round `last_round` (or `last_round_nego`) is appended.
