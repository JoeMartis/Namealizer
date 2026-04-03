# Namealyzer

**The proper name anomaly finder**

A lightweight, zero-dependency, single-file web app for validating name data. Drop a CSV with `F_Name`, `L_Name`, and `user_full_name` columns, review flagged issues, fix them inline, and export clean results.

**No server, no API, no build tools.** Just open `index.html` in a browser.

---

## How It Works

1. **Drop or paste** a CSV file with three columns: `F_Name`, `L_Name`, `user_full_name`
2. Click **Analyze Names** — header rows are auto-detected and skipped
3. **Review** flagged issues in the results table with confidence scores (1-10)
4. **Fix** names inline, accept suggestions, or dismiss false positives
5. **Export** cleaned names as CSV or text (3-column layout, sorted by surname)

---

## Input Format

CSV with three fields (comma or tab separated):

```
F_Name,L_Name,user_full_name
John,Smith,John Smith
Jane,Doe,Jane Marie Doe
```

- Header row auto-detected and skipped
- Supports quoted fields: `"Smith, Jr.",John,"John Smith, Jr."`
- Handles escaped quotes: `"Smith""Jr"` → `Smith"Jr`
- Drag-and-drop `.csv`, `.txt`, or `.tsv` files onto the textarea
- **Load File** button as alternative to drag-and-drop

---

## Detection Engine

### Cross-Field Validation (CSV-specific)

Compares F_Name and L_Name against user_full_name per record:

| Issue | Description | Score |
|---|---|---|
| **F/L Names Swapped** | F_Name looks like a surname, L_Name looks like a given name | 9 |
| **Name Part Missing** | First or last name not found in the full name | 8 |
| **Fields Mismatch** | Full name doesn't contain either first or last name | 8 |
| **Extra in Full Name** | Full name has parts not in F_Name/L_Name (middle names, suffixes) | 2 |

### Per-Name Detection

Runs on the `user_full_name` field:

| Issue | Description | Score |
|---|---|---|
| **Possibly Reversed** | Name parts in wrong order (heuristic or "Last, First" format) | 3-8 |
| **Capitalization** | ALL CAPS, all lowercase, inverted case, uncapitalized parts | 6-9 |
| **Data Error** | Numbers, special characters, placeholders, mojibake, invisible chars, formula injection, concatenated names, repeated parts | 8-10 |
| **Unusual Format** | Mixed ALL-CAPS parts, parenthetical nicknames, S/O D/O W/O patterns | 2 |
| **Duplicate** | Exact match after normalizing case, diacritics, and non-alpha characters | 9 |
| **Near Duplicate** | Edit distance within threshold (catches typos like "Micheal"/"Michael") | 5 |
| **Single Initial** | Single-letter tokens that may indicate truncation | 3 |
| **Possible Typo** | 5+ consecutive consonants or triple-repeated characters | 3 |
| **Has Title/Honorific** | Dr., Mr., Mrs., Prof., Rev., military ranks (requires period) | 8 |
| **Merged Entry** | Contains "&" or "and" suggesting two people in one field | 9 |
| **Possibly Truncated** | Raw input has trailing whitespace suggesting field-length cutoff | 5 |

### List-Context Analysis

Uses the full list to find issues that can't be detected per-name:

| Issue | Description | Score |
|---|---|---|
| **Format Outlier** | Uses "Last, First" when >80% of list uses "First Last" (or vice versa) | 5 |
| **Swapped (confirmed)** | Both "John Smith" and "Smith John" exist in the same list | 9 |

### Confidence Scores (1-10)

Each row shows the highest score across all its issues:

- **8-10** (red) — Almost certainly an error
- **5-7** (amber) — Likely issue, worth reviewing
- **1-4** (gray) — May be valid, review recommended

---

## Reversed Name Detection

Three methods, applied in order:

