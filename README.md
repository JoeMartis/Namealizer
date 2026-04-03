# Namealizer

A lightweight, zero-dependency, single-file web app for detecting and fixing name formatting issues. Paste a list of names, review flagged problems, fix them inline, and export clean results.

**No server, no API, no build tools.** Just open `index.html` in a browser.

---

## Features

- **12 issue types** detected with confidence levels (High / Medium / Low)
- **~990 given names** and **~810 surnames** across 30+ cultures
- **Inline editing** — click any name to fix it directly
- **Smart suggestions** — auto-fix capitalization, reversed names, titles, spacing
- **Fuzzy duplicate detection** — Levenshtein edit distance catches typos like "Micheal" vs "Michael"
- **Filter by issue type** — focus on one problem category at a time
- **Ignore whitespace toggle** — hide spacing issues when working with CSV data
- **Export** — CSV (3-column) or text file, sorted alphabetically by surname
- **Copy to clipboard** — one click
- **Undo dismissed** — restore entries you dismissed
- **Fully client-side** — no data leaves the browser

---

## Detection Engine

### Issue Types

| Issue | Tag | Description | Default Confidence |
|---|---|---|---|
| **Possibly Reversed** | `reversed` | Name parts appear in wrong order | Medium |
| **Capitalization** | `caps` | ALL CAPS, all lowercase, inverted case, or uncapitalized parts | High |
| **Spacing/Punctuation** | `spacing` | Tabs, extra spaces, leading/trailing whitespace, CSV quotes, trailing commas | High |
| **Data Error** | `data` | Numbers, special characters, placeholders, mojibake, invisible chars, formula injection, concatenated names, repeated parts | High |
| **Unusual Format** | `format` | Mixed ALL-CAPS parts, parenthetical nicknames, S/O D/O W/O patterns | Low |
| **Duplicate** | `duplicate` | Exact match after normalizing case, diacritics, and non-alpha characters | High |
| **Near Duplicate** | `neardupe` | Edit distance within threshold (catches typos, spelling variants) | Medium |
| **Single Initial** | `initial` | Single-letter tokens that may indicate truncation (e.g., "N Ramirez") | Low |
| **Possible Typo** | `spelling` | 5+ consecutive consonants without known valid clusters, or triple-repeated characters | Low |
| **Has Title/Honorific** | `title` | Mr., Mrs., Dr., Prof., Rev., military ranks, etc. | High |
| **Merged Entry** | `merged` | Contains "&" or "and" suggesting two people in one field | High |
| **Possibly Truncated** | `truncated` | Name ends with trailing space or last word is suspiciously short at a field-length boundary | Medium |

### Confidence Levels

Each issue gets a context-sensitive confidence level:

- **High** (red) — Almost certainly an error. Examples: ALL CAPS, numbers in names, tab characters, exact duplicates, placeholder entries like "NULL" or "test", formula injection
- **Medium** (amber) — Likely an issue but may be intentional. Examples: uncapitalized name part ("svetoslav Dzhonev"), reversed name heuristic, near-duplicate, repeated name parts
- **Low** (gray) — Worth reviewing but often valid. Examples: family-name-first order (could be intentional), single initials (could be a legal name), unusual formatting, possible typo

---

## Detection Logic Details

### Reversed Name Detection

Three methods, applied in order:

1. **"Last, First" comma format** — `O'Brien, Mary` → High confidence
2. **English name heuristic** — first token is a known surname AND last token is a known given name AND first token is NOT also a known given name. Example: "Smith John" → Medium confidence
3. **Family-name-first cultures** — first token is in the `FAMILY_FIRST_SURNAMES` set (243 entries covering Chinese, Korean, Vietnamese, Japanese, Hungarian surnames) AND the remaining tokens aren't all also surnames. Example: "Kim Minho", "Cheung Wing-Kei" → Low confidence

### Capitalization Detection

Checks significant parts only (skips particles like "de", "van", "al" and single characters):

- **ALL CAPS** — every significant part is uppercase: `JANE DOE`
- **all lowercase** — every significant part is lowercase: `jane doe`
- **Inverted case** — first letter lowercase, later letters uppercase: `jOHN sMITH`
- **Uncapitalized part** — at least one significant part starts lowercase: `svetoslav Dzhonev`

### Duplicate Detection

1. **Exact duplicates** — O(n) via Map lookup after normalizing: lowercase, strip diacritics (NFD decomposition), European romanization (ø→o, æ→ae, ß→ss, ð→d, þ→th, ł→l), smart quote normalization, invisible character removal
2. **Near duplicates** — Levenshtein edit distance comparison. Threshold: ≤2 edits for names ≤10 chars, or ≤8% of length for longer names. Skipped for lists >500 names (performance guard)

### Data Error Detection

