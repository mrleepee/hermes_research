# MarkLogic Universal Index: Patent Research

**Patents Referenced:**
- [US8935267B2](https://patents.google.com/patent/US8935267B2/en) — Apparatus and method for executing different query language queries on tree structured data using pre-computed indices of selective document paths
- [US7962474B2](https://patents.google.com/patent/US7962474B2/en) — Parent-child query indexing for XML databases

**Priority Date:** 2002-06-13 (US7962474), 2012-06-19 (US8935267)  
**Assignee:** MarkLogic Corporation

---

## What is the "Universal Index"?

MarkLogic's Universal Index refers to its ability to index **everything** in a document — structure, values, and relationships — and then query that indexed data using **multiple query languages** from a single unified index. Unlike traditional databases where SQL indexes only what you tell it to index, MarkLogic automatically indexes all document paths and values, making any part of the document searchable without pre-declaring schemas or indexes.

The key patents describe two complementary mechanisms:

1. **US7962474B2** — Subtree-based storage and parent-child indexing of XML trees
2. **US8935267B2** — Pre-computed selective indices that support XQuery, XPath, XSLT, Full-text, Geospatial, and SQL simultaneously

---

## Core Architecture: Two-Index Approach

MarkLogic stores documents as **trees** and maintains **two complementary index structures**:

### 1. Top-Down Trees (Structure Index)

Each document is converted into a top-down tree that captures the full document hierarchy:

```
Document (root)
  └── <citation>                ← tag node
        └── <journal>           ← child tag
              └── "Nature"      ← text node (leaf data)
        └── <year>
              └── "2024"
        └── <author>
              └── "Smith"
```

**What it captures:**
- Parent-child relationships (tag → tag, tag → text)
- Ancestor-descendant relationships
- Sibling relationships
- Attribute edges (tag → @attr → value)

**Why it matters:** The tree structure supports hierarchical path queries (XPath, XQuery) natively, without scanning documents.

### 2. Pre-Computed Selective Indices (Content Index)

While building trees, MarkLogic also builds **pre-computed indices** that summarize document content:

> "The pre-computed indices represent the structure of ingested documents. Thus, re-ordering of data to form tables is not performed." — US8935267B2

**What they index:**
- Every unique path in every document
- Value ranges for each path
- Statistics about path occurrences
- Relationships between paths

**The key innovation:** These indices are **language-neutral** — they can be queried by XQuery, XPath, SQL, full-text, or geospatial operators without re-indexing.

---

## US7962474B2: Subtree Storage and Parent-Child Indexing

### The Problem Being Solved

XML documents are hierarchical, but earlier XML databases had to scan entire documents or build slow parent-child joins. This patent describes how MarkLogic decomposes large XML trees into **indexed subtrees** for efficient querying.

### Tree Representation

Every XML document becomes a directed acyclic graph (DAG):

```
    <citation> (root)
        ├── <journal> ── "Nature"
        ├── <year> ── "2024"
        ├── <author> ── "Smith"
        │     └── <affiliation> ── "MIT"
        └── <title>
              └── "A Study of Something"
```

**Node types:**
- **Tag nodes** — XML element names, delimited with `< >`
- **Data nodes** — Text content (leaf nodes)
- **Attribute nodes** — `@name="value"` pairs, marked with `@`

**Edge types:**
- **Parent-child edges** — tag → child tag
- **Tag-data edges** — tag → text content
- **Attribute edges** — tag → @attr → value (special "attribute edge")

### Subtree Decomposition

Large documents are decomposed into **subtrees** for indexing efficiency:

> "Each subtree border defines a subtree; each subtree has a subtree root node and zero or more descendant nodes." — US7962474B2

```
Original tree:
    [root]
       │
    <citation>
       │
    ├──<journal>
    │    └── "Nature"
    ├──<year>
    │    └── "2024"
    └──<author>        ← decomposition boundary (at "c" tag)
         └── "Smith"

Decomposed into:
    Subtree 1: root → citation → journal → "Nature"
    Subtree 2: root → citation → year → "2024"  
    Subtree 3: root → citation → author → "Smith"
```

**Decomposition rules are configurable:**
- Break at specific tag names (e.g., every `<author>` is a subtree root)
- Break at depth thresholds
- Break at content size thresholds
- Break on user-defined conditions

**Benefits:**
- Subtrees can be indexed independently
- Queries can target specific subtrees without loading entire document
- Parallel processing of subtrees

### Parent-Child Index Entries

For each parent-child relationship, an index entry is created:

| Field | Example |
|-------|---------|
| Parent path | `/citation/author` |
| Parent tag | `author` |
| Child path | `/citation/author` |
| Child value | `Smith` |
| Occurrence | 1st, 2nd, 3rd... |

This enables efficient queries like:
```xquery
/citation/author[. = "Smith"]/../year  (: Find Smith's year :)
```

---

## US8935267B2: Pre-Computed Selective Indices

### The Problem Being Solved

Different query languages (SQL, XQuery, XPath, full-text) had incompatible index requirements. This patent describes how MarkLogic creates **universal pre-computed indices** that serve all query types.

### Index Parameters

> "Index parameters 300 are specified. The index parameters 300 may be specified through the user interface or they may be specified in a default configuration file." — US8935267B2

**Configurable index parameters include:**
- Which paths to index (selective — not all paths)
- Index granularity (element, attribute, text)
- Range index settings (numeric, date, string bounds)
- Word indexes for full-text search
- Geospatial index settings

### How Pre-Computed Indices Work

The data loader builds indices **as documents are ingested:**

```
Document ingested
      │
      ▼
┌─────────────────┐
│  Data Loader    │
│  - Tokenizer     │─── tokens
│  - Tree Analyzer│─── top-down tree
│                 │─── pre-computed indices
└─────────────────┘
      │
      ▼
┌─────────────────┐
│ Tree-Structured  │
│ Database         │
│  - Top-down trees│─── for XPath/XQuery
│  - Pre-computed  │─── for SQL, full-text,
│    indices       │    geospatial
└─────────────────┘
```

### Table Views from Indices

A key innovation: **pre-computed indices are mapped to table columns**:

> "One or more table views may then be defined using the pre-computed indices 304. That is, indices are mapped to columns of a table, as shown below. A SQL query is then resolved against a table view." — US8935267B2

```
Pre-computed Index          Table View
─────────────────           ───────────────
/citation/author    ──→   author (column)
/citation/year      ──→   year (column)
/journal            ──→   journal (column)
```

**Result:** SQL queries work on tree-structured XML data without converting XML to relational tables. The index IS the table.

### Supported Query Languages

The pre-computed indices support multiple languages simultaneously:

| Language | What it uses |
|----------|--------------|
| **XQuery** | Top-down trees + pre-computed indices |
| **XPath** | Top-down trees |
| **XSLT** | Top-down trees |
| **Full-text** | Word/term indices |
| **Geospatial** | Spatial indices |
| **SQL** | Table views from indices |

---

## Path Expressions in MarkLogic Index

MarkLogic indexes support all standard XPath patterns:

### Simple Paths
```xquery
/names/name/first           (: /a/b/c pattern :)
```

### Paths with Predicates
```xquery
/names/name[middle="James"]/first    (: /a/b[cond]/c :)
```

### Wildcard Paths
```xquery
/*/name/first              (: /*/b/c - any tag at position :)
/names//first              (: //c - descendant anywhere :)
```

### Paths with Position
```xquery
/names/name[1]/first       (: First name only :)
```

### Attribute Paths
```xquery
/order/@id                 (: @id attribute :)
/order/item[@qty > 100]    (: Attribute predicate :)
```

---

## How Queries Use the Index

### Query Decomposition

When a query arrives, the query processor:

1. **Decomposes** the query into parts
2. **Matches** parts against pre-computed indices
3. **Collects** matching data using index statistics
4. **Filters** results with any remaining predicates
5. **Returns** results

```
XQuery: /citation/author[. = "Smith"]/../year

Decomposed into:
  1. Find all authors matching "Smith"     → uses value index
  2. Navigate to parent <year>             → uses tree structure
  3. Return year values                    → uses path index
```

### Index Usage Example

Given this document:
```xml
<citations>
  <citation>
    <author>Smith</author>
    <year>2024</year>
    <journal>Nature</journal>
  </citation>
</citations>
```

**Pre-computed index entries:**

| Path | Value | Position |
|------|-------|----------|
| `/citations/citation/author` | Smith | 1 |
| `/citations/citation/year` | 2024 | 1 |
| `/citations/citation/journal` | Nature | 1 |

**Query:** Find the year for Smith

```xquery
/citations/citation[author = "Smith"]/year
```

**How it's resolved:**
1. Index lookup: find `/citations/citation/author` = "Smith" → finds position 1
2. Index lookup: find `/citations/citation/year` at same position → "2024"
3. Return result without loading full document

---

## "Selective" Indexing

Not everything is indexed — MarkLogic uses **selective indexing**:

> "While forming top-down trees for documents, selective pre-computed indices are formed." — US8935267B2

**You can configure:**
- Which element paths get range indexes
- Which paths get word indexes
- Which paths get geospatial indexes
- Element-level vs attribute-level indexing

**Benefits:**
- Reduces index size for sparse documents
- Focuses index on queryable paths
- Allows different index types for different data

**In MarkLogic:**
```xquery
xdmp:document Insert(..., 
  <options xmlns="xdmp:document-insert">
    <index-directories>true</index-directories>
    <schema...</schema>
  </options>
)
```

---

## Why "Universal"

The "universal" aspect comes from:

1. **Schema-less by default** — All paths indexed without declaring a schema
2. **Multi-language support** — SQL, XQuery, XPath, full-text, geospatial from one index
3. **Tree + relational** — Hierarchical and tabular views of same data
4. **No re-ordering** — Index represents original document structure

> "The pre-computed indices support multiple query languages, such as XQuery, XPath, XSLT, Full-text, Geospatial and SQL. Thus, the pre-computed indices support relational queries in a tree structured database, which otherwise does not support such queries." — US8935267B2

---

## Summary

| Patent | US7962474B2 | US8935267B2 |
|--------|-------------|-------------|
| **Focus** | Subtree storage & parent-child indexing | Pre-computed selective indices |
| **Key concept** | Decompose XML trees into indexed subtrees | Map indices to table columns for SQL |
| **Enables** | Efficient hierarchical queries | Multi-language query support |
| **What it indexes** | Tree structure + parent-child relationships | Path values + statistics |
| **Used by** | XQuery, XPath | XQuery, XPath, XSLT, SQL, full-text, geo |

Together, they form the foundation of MarkLogic's Universal Index — indexing every document's structure and content in a way that supports any query language without pre-defined schemas or manual index management.

---

## References

- [US8935267B2 — Pre-computed indices](https://patents.google.com/patent/US8935267B2/en)
- [US7962474B2 — Parent-child indexing](https://patents.google.com/patent/US7962474B2/en)
- [MarkLogic Universal Index Documentation](https://docs.marklogic.com/)

---

*Research document generated from patent analysis.*
*Patents: US8935267B2 (filed 2012), US7962474B2 (filed 2002)*
