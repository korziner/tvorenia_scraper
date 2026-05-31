# tvorenia_scraper

Rust checkpointing scraper for `http://tvorenia.russportal.ru/`.

It downloads actual document pages, not only the link structure:

- `raw/*.html` — full fetched page HTML
- `content_html/*.html` — only the page body between `<!--Main Content-->` and `<!--EndOf Main Content-->`
- `markdown/*.md` — readable offline text/Markdown-ish conversion
- `state.json` — checkpoint with queue, seen URLs, downloaded URLs, failures
- `manifest.jsonl` — one JSON record per saved page

## Build

```bash
cargo build --release
```

## Run / resume

```bash
cargo run --release -- --out tvorenia_dump --delay-ms 1500
```

The log now includes progress and a real UTF-8 text sample after each saved page:

```text
progress: 6/1777 frontier (0.34%), queue=1771, seen=1777, failed=0 GET http://tvorenia.russportal.ru/index.php?id=saeculum.vi_x.i_03_0010
  saved: md=markdown/saeculum.vi_x.i_03_0010.md, html=content_html/saeculum.vi_x.i_03_0010.html, raw=raw/saeculum.vi_x.i_03_0010.html, text_chars=16942, discovered=34
  sample UTF-8: # VI-X ВѢКЪ Преп. Іоаннъ Дамаскинъ († ок. 780 г.) Слово объ усопшихъ въ вѣрѣ, — о томъ, какую пользу приносятъ имъ совершаемыя о нихъ литургіи...
  progress: 7/1810 frontier (0.39%), queue=1803, seen=1810, failed=0
```

`progress.json` is also written next to `state.json`, so another terminal can watch it.

Razryadka / letter-spaced emphasis is fixed automatically in new `markdown/*.md` output. Example:

```text
П у ш к и н ъ   у ч и л ъ   Р о с с і ю
```

becomes:

```text
Пушкинъ училъ Россію
```

Useful display options:

```bash
# print progress every 25 processed pages instead of every page
cargo run --release -- --out tvorenia_dump --progress-every 25

# longer/shorter text examples; 0 disables samples
cargo run --release -- --out tvorenia_dump --sample-chars 500
cargo run --release -- --out tvorenia_dump --sample-chars 0
```

Encoding note: source pages are decoded as Windows-1251 and the Markdown files are written as UTF-8. HTML numeric entities such as `&#1123;` are decoded in `markdown/*.md`, so old characters are real Unicode, for example `ѣ`, `Ѣ`, `і`, `І`, `ѳ`, `Ѳ`, `ѵ`. See `examples/encoding_sample.md`.

Stop with `Ctrl+C` whenever you want. Run the same command again; it resumes from
`tvorenia_dump/state.json` and skips pages already written to disk, so it does not
waste traffic re-downloading completed documents.

For a small test:

```bash
cargo run --release -- --limit 5 --out tvorenia_test
```

Start from a subsection:

```bash
cargo run --release -- \
  --start 'http://tvorenia.russportal.ru/index.php?id=saeculum.vi_x' \
  --out tvorenia_vi_x
```

The crawler now uses the host from `--start`, so other RussPortal sub-sites work too. Use a separate `--out` directory per host:

```bash
cargo run --release -- \
  --start 'http://lib.russportal.ru/' \
  --out lib_dump
```

Retry failures later:

```bash
cargo run --release -- --out tvorenia_dump --retry-failed
```

## Fix already downloaded Markdown files

A separate Rust fixer is included for existing files:

```bash
# detect files with razryadka
cargo run --release --bin derazryadka -- --detect-only tvorenia_dump/markdown/*.md

# print fixed text to stdout
cargo run --release --bin derazryadka -- tvorenia_dump/markdown/some_file.md

# rewrite files in place
cargo run --release --bin derazryadka -- -i tvorenia_dump/markdown/*.md
```

There is also a simple GNU awk fallback:

```bash
gawk -f scripts/derazryadka.awk old.md > fixed.md
```

The Rust fixer is more accurate: it understands NBSP, old Cyrillic Unicode letters, and combining accents such as `и́`.

## Traffic-saving checkpoints

The program saves `state.json` after every page and every failure. A page is marked
as downloaded only after all three files (`raw`, `content_html`, `markdown`) are
written atomically. If the process dies, the next run continues from the queue.
