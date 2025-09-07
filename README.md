# URI Fragment Identifiers for Spreadsheet formats

This document describes a proposal for identifying fragments of spreadsheet documents (e.g. `.xlsx`, `.ods`, `.gsheet`). It is inspired by [RFC 7111](https://datatracker.ietf.org/doc/html/rfc7111) (fragment selectors for `text/csv`), but is scoped and adapted for richer spreadsheet formats. The goal is to provide a consistent way for tools to reference specific sheets, ranges, or cells inside a spreadsheet resource.

---

## 1. Introduction

### 1.1 Background

Spreadsheets are widely used for structured data. Unlike CSV, they support multiple sheets, named ranges, and richer referencing. A consistent fragment selector syntax makes it easier for tools and APIs to link to specific parts of spreadsheets.

Spreadsheets usually consist of multiple sheets (or tabs), each containing a grid of cells. Users can also define named ranges and structured tables.
Cell addresses usually use A1 notation as defined in [Excel](https://learn.microsoft.com/en-us/office/vba/excel/concepts/cells-and-ranges/refer-to-cells-and-ranges-by-using-a1-notation), [Google sheets](https://developers.google.com/workspace/sheets/api/guides/concepts?hl=de#a1_notation) or [OpenDocument](https://docs.oasis-open.org/office/OpenDocument/v1.3/cs01/part3-schema/OpenDocument-v1.3-cs01-part3-schema.html#__RefHeading__1415614_253892949). This means columns are labeled A, B, C...Z, AA, AB... and rows are numbered 1, 2, 3... Cells are referenced by their column letter followed by their row number (e.g. `B3`).

### 1.2 Goals

- Enable programmatic linking to **sheets**, **ranges**, **cells**, and **named areas**.  
- Keep selectors simple, human-readable, and close to how users already reference cells in spreadsheet applications.  
- Define predictable fallback behavior when a selector cannot be resolved.  

### 1.3 Scope

- Applies to spreadsheet-like formats: `.xlsx`, `.xls`, `.ods`, `.gsheet`, and `.numbers`.  
- Does not cover rendering concerns (e.g. charts, formatting). Focus is on data references.  
- Intended as a **working draft** for use in open source tooling.

---

## 2. Fragment Selector Types

### 2.1 Cell

Selects individual cells by their A1 notation. Multiple cells can be comma-separated.

Examples:

```url
http://example.com/data.xlsx#cell=B3
http://example.com/data.xlsx#cell=A1,C5,E10
```

### 2.1 Sheet

Selects a whole sheet by name.
Example:  

```url
http://example.com/data.xlsx#sheet=Sheet1
```

### 2.2 Range

Selects a rectangular range of cells using A1 notation.

Examples:  

```url
http://example.com/data.xlsx#range=A1:D10
http://example.com/data.xlsx#sheet=Sheet2&range=B2:C5
```

### 2.4 Named Ranges

Uses spreadsheet-defined names.  
Example:  

```url
http://example.com/data.xlsx#named=SalesData

```

### 2.5 Tables (Excel-style)

References structured tables.

Example:

```url
http://example.com/data.xlsx#table=Expenses

```

### 2.6 Multi-Selections

Multiple selectors may be combined.

Example:

```url
http://example.com/data.xlsx#sheet=Sheet1&cell=A1&range=B2:C3
```

---

## 3. Syntax

Selectors follow a `key=value` format joined by `&`.  

Proposed grammar (simplified):

```text
fragment            = selector *( "&" selector )
selector            = sheet-selector / range-selector / cell-selector /
                      named-selector / table-selector

sheet-selector      = "sheet=" value
range-selector      = "range=" cell-ref ":" cell-ref
cell-selector       = "cell=" cell-list
named-selector      = "named=" value
table-selector      = "table=" value

cell-list           = cell-ref *( "," cell-ref )
cell-ref            = column-label row-number  ; e.g. A1, B2, AA10
column-label        = 1*( %x41-5A )            ; A..Z, AA..ZZ, etc.
row-number          = 1*DIGIT

value               = 1*( pchar-no-amp-hash )
pchar-no-amp-hash   = unreserved / pct-encoded / sub-delims-no-amp / ":" / "@"
sub-delims-no-amp   = "!" / "$" / "'" / "(" / ")" / "*" / "+" / "," / ";" / "=" 
                    ; no "&" or "#"
pct-encoded         = "%" HEXDIG HEXDIG
unreserved          = ALPHA / DIGIT / "-" / "." / "_" / "~"
HEXDIG              = DIGIT / "A" / "B" / "C" / "D" / "E" / "F"
ALPHA               = %x41-5A / %x61-7A
DIGIT               = %x30-39
```

### Notes for implementers

- Disallowed literally in values: & and #. If needed in a name, encode them (& → %26, # → %23).

- Everything else allowed by pchar is fine, but if your name includes characters outside pchar (e.g., space), percent-encode ( → %20).

- Percent-decoding is case-insensitive (%2F ≡ %2f).

- This spec intentionally does not enforce application-specific limits (e.g., Excel’s 31-char sheet name limit or forbidden characters). Different formats vary. Tools may validate those separately.

### Examples

- Sheet with space:

    ```url
    http://example.com/data.xlsx#sheet=Annual%20Report
    ```

- Sheet with & in the name (R&D):

    ```url
    http://example.com/data.xlsx#sheet=R%26D
    ```

- Named range with # in the name (Sales#2025):

    ```url
    http://example.com/data.xlsx#named=Sales%232025
    ```

---

## 4. Processing Rules

### 4.1 Resolution

- If `sheet` is specified, selectors apply to that sheet.
- If no sheet is specified, default to the first sheet.
- Named ranges and tables are resolved first, then ranges and cells.

### 4.2 Errors

- Unknown sheet, named range, or table → treat as no match.  
- Malformed fragment syntax → ignore fragment (use full document).  

### 4.3 Fallback

- Tools that do not understand fragments should process the entire spreadsheet.  
- Tools may choose to return partial data when only part of the selector resolves.  

---

## 5. Security and Privacy

- Linking to specific cells may expose sensitive information if documents are shared externally.
- Tools should consider redacting or refusing to resolve selectors that point to hidden or protected areas.

---

## 6. Future Extensions

- Chart or pivot table references  
- Row/column header references by label  
- Rich query selectors (`filter`, `sort`)  

---

## 7. Status

This document is an **internal draft** for open source tooling. It is not a formal standard, but may form the basis of future proposals if community adoption grows.
