# ThaqalaynWordSources

Sacred archive of raw data sources and LLM outputs that feed the
**Thaqalayn Words** project. This repo is the input side; the served
output lives in [`ThaqalaynWords`](https://thaqalaynwords.netlify.app/),
built by [`ThaqalaynDataGenerator`](../ThaqalaynDataGenerator/).

Nothing here is stripped or modified post-hoc — what the LLM emitted is
what's persisted. Build-time normalisation happens only in the merger
when writing to `ThaqalaynWords/`.

## Layout

```
ThaqalaynWordSources/
├── extracted/                  # Corpus surface-form set extracted from
│                                 ThaqalaynDataSources hadith responses
│                                 (corpus_surface_set.json, ~39 MB)
├── sources/                    # Raw third-party lexicon data
│   ├── lanes-lexicon/            Perseus TEI XML + parsed structured JSON
│   ├── wiktextract-arabic/       Kaikki.org Arabic Wiktionary dump (slim)
│   ├── quranic-arabic-corpus/    QAC v0.4 morphology
│   └── hawramani-classical/      38-lexicon aggregator (parsed JSON only;
│                                 raw HTML dump is gitignored)
└── translation/                # Path B Spark translation outputs
    ├── pilot_set.json            100+100 stratified pilot (random.seed locked)
    ├── pilot_set_surface_lemmas.json   side-pilot of 90 lemmas
    ├── lemma_responses/          per-slug Spark output (~13K files, sacred)
    │   └── round-{1,2}/            pilot-round artifacts
    └── surface_responses/        per-slug Spark output (~102K files, sacred)
        └── round-{3,4}/            pilot-round artifacts
```

The following are deliberately **not committed** (gitignored —
regenerable from the state above + the generator scripts):

- `translation/lemma_prompts.jsonl` (~25 MB)
- `translation/surface_prompts.jsonl` (~332 MB)
- `translation/surface_contexts.json` (~32 MB)
- `sources/wiktextract-arabic/wiktextract_corpus_lemmas.json` (~155 MB)
- `sources/hawramani-classical/raw/` (~140 MB of small HTML files)

## End-to-end pipeline order (Path B = LLM translation)

The translation pipeline lives in
[`ThaqalaynDataGenerator/scripts/`](../ThaqalaynDataGenerator/scripts/).
Full chain run via `regen_words.ps1 -IncludeTranslations`. Each stage
is **idempotent** — per-slug response files on disk are the resume
checkpoint, so re-running picks up where it left off.

```
┌───────────────────────────────────────────────────────────────────────────────┐
│ INPUTS (already on disk before Path B runs)                                   │
│  • ThaqalaynDataSources/        hadith corpus + AI content                    │
│  • ThaqalaynData/               built shipped corpus (verse text)             │
│  • ThaqalaynWordSources/extracted/corpus_surface_set.json                     │
│  • ThaqalaynWordSources/sources/  Lane's, Wiktextract, QAC, hawramani         │
└───────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 1 — build_word_pages.py (regen_words.ps1 default stage)              │
│ Walks corpus_surface_set + CAMeL Tools + all sources/, writes:             │
│   ThaqalaynWords/{lemmas,surfaces,roots}/*.json                            │
│ At this point `translations: null` on every lemma. ~30 min on full corpus. │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 2 — extract_lemma_translation_prompts.py                             │
│ Reads ThaqalaynWords/lemmas/*.json (slug, pos, Wiktextract gloss,          │
│   Lane's body, hawramani classical_definitions).                           │
│ Writes ../ThaqalaynWordSources/translation/lemma_prompts.jsonl             │
│   (gitignored — regenerable)                                               │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 3 — run_path_b_translations.py --pass lemma                          │
│ Streams lemma_prompts.jsonl → Spark Qwen 3.6-35B → 11-lang glosses.        │
│ Writes one file per lemma:                                                 │
│   ../ThaqalaynWordSources/translation/lemma_responses/{slug}.json          │
│   ★ SACRED — irreproducible LLM output, never overwritten on resume        │
│ ~1-3 h Spark wall time, $0.                                                │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 4 — extract_corpus_contexts.py                                       │
│ Walks ThaqalaynWords/surfaces/*.json + ThaqalaynData/books/*.json,         │
│ tokenises each surface's occurrence_paths, slices ±10-word windows.        │
│ Writes ../ThaqalaynWordSources/translation/surface_contexts.json           │
│   (gitignored — regenerable)                                               │
│ ~10-20 min one-time CPU walk.                                              │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 5 — extract_surface_translation_prompts.py                           │
│ Reads ThaqalaynWords/surfaces/*.json + lemma_responses/*.json (anchor)     │
│   + surface_contexts.json.                                                 │
│ Writes ../ThaqalaynWordSources/translation/surface_prompts.jsonl           │
│   (gitignored — regenerable)                                               │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 6 — run_path_b_translations.py --pass surface                        │
│ Same shape as Stage 3, surfaces side. ~6-11 h Spark wall time, $0.         │
│ Writes ../ThaqalaynWordSources/translation/surface_responses/{slug}.json   │
│   ★ SACRED                                                                 │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 7 — merge_translations_into_pages.py                                 │
│ Reads {lemma,surface}_responses/*.json, folds glosses into the matching    │
│ ThaqalaynWords pages as `translations: {…11 langs…}` +                     │
│ `translations_attribution: {…model, date, version…}`.                      │
│ Skip-on-existing by default; --overwrite to replace.                       │
└────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Stage 8 — build_word_indexes.py                                            │
│ Re-walks ThaqalaynWords/{lemmas,surfaces,roots}/ and writes                │
│   ThaqalaynWords/index/{lemmas,surfaces,roots}.json                        │
│ When translations exist, lemmas index gains an 11-lang `glosses` map.      │
└────────────────────────────────────────────────────────────────────────────┘
```

## Reproducibility

Two clean recovery scenarios:

1. **Lost a merge / want to rebuild ThaqalaynWords from scratch.**
   You only need the response files in this repo + the existing
   ThaqalaynWords scaffolding. Run stages 7 and 8 (merger + index
   rebuild). No LLM calls.

2. **Lost everything (both `ThaqalaynWords` and `translation/`).**
   Run the full pipeline from Stage 1. Stages 2/4/5 regenerate the
   prompt JSONL files from disk state; Stages 3 and 6 hit Spark and
   *will* produce non-identical outputs (LLM non-determinism), so the
   committed responses act as the canonical archive for the deployed
   ThaqalaynWords if you want byte-exact reproduction.

## Cross-references

- Pipeline plan: [`Thaqalayn/docs/WORDS_PROJECT_PLAN.md`](../Thaqalayn/docs/WORDS_PROJECT_PLAN.md)
- Round-by-round results: [`Thaqalayn/docs/PATH_B_SPARK_LOG.md`](../Thaqalayn/docs/PATH_B_SPARK_LOG.md)
- Run / resume runbook: [`Thaqalayn/docs/PATH_B_STATUS.md`](../Thaqalayn/docs/PATH_B_STATUS.md)
- Per-script command reference: [`Thaqalayn/docs/COMMANDS.md`](../Thaqalayn/docs/COMMANDS.md) §3i
