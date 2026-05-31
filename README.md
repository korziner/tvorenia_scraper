# tvorenia_scraper

Rust-сниматель страницъ для `http://tvorenia.russportal.ru/` съ сохраненіемъ состоянія и возможностью продолжать скачиваніе.

Онъ скачиваетъ настоящія страницы съ текстомъ, а не одну только ссылочную структуру:

- `raw/*.html` — полный исходный HTML полученной страницы;
- `content_html/*.html` — только тѣло страницы между `<!--Main Content-->` и `<!--EndOf Main Content-->`;
- `markdown/*.md` — удобочитаемый внѣсѣтевой текстъ, обращенный въ подобіе Markdown;
- `state.json` — состояніе обхода: очередь, видѣнные URL, скачанныя страницы, ошибки;
- `manifest.jsonl` — по одной JSON-записи на всякую сохраненную страницу.

## Сборка

```bash
cargo build --release
```

## Запускъ / продолженіе

```bash
cargo run --release -- --out tvorenia_dump --delay-ms 1500
```

Въ журналѣ нынѣ показываются ходъ работы и настоящій образчикъ текста UTF-8 послѣ каждой сохраненной страницы:

```text
progress: 6/1777 frontier (0.34%), queue=1771, seen=1777, failed=0 GET http://tvorenia.russportal.ru/index.php?id=saeculum.vi_x.i_03_0010
  saved: md=markdown/saeculum.vi_x.i_03_0010.md, html=content_html/saeculum.vi_x.i_03_0010.html, raw=raw/saeculum.vi_x.i_03_0010.html, text_chars=16942, discovered=34
  sample UTF-8: # VI-X ВѢКЪ Преп. Іоаннъ Дамаскинъ († ок. 780 г.) Слово объ усопшихъ въ вѣрѣ, — о томъ, какую пользу приносятъ имъ совершаемыя о нихъ литургіи...
  progress: 7/1810 frontier (0.39%), queue=1803, seen=1810, failed=0
```

`progress.json` также пишется рядомъ съ `state.json`, такъ что за ходомъ работы можно слѣдить изъ другого терминала.

Разрядка / письмо съ разставленными буквами автоматически исправляется въ новыхъ `markdown/*.md`. Примѣръ:

```text
П у ш к и н ъ   у ч и л ъ   Р о с с і ю
```

обращается въ:

```text
Пушкинъ училъ Россію
```

Полезныя возможности вывода:

```bash
# печатать ходъ работы послѣ каждыхъ 25 обработанныхъ страницъ, а не послѣ каждой
cargo run --release -- --out tvorenia_dump --progress-every 25

# болѣе длинные / короткіе образчики текста; 0 отключаетъ образчики
cargo run --release -- --out tvorenia_dump --sample-chars 500
cargo run --release -- --out tvorenia_dump --sample-chars 0

# уменьшить IO: писать state.json/progress.json только послѣ каждыхъ 1000 URL
cargo run --release -- --out tvorenia_dump --checkpoint-every 1000

# держать state.json/progress.json въ RAM (/dev/shm), а тексты писать на дискъ
mkdir -p /dev/shm/tvorenia_state
cargo run --release -- --out tvorenia_dump --checkpoint-dir /dev/shm/tvorenia_state --checkpoint-every 1000

# ВАЖНО: если употребляете --checkpoint-dir /dev/shm, передъ выключеніемъ/перезагрузкой
# скопируйте state.json назадъ, иначе точка продолженія пропадетъ:
rsync -a /dev/shm/tvorenia_state/ tvorenia_dump/

# уменьшить IO и объемъ: писать только markdown, безъ raw и content_html
cargo run --release -- --out tvorenia_dump --no-raw-html --no-content-html

# принудительно перекачать уже сохраненныя страницы; полезно послѣ исправленій декодера/разборщика
cargo run --release -- --out azbyka_ru_otechnik_dump --start 'https://azbyka.ru/otechnik' --redownload

# пропускать 403 Forbidden, не занося ихъ въ неудачи; 403 — умолчаніе
cargo run --release -- --out azbyka_ru_otechnik_dump --start 'https://azbyka.ru/otechnik' --skip-http-status 403

# пропускать URL, уже записанныя въ failed въ старомъ state.json
cargo run --release -- --out azbyka_ru_otechnik_dump --start 'https://azbyka.ru/otechnik' --skip-failed

# исключать URL по регулярному выраженію; можно указывать нѣсколько разъ
cargo run --release -- --out tvorenia_dump \
  --exclude-url-regex 'https?://russportal\.ru/news/' \
  --exclude-url-regex '/gb/'
```

