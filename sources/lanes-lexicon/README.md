# Lane's Arabic-English Lexicon

TEI XML edition of Lane's Arabic-English Lexicon (1863–93) — 8 volumes,
3,064 pages, the most comprehensive classical Arabic dictionary in English.

**Source:** <https://github.com/laneslexicon/lexicon_xml>
(Perseus Digital Library digitization with corrections / amendments.)

**License:** Public domain. The original lexicon (1863–93) is long out of
copyright; the Perseus digitization is also released to the public domain.

Per DECISION_LOG D047 we chose Perseus XML over the various JSON forks
because it preserves the original structural markup: source citations
(S=Sihah, K=Kamoos, etc.), cross-references, italic markup. JSON forks
tend to flatten these out.

## Files

| File | Size | Description |
|------|------|-------------|
| `*.xml` (36 files) | ~80 MB | Raw TEI XML, one per alphabet letter chapter (`b0.xml`, `_E0.xml` etc.). Buckwalter-encoded Arabic. |
| `parsed_entries.json` | 31 MB | Flat list of 48,103 entries with id/key/letter/root/orth_ar/raw_text. |
| `root_index.json` | 31 MB | Entries grouped by Arabic root — 5,187 unique. |
| `orth_index.json` | 1 MB | Arabic head-form → entry IDs — 46,924 unique. |

## TEI XML hierarchy

```
<TEI>
  <text><body>
    <div1 type="alphabetical letter" n="b">
      <div2 type="root" n="bAb">              <!-- root: BAb -->
        <entryFree id="..." key="..." type="main">
          <form><orth lang="ar">...</orth></form>
          ...definitions, examples, refs...
        </entryFree>
      </div2>
    </div1>
  </body></text>
</TEI>
```

## Buckwalter encoding note

Perseus stores Arabic in Buckwalter transliteration (e.g., `$` = ش,
`E` = ع, `*` = ذ), plus some Perseus-specific extensions: `^` is used
as a diacritic marker and digits like `1` appear as positional markers
in some entries.

Phase 5 of the Words project will apply CAMeL Tools' `bw2ar` CharMapper
to convert these to UTF-8 Arabic before rendering. Custom mapping
needed for `^` and digit handling.

## Re-build

```
python scripts/download_lanes_lexicon.py            # downloads + parses
python scripts/download_lanes_lexicon.py --force    # force redownload
python scripts/download_lanes_lexicon.py --parse-only  # parse existing XML
```

The script writes outputs to this directory.