- Numbers anywhere in name
- Special characters: `< > @ # $ % ^ & * = + { } [ ] | \ ~ \``
- Mojibake patterns: `Ã` followed by high-byte Latin character (UTF-8 misread as Latin-1)
- Invisible Unicode: zero-width spaces (U+200B), ZWNJ (U+200C), ZWJ (U+200D), BOM (U+FEFF), soft hyphens (U+00AD)
- Spreadsheet formula injection: name starts with `=`, `+`, `-`, or `@`
- Concatenated names: camelCase pattern like "JohnSmith" (lowercase→uppercase mid-token, token >6 chars)
- Repeated parts: same multi-letter word appears twice in a single entry (e.g., "Fekri M Wa'el M Fekri Zaid")
- Placeholder entries: `NULL`, `N/A`, `test`, `unknown`, `xxx`, `asdf`, `tbd`, `pending`, `temp`, `dummy`, `sample`, `example`

### Smart Title Case (Suggestion Engine)

The suggestion engine applies intelligent capitalization that respects:

- **Particles** — lowercase mid-name: de, del, da, van, von, al, bin, etc. (53 particles)
- **Mc/Mac prefixes** — McDonald, MacArthur (preserves internal cap only if original had it)
- **O' apostrophe names** — O'Brien, O'Connor
- **Sant' names** — Sant'Anna
- **CamelCase prefixes** — DiMasi, DeLuca, LaForge, LeBron (preserves if original had internal cap)
- **Unknown internal caps** — InJoo, SunHee, JiYeon (preserved rather than flattened)
- **Hyphenated names** — each part title-cased: Jean-Pierre, Anna-Lise
- **Suffixes** — Jr., Sr., II, III, IV, etc. (preserved/uppercased)
- **al-/el- prefixes** — Al-Rashid, El-Sayed

### Title/Honorific Detection

Detects and strips (in suggestions): Mr., Mrs., Ms., Miss, Dr., Prof., Sir, Lady, Rev., Father, Mother, Fr., Sr., Sra., Herr, Frau, Senor, Senora, Hon., Judge, Captain, Capt., Sgt., Cpl., Pvt., Lt., Col., Maj., Gen., Adm.

### Suggestion Suppression

Suggestions are NOT generated for entries with:
- **Data errors** — numbers, special characters, etc. need human judgement
- **Merged entries** — can't auto-split "John & Mary Smith"
- **Truncated names** — can't guess what was cut off

---

## Language & Culture Coverage

### Given Names (~990 total)

| Region | Count | Examples |
|---|---|---|
| **English** | ~180 | James, Mary, Christopher, Elizabeth, Brandon, Samantha |
| **Spanish / Portuguese / Latin American** | ~120 | Alejandro, Valentina, Guadalupe, Ximena, Vinicius, Fernanda |
| **Arabic / Middle Eastern** | ~80 | Mohamed, Fatima, Walid, Yasmin, Rami, Noura |
| **Chinese (Mandarin)** | ~75 | Xiaoming, Yuxuan, Haoran, Jiayi, Wenxuan, Anqi |
| **Chinese (Cantonese/HK)** | ~26 | Wing, Wai, Kai, Fai, Kit, Kei, Tak, Lok |
| **South Asian (India, Pakistan, Sri Lanka)** | ~120 | Arjun, Priya, Karthik, Selvam, Gurpreet, Simran |
| **Japanese** | ~40 | Haruto, Sakura, Arata, Mei, Takuya, Nanami |
| **Korean** | ~30 | Minho, Jisoo, Jihoon, Seungmin, Yeji, Nayeon |
| **Vietnamese** | ~28 | Minh, Thanh, Quang, Phuong, Ngoc, Khanh |
| **African** | ~70 | Kwame, Ngozi, Chinwe, Babatunde, Aminata, Djibril |
| **Southeast Asian** | ~35 | Somchai, Siti, Dewi, Aung, Sokha, Bopha |
| **Russian / Eastern European** | ~20 | Sergei, Tatiana, Mikhail, Katarzyna, Jakub |
| **Scandinavian / Greek / Turkish / Balkan** | ~30 | Bjorn, Astrid, Konstantinos, Mehmet, Dragan |
| **French / German** | ~20 | Pierre, Sophie, Philippe, Hans, Monika |

### Surnames (~810 total)

| Region | Count | Examples |
|---|---|---|
| **English** | ~120 | Smith, Johnson, Williams, O'Brien, McDonald |
| **Spanish / Portuguese / Latin American** | ~130 | Gonzalez, Aguilar, Orozco, Silva, Oliveira, Ferreira |
| **Chinese (Mandarin Pinyin)** | ~60 | Wang, Zhang, Liu, Huang, Zhao |
| **Chinese (Cantonese/HK)** | ~36 | Wong, Cheung, Leung, Lau, Kwok, Tam |
| **Chinese (Taiwanese/Wade-Giles)** | ~17 | Tsai, Hsieh, Hsiao, Chiang, Hsu |
| **Chinese (Hokkien/Teochew/SE Asian)** | ~20 | Tan, Lim, Ong, Goh, Chua, Koh |
| **South Asian** | ~130 | Patel, Sharma, Krishnamurthy, Bhattacharya, Gill, Jayawardena |
| **Korean** | ~30 | Kim, Park, Choi, Ryu, Oh, Byun |
| **Japanese** | ~30 | Tanaka, Suzuki, Sato, Abe, Murakami |
| **Vietnamese** | ~22 | Nguyen, Tran, Le, Pham, Huynh |
| **Arabic / Middle Eastern** | ~40 | Al-Masri, Al-Qahtani, Darwish, Khoury, Haddad |
| **African** | ~50 | Diallo, Mensah, Boateng, Okonkwo, Dembele |
| **Russian / Eastern European** | ~15 | Ivanov, Smirnov, Kowalski, Nowak |
| **Scandinavian / Greek / Turkish / Balkan** | ~30 | Johansson, Papadopoulos, Yilmaz, Jovanovic |
| **French / German** | ~20 | Dubois, Moreau, Muller, Schmidt |
| **Filipino** | ~5 | Santos, Aquino, Del Rosario |