Замѣчаніе объ кодировкѣ: страницы декодируются по HTTP/meta `charset`, когда онъ указанъ; иначе сначала пробуется UTF-8, а Windows-1251 служитъ запаснымъ способомъ. Файлы Markdown пишутся какъ UTF-8. Числовыя HTML-сущности, напр. `&#1123;`, въ `markdown/*.md` обращаются въ настоящіе Unicode-знаки, такъ что старыя буквы сохраняются какъ подлинныя: `ѣ`, `Ѣ`, `і`, `І`, `ѳ`, `Ѳ`, `ѵ`. См. `examples/encoding_sample.md`.

Остановить можно когда угодно чрезъ `Ctrl+C`. Потомъ запустите ту же команду снова; программа продолжитъ изъ
`tvorenia_dump/state.json` и пропуститъ уже записанныя на дискъ страницы, дабы не тратить трафикъ на повторное скачиваніе готовыхъ документовъ.

Малый опытъ:

```bash
cargo run --release -- --limit 5 --out tvorenia_test
```

Начать съ отдѣла:

```bash
cargo run --release -- \
  --start 'http://tvorenia.russportal.ru/index.php?id=saeculum.vi_x' \
  --out tvorenia_vi_x
```

Обходчикъ нынѣ беретъ узелъ и начальный путь изъ `--start`, такъ что и другіе сайты / подотдѣлы работаютъ. Для всякаго узла или подотдѣла употребляйте отдѣльную папку `--out`:

```bash
cargo run --release -- \
  --start 'http://lib.russportal.ru/' \
  --out lib_dump

cargo run --release -- \
  --start 'https://azbyka.ru/otechnik' \
  --out azbyka_ru_otechnik_dump
```

Для `https://azbyka.ru/otechnik` обходчикъ остается внутри `/otechnik` и пропускаетъ очевидныя принадлежности сайта: изображенія, CSS, JS, PDF, звукъ/видео, шрифты и архивы.

Позднѣе вновь пробовать неудавшіяся страницы:

```bash
cargo run --release -- --out tvorenia_dump --retry-failed
```

По умолчанію HTTP 403 записывается въ `skipped`, а не въ `failed`, и при послѣдующихъ запускахъ не пробуется вновь. Иные коды можно указать спискомъ:

```bash
cargo run --release -- --out azbyka_ru_otechnik_dump --start 'https://azbyka.ru/otechnik' --skip-http-status 403,404,429
```

Чтобы отключить сіе правило:

```bash
cargo run --release -- --out azbyka_ru_otechnik_dump --start 'https://azbyka.ru/otechnik' --skip-http-status ''
```

## Устраненіе повторовъ съ учетомъ орѳографіи

`huniq` не видитъ, что дореформенное и современное написаніе могутъ быть однимъ и тѣмъ же текстомъ. Вмѣсто него употребляйте `orthodedup`. Онъ строитъ ключъ повтора, приводя старое написаніе къ современно-подобному виду, но при нахожденіи повторовъ **оставляетъ дореформенный вариантъ**.

Замѣна `huniq` въ строчномъ конвейерѣ:

```bash
time bfs -name "*md" -exec rg "ѣ.*ѣ" {} \; \
  | rg -v "Сейчасъ на порталѣ посѣтителей" \
  | cargo run --release --bin orthodedup -- --mode lines \
  | zstd -19 -vT2 > russportal.zst
```

Одинъ разъ собрать и употреблять исполнимый файлъ прямо:

```bash
cargo build --release --bin orthodedup

time bfs -name "*md" -exec rg "ѣ.*ѣ" {} \; \
  | rg -v "Сейчасъ на порталѣ посѣтителей" \
  | ./target/release/orthodedup --mode lines \
  | zstd -19 -vT2 > russportal.zst
```

Устраненіе повторовъ цѣлыхъ файловъ / документовъ:

```bash
find russportal_dump/markdown -name '*.md' -print0 \
  | ./target/release/orthodedup \
      --mode files \
      --file-list0 \
      --pairs duplicate_pairs.tsv \
      --keepers keepers.txt
```

Скопировать оставленныя файлы въ новую папку:

```bash
find russportal_dump/markdown -name '*.md' -print0 \
  | ./target/release/orthodedup \
      --mode files \
      --file-list0 \
      --pairs duplicate_pairs.tsv \
      --copy-kept deduped_markdown
```

Примѣры нормализаціи, употребляемыя для ключей повторовъ:

```text
въ вѣрѣ пришелъ       ~= в вере пришел
духовнаго дѣла        ~= духовного дела
мудрыя змѣи           ~= мудрые змеи
Россія / національный ~= Россия / национальный
```

## Исправленіе уже скачанныхъ Markdown-файловъ

Есть отдѣльный Rust-исправитель для уже имѣющихся файловъ:

```bash
# обнаружить файлы съ разрядкой
cargo run --release --bin derazryadka -- --detect-only tvorenia_dump/markdown/*.md

# вывести исправленный текстъ въ stdout
cargo run --release --bin derazryadka -- tvorenia_dump/markdown/some_file.md

# переписать файлы на мѣстѣ
cargo run --release --bin derazryadka -- -i tvorenia_dump/markdown/*.md
```

Есть также простая запасная программа GNU awk:

```bash
gawk -f scripts/derazryadka.awk old.md > fixed.md
```

Rust-исправитель точнѣе: онъ понимаетъ NBSP, старыя кириллическія Unicode-буквы и сочетанныя ударенія, напр. `и́`.

## Сбереженіе трафика: точки продолженія

По умолчанію программа нынѣ сохраняетъ `state.json` не послѣ каждой страницы, а послѣ каждыхъ 100 обработанныхъ URL (`--checkpoint-every 100`). Это сберегаетъ дискъ: при большихъ обходахъ `state.json` становится великъ, и запись его послѣ каждой страницы даетъ десятки гигабайтъ лишняго IO. Для стараго максимально осторожнаго поведѣнія укажите `--checkpoint-every 1`; для меньшаго IO — `--checkpoint-every 1000` или больше. `state.json` пишется компактнымъ JSON, не pretty-JSON.

Для уменьшенія дисковаго трешинга можно вынести `state.json` и `progress.json` въ RAM:

```bash
mkdir -p /dev/shm/tvorenia_state
cargo run --release -- --out tvorenia_dump --checkpoint-dir /dev/shm/tvorenia_state --checkpoint-every 1000
```

Но `/dev/shm` очищается при перезагрузкѣ; скопируйте состояніе назадъ:

```bash
rsync -a /dev/shm/tvorenia_state/ tvorenia_dump/
```

Страница считается скачанною только послѣ того, какъ нужные файлы записаны атомарно. Если указанъ `--no-raw-html`, не пишется `raw/*.html`; если указанъ `--no-content-html`, не пишется `content_html/*.html`. Если процессъ прервется, слѣдующій запускъ продолжитъ работу изъ ближайшей сохраненной точки.


EN
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
