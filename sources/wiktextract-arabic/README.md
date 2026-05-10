# Wiktextract Arabic

Lemma-indexed extract of Wiktionary's Arabic entries, derived from
the Wiktextract (Tatu Ylönen) postprocessed dump at
<https://kaikki.org/dictionary/Arabic/kaikki.org-dictionary-Arabic.jsonl>.

**License:** Wiktionary content is CC-BY-SA 4.0. Attribution to
Wiktionary contributors required for any redistribution.

## Files

- `summary_index.json` — committed. Maps word → {entry_count, pos_tags, has_etymology, sense_count}. Used to cheaply check whether a Wiktextract entry exists for a given lemma + which POS variants are available.
- (NOT committed) `wiktextract_arabic_lemmas.json` — full slimmed entries with definitions/senses/etymology. Lives in `ThaqalaynDataGenerator/tmp/wiktextract_cache/`. ~221 MB, too large for git. Re-derive with the download script.
- (NOT committed) `kaikki.org-dictionary-Arabic.jsonl` — raw Wiktextract dump in the same cache dir. ~499 MB.

## Re-build

```
python scripts/download_wiktextract_arabic.py
```

After Phase 5 produces our corpus lemma list, a filtered
`wiktextract_corpus_lemmas.json` (small enough to commit) will be
added by a filter step.

**Total entries:** 57,912