1. **"Last, First" comma format** — `O'Brien, Mary` → score 8
2. **English name heuristic** — first token is a known surname AND last token is a known given name → score 5
3. **Family-name-first cultures** — first token is in the `FAMILY_FIRST_SURNAMES` set (243 entries: Chinese, Korean, Vietnamese, Japanese, Hungarian) → score 3

---

## Smart Title Case (Suggestion Engine)

When suggesting fixes, the engine respects:

- **Particles** — lowercase mid-name: de, del, da, van, von, al, bin, etc. (53 particles)
- **Mc/Mac prefixes** — McDonald, MacArthur
- **O' apostrophe names** — O'Brien, O'Connor
- **CamelCase prefixes** — DiMasi, DeLuca, LaForge, LeBron
- **Unknown internal caps** — InJoo, SunHee, JiYeon (preserved)
- **Hyphenated names** — Jean-Pierre, Anna-Lise
- **Suffixes** — Jr., Sr., II, III, IV
- **al-/el- prefixes** — Al-Rashid, El-Sayed

### Suggestion Suppression

Suggestions are NOT generated for:
- Data errors (numbers, special characters)
- Merged entries (can't auto-split)
- Truncated names (can't guess what was cut off)

---

## Language & Culture Coverage

### Given Names (~990 total)

| Region | Count | Examples |
|---|---|---|
| English | ~180 | James, Mary, Christopher, Elizabeth |
| Spanish / Portuguese / Latin American | ~120 | Alejandro, Valentina, Guadalupe, Vinicius |
| Arabic / Middle Eastern | ~80 | Mohamed, Fatima, Walid, Yasmin |
| Chinese (Mandarin + Cantonese) | ~100 | Yuxuan, Haoran, Wing, Wai, Kai |
| South Asian (India, Pakistan, Sri Lanka) | ~120 | Arjun, Priya, Karthik, Selvam, Gurpreet |
| Japanese | ~40 | Haruto, Sakura, Arata, Mei |
| Korean | ~30 | Minho, Jisoo, Jihoon, Seungmin |
| Vietnamese | ~28 | Minh, Thanh, Quang, Phuong |
| African | ~70 | Kwame, Ngozi, Chinwe, Babatunde, Aminata |
| Southeast Asian | ~35 | Somchai, Siti, Dewi, Aung, Sokha |
| Scandinavian / Greek / Turkish / Balkan | ~30 | Bjorn, Astrid, Konstantinos, Mehmet |
| Russian / Eastern European | ~20 | Sergei, Tatiana, Mikhail, Katarzyna |
| French / German | ~20 | Pierre, Sophie, Philippe, Hans |

### Surnames (~810 total)

| Region | Count | Examples |
|---|---|---|
| English | ~120 | Smith, Johnson, Williams, O'Brien |
| Spanish / Portuguese / Latin American | ~130 | Gonzalez, Aguilar, Silva, Oliveira |
| Chinese — Mandarin Pinyin | ~60 | Wang, Zhang, Liu, Huang |
| Chinese — Cantonese/HK | ~36 | Wong, Cheung, Leung, Lau, Kwok |
| Chinese — Taiwanese (Wade-Giles) | ~17 | Tsai, Hsieh, Hsiao, Chiang |
| Chinese — Hokkien/SE Asian | ~20 | Tan, Lim, Ong, Goh, Chua |
| South Asian | ~130 | Patel, Sharma, Krishnamurthy, Bhattacharya, Gill |
| Korean | ~30 | Kim, Park, Choi, Ryu, Oh |
| Japanese | ~30 | Tanaka, Suzuki, Sato, Abe, Murakami |
| Vietnamese | ~22 | Nguyen, Tran, Le, Pham |
| Arabic / Middle Eastern | ~40 | Al-Masri, Al-Qahtani, Darwish, Khoury |
| African | ~50 | Diallo, Mensah, Boateng, Okonkwo |
| Scandinavian / Greek / Turkish / Balkan | ~30 | Johansson, Papadopoulos, Yilmaz |
| Russian / Eastern European | ~15 | Ivanov, Smirnov, Kowalski |
| French / German | ~20 | Dubois, Moreau, Muller, Schmidt |
| Filipino | ~5 | Santos, Aquino, Del Rosario |

### Family-Name-First Surnames (243 total)

For reversed-name detection in cultures where family name comes first:

| Culture | Count |
|---|---|
| Chinese (Pinyin + Cantonese + Taiwanese + Hokkien) | ~133 |
| Korean | ~32 |
| Vietnamese | ~24 |
| Japanese | ~38 |
| Hungarian | ~10 |

### Particles (53)

Lowercase mid-name connectors:

`de, del, da, do, dos, das, du, le, la, des, di, dal, della, dei, degli, delle, lo, van, het, ter, ten, von, zu, zum, zur, al, el, bin, ibn, bint, abu, ap, ab, y, e, i, und, of, ben, ould, ait, ag, af, av, na, binti, bte, bt, um, mac, nic, ui, ni`

---

## Duplicate Detection Normalizer

Before comparing names, the engine normalizes by:

1. Lowercasing
2. NFD Unicode decomposition + stripping combining marks (diacritics)
3. European romanization: `ø→o`, `æ→ae`, `ß→ss`, `ð→d`, `þ→th`, `ł→l`
4. Smart quote normalization (curly → straight apostrophes)
5. Stripping invisible characters (zero-width spaces, BOM, soft hyphens)
6. Removing all non-alpha characters

Detected as duplicates: "José García" / "Jose Garcia", "Müller" / "Mueller", "O'Brien" / "O'Brien" (smart quotes)

---

## Data Error Detection

- Numbers in names
- Special characters: `< > @ # $ % ^ & * = + { } [ ] | \ ~ \``
- Mojibake (UTF-8 misread as Latin-1): `JosÃ©` for `José`
- Invisible Unicode: zero-width spaces, BOM markers, soft hyphens
- Spreadsheet formula injection: names starting with `= + - @`
- Concatenated names: `JohnSmith` (camelCase mid-token)
- Repeated name parts: `Fekri M Wa'el M Fekri Zaid`
- Placeholder entries: `NULL`, `N/A`, `test`, `unknown`, `xxx`, etc.

---

## Export Formats

### Text Export (`cleaned-names.txt`)

Three equal columns, sorted alphabetically by surname:

```
Adams Sarah          Kim Minho            Smith John
Chen Emily           Lee David            Thompson Robert
Garcia Maria         Nguyen Thi Lan       Williams James
```

### CSV Export (`cleaned-names.csv`)

Same three-column layout, CSV-formatted.

### Surname Sorting

Sorts by the last significant word, skipping suffixes (Jr., Sr., II, III):
- "Reginald P Worthington III" → sorts under **W**
- "John Smith Jr." → sorts under **S**

---

## UI

| Control | Description |
|---|---|
| **Analyze Names** | Parse CSV and run all detections |
| **Load File** | Browse for a .csv/.txt/.tsv file |
| **Drag & Drop** | Drop a file onto the textarea |
| **Load Sample** | Load example CSV with various issue types |
| **Filter bar** | All / Issues / Clean, then per-issue-type filters with counts |
| **Inline edit** | Click any name to edit directly |
| **Accept** | Apply suggested fix for one name |
| **Dismiss** | Remove flagged name from results (undo available) |
| **Accept All Suggestions** | Batch-apply all suggestions (sticky bottom bar) |
| **Copy to Clipboard** | Copy all active names |
| **Export CSV / Export Text** | Download cleaned names |

---

## Technical Details

- **Single file**: `index.html` (~1,480 lines)
- **Zero dependencies**: no frameworks, no build step, no API calls
- **Performance**: exact duplicates O(n) via Map; fuzzy matching auto-disabled for lists >300 names
- **Privacy**: all processing happens in the browser — no data is transmitted
- **Browser support**: any modern browser (Chrome, Firefox, Safari, Edge)

---

## License

MIT