### Family-Name-First Surnames (~243 total)

Used for reversed-name detection in cultures where family name comes first:

| Culture | Count | Examples |
|---|---|---|
| **Chinese (Pinyin)** | ~60 | Wang, Li, Zhang, Liu, Chen |
| **Chinese (Cantonese)** | ~36 | Wong, Cheung, Leung, Lau |
| **Chinese (Taiwanese)** | ~17 | Tsai, Hsieh, Hsiao |
| **Chinese (Hokkien)** | ~20 | Tan, Lim, Ong, Goh |
| **Korean** | ~32 | Kim, Park, Lee, Choi, Ryu |
| **Vietnamese** | ~24 | Nguyen, Tran, Le, Pham |
| **Japanese** | ~38 | Tanaka, Suzuki, Sato, Abe |
| **Hungarian** | ~10 | Nagy, Kovacs, Toth, Szabo |

### Particles (53)

Lowercase mid-name connectors recognized by the engine:

`de, del, da, do, dos, das, du, le, la, des, di, dal, della, dei, degli, delle, lo, van, het, ter, ten, von, zu, zum, zur, al, el, bin, ibn, bint, abu, ap, ab, y, e, i, und, of, ben, ould, ait, ag, af, av, na, binti, bte, bt, um, mac, nic, ui, ni`

---

## Duplicate Detection Normalization

Before comparing names for duplicates, the engine normalizes by:

1. Lowercasing
2. NFD Unicode decomposition + stripping combining marks (diacritics)
3. European romanization: `ø→o`, `æ→ae`, `ß→ss`, `ð→d`, `þ→th`, `ł→l`
4. Smart quote normalization: curly/fancy apostrophes → straight
5. Stripping invisible characters: zero-width spaces, BOM, soft hyphens
6. Non-breaking space → regular space
7. Removing all non-alpha characters

This means these pairs are all detected as duplicates:
- "José García" / "Jose Garcia"
- "Müller" / "Mueller" / "Muller"
- "Björk" / "Bjork"
- "O'Brien" / "O'Brien" (smart quote vs straight)
- "Jane  Doe" / "Jane Doe" (extra space)

---

## Export Formats

### Text Export (`cleaned-names.txt`)
Three equal columns, sorted alphabetically by surname, space-padded:

```
Adams, Sarah          Kim, Minho            Smith, John
Chen, Emily           Lee, David            Thompson, Robert
Garcia, Maria         Nguyen, Thi Lan       Williams, James
```

### CSV Export (`cleaned-names.csv`)
Same three-column layout with headers:

```csv
Column 1,Column 2,Column 3
"Adams Sarah","Kim Minho","Smith John"
"Chen Emily","Lee David","Thompson Robert"
```

### Surname Sorting
Sorts by the last significant word, skipping suffixes (Jr., Sr., II, III, etc.):
- "Reginald P Worthington III" sorts under **W** (Worthington)
- "John Smith Jr." sorts under **S** (Smith)

---

## UI Controls

| Control | Description |
|---|---|
| **Analyze Names** | Parse textarea and run all detections |
| **Load Sample** | Load example names demonstrating all issue types |
| **Filter bar** | Filter by specific issue type or show all/issues/clean |
| **Ignore whitespace toggle** | Hide spacing issues (useful for CSV data) |
| **Accept** | Apply the suggested fix for one name |
| **Dismiss** | Remove a flagged name from results (can undo) |
| **Accept All Suggestions** | Batch-apply all suggestions |
| **Undo All** | Restore all dismissed names |
| **Copy to Clipboard** | Copy all active names |
| **Export CSV / Export Text** | Download cleaned names |

---

## Technical Details

- **Single file**: `index.html` (1,289 lines)
- **Zero dependencies**: no frameworks, no build step, no API calls
- **Performance**: exact duplicates O(n) via Map; fuzzy matching O(n²) with Levenshtein, auto-disabled for lists >500 names
- **Privacy**: all processing happens in the browser — no data is transmitted
- **Browser support**: any modern browser (Chrome, Firefox, Safari, Edge)

---

## License

MIT
