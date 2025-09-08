# URI Fragment Identifiers for Spreadsheet formats

This document describes a proposal for identifying fragments of spreadsheet documents (e.g. `.xlsx`, `.ods`, `.gsheet`). It is inspired by [RFC 7111](https://datatracker.ietf.org/doc/html/rfc7111) (fragment selectors for `text/csv`), but is scoped and adapted for richer spreadsheet formats. The goal is to provide a consistent way for tools to reference specific sheets, ranges, or cells inside a spreadsheet resource.

---

## 1. Introduction

### 1.1 Background

Spreadsheets are widely used for structured data. Unlike CSV, they support multiple sheets, named ranges, and richer referencing. A consistent fragment selector syntax makes it easier for tools and APIs to link to specific parts of spreadsheets.

Spreadsheets usually consist of multiple sheets (or tabs), each containing a grid of cells. Users can also define named ranges and structured tables.
Cell addresses usually use A1 notation as defined in [Excel](https://learn.microsoft.com/en-us/office/vba/excel/concepts/cells-and-ranges/refer-to-cells-and-ranges-by-using-a1-notation), [Google sheets](https://developers.google.com/workspace/sheets/api/guides/concepts?hl=de#a1_notation) or [OpenDocument](https://docs.oasis-open.org/office/OpenDocument/v1.3/cs01/part3-schema/OpenDocument-v1.3-cs01-part3-schema.html#__RefHeading__1415614_253892949). This means columns are labeled A, B, C...Z, AA, AB... and rows are numbered 1, 2, 3... Cells are referenced by their column letter followed by their row number (e.g. `B3`).

### 1.2 Goals

- Enable programmatic linking to **sheets**, **tables**, **named areas**, and **cells/ranges/columns/rows within them**.  
- Keep selectors simple, human-readable, and close to how users already reference cells in spreadsheet applications.  
- Define predictable fallback behavior when a selector cannot be resolved.  

### 1.3 Scope

- Applies to spreadsheet-like formats: `.xlsx`, `.xls`, `.ods`, `.gsheet`, and `.numbers`.  
- Does not cover rendering concerns (e.g. charts, formatting). Focus is on data references.  
- Intended as a **working draft** for use in open source tooling.

---

## 2. Fragment Selector Types

This spec uses a **two-level model**:

- **First-level (target) selectors:** identify a *container* of cells.
  - `sheet=<name>` — a worksheet by name.  
  - `table=<name>` — a structured table by name.  
  - `named=<name>` — a named range (resolves to a rectangular block).  
  - *(If omitted, the default target is the first sheet in document order.)*

- **Second-level (within-target) selectors:** select cells inside the target.
  - `range=<A1>:<A1>` — rectangular block in A1 notation.  
  - `cell=<A1>[,<A1>...]` — one or more individual cells.  
  - `column=<Col>[ : <Col> ]` — one column or a contiguous column range.  
  - `row=<Row>[ : <Row> ]` — one row or a contiguous row range.  

Composition rule: **at most one `&`** may be used to attach **one** second-level selector to **one** first-level selector (or the implicit default sheet).

### 2.1 Sheet (first-level)

Selects a whole sheet by name.

```url
http://example.com/data.xlsx#sheet=Sheet1
````

### 2.2 Table (first-level)

References a structured table by name.

```url
http://example.com/data.xlsx#table=Expenses
```

### 2.3 Named Range (first-level)

Uses a spreadsheet-defined named range.

```url
http://example.com/data.xlsx#named=SalesData
```

### 2.4 Range (second-level)

Selects a rectangular region of cells using A1 notation **within the target**.

```url
http://example.com/data.xlsx#sheet=Sheet2&range=B2:C5
http://example.com/data.xlsx#range=A1:D10
```

### 2.5 Cell (second-level)

Selects one or more individual cells **within the target**.

```url
http://example.com/data.xlsx#cell=B3
http://example.com/data.xlsx#sheet=Sheet1&cell=A1,C5,E10
```

### 2.6 Column (second-level)

Selects **all non-empty cells** in a column or contiguous column range **within the target**.

```url
http://example.com/data.xlsx#column=C
http://example.com/data.xlsx#sheet=Data&column=C:D
```

### 2.7 Row (second-level)

Selects **all non-empty cells** in a row or contiguous row range **within the target**.

```url
http://example.com/data.xlsx#row=3
http://example.com/data.xlsx#named=Region2025&row=2:5
```

> **Non-empty** means cells that contain a stored value or a formula whose evaluated result is not an empty value. Implementations MAY define empty-value handling more precisely per platform.

> **No multi-`&` chaining.** `#sheet=Sheet1&range=A1:B2` is valid; `#sheet=Sheet1&range=A1:B2&row=3` is **invalid**.

---

## 3. Syntax

Selectors follow a `key=value` format. **At most one `&`** is permitted, to attach a **single second-level selector** to a **single first-level selector**. If no first-level selector is given, the second-level selector applies to the default (first) sheet.

### 3.1 ABNF

```text
; Top-level
fragment            = target [ "&" within ]

; First-level targets (choose at most one)
target              = sheet-selector / table-selector / named-selector / within
                      ; If target is omitted, 'within' applies to the default sheet.

sheet-selector      = "sheet=" value
table-selector      = "table=" value
named-selector      = "named=" value

; Second-level selections
within              = range-selector / cell-selector / column-selector / row-selector

range-selector      = "range=" cell-ref ":" cell-ref
cell-selector       = "cell=" cell-list
column-selector     = "column=" column-label [ ":" column-label ]
row-selector        = "row=" row-number [ ":" row-number ]

cell-list           = cell-ref *( "," cell-ref )
cell-ref            = column-label row-number
column-label        = 1*( %x41-5A )            ; A..Z, AA..ZZ, etc.
row-number          = 1*DIGIT

; Names and encoding
value               = 1*( pchar-no-amp-hash )
pchar-no-amp-hash   = unreserved / pct-encoded / sub-delims-no-amp / ":" / "@"
sub-delims-no-amp   = "!" / "$" / "'" / "(" / ")" / "*" / "+" / "," / ";" / "="
pct-encoded         = "%" HEXDIG HEXDIG
unreserved          = ALPHA / DIGIT / "-" / "." / "_" / "~"
HEXDIG              = DIGIT / "A" / "B" / "C" / "D" / "E" / "F"
ALPHA               = %x41-5A / %x61-7A
DIGIT               = %x30-39
```

### Notes for implementers

* Disallowed literally in values: `&` and `#`. If needed in a name, encode them (`&` → `%26`, `#` → `%23`).
* Everything else allowed by `pchar` is fine, but if your name includes characters outside `pchar` (e.g., space), percent-encode (` ` → `%20`).
* Percent-decoding is case-insensitive (`%2F` ≡ `%2f`).
* This spec intentionally does not enforce application-specific limits (e.g., Excel’s 31-char sheet name limit or forbidden characters). Different formats vary. Tools may validate those separately.

---

## 4. Processing Rules

### 4.1 Resolution

* If `sheet`, `table`, or `named` is present, resolve that **target** first.
* If no target is present, the **default sheet** is the target.
* If a `within` selector is present via `&`, apply it inside the resolved target.
* `column` and `row` select **all non-empty cells** in the specified column(s)/row(s) within the target’s bounds.

### 4.2 Errors

* More than one `&` → invalid fragment (treat as no match or ignore fragment).
* Multiple first-level targets (e.g., `sheet=…&table=…`) → invalid.
* Unknown target name or out-of-bounds `column`/`row` → no match.
* Malformed syntax → ignore fragment (use full document).

### 4.3 Fallback

* Tools that do not understand fragments should process the entire spreadsheet.
* Tools MAY return partial data when only part of the selector resolves.

---

## 5. Security and Privacy

* Linking to specific cells may expose sensitive information if documents are shared externally.
* Tools should consider redacting or refusing to resolve selectors that point to hidden or protected areas.

---

## 6. Future Extensions

* Header-based selection (e.g., `column-by-header=Total`)
* Label-based row selection
* Rich query selectors (`filter`, `sort`)
* Explicit empty-value semantics per platform

---

## 7. Status

This document is an **internal draft** for open source tooling. It is not a formal standard, but may form the basis of future proposals if community adoption grows.