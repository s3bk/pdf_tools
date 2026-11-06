# pdf_tools
Possibly useful tools for working with PDFs that don't fit into the pdf crate.

# Contents

## Page text extractor:
See `src/bin/pdf2text.rs` for details.

## Page extractor and merger: `pdf-cut`

Copies selected pages out of one or more PDFs into a new one. Pages are deep
cloned with the writers from the pdf crate (`PdfBuilder`, `PageBuilder`,
`CatalogBuilder`, `Importer`), so the result is self-contained: fonts, images
and other resources a page refers to are copied along with it.

```text
pdf-cut -i foo.pdf -p 2-5 -i bar.pdf -p 1 -o merged.pdf
```

Pages appear in the order the inputs and selections are given, so a page can be
taken twice by naming it twice (`-i a.pdf -p 3,1` takes page 3 and then page 1).

### Options

| option | meaning |
| --- | --- |
| `-i`, `--input <file>` | input PDF, may be given several times |
| `-p`, `--pages <spec>` | pages to take from the preceding `-i` (default: all pages) |
| `-o`, `--output <file>` | the PDF to write |
| `--verify` | compare every copied object with its source (slow, catches a broken import) |
| `-q`, `--quiet` | do not print the summary |

### Page selections

Page numbers are 1-based, the way a PDF viewer counts them.

| spec | pages |
| --- | --- |
| `3` | page 3 |
| `2-5` | pages 2, 3, 4, 5 |
| `4-` | page 4 to the last page |
| `-3` | pages 1, 2, 3 |
| `1,4,7-9` | pages 1, 4, 7, 8, 9 |
| `all` / `*` | every page |

A selection that runs past the end of a document is an error rather than being
silently clipped, so a typo cannot quietly drop pages:
`pdf-cut: foo.pdf: page 9 is out of range, the document has 7 page(s)`.

### Exit codes

`0` on success, `1` when the PDFs cannot be read or written, `2` for a bad
command line.

# Notes

## pdf 0.10 object numbering

Written files number their objects from 0, because `XRefTable::new` in
`pdf/src/xref.rs` pushes the mandatory free object 0 and then resizes the table
to `num_objects`, which drops that entry again when a new document starts out
with an empty table. Object 0 is free by definition (`7.5.4`), so a reference
to it, such as a page in `/Kids`, is unreadable: `Tried to dereference free
object nr 0.`

`pdf-cut` works around this by writing an empty placeholder as object 0 before
anything else, which keeps every real object at 1 and above. The clean fix is
in the pdf crate: `entries.resize(num_objects.max(1) as usize, XRef::Invalid);`

## What the output contains

`pdf-cut` copies *pages*, not documents: a page is rebuilt from its content
stream and the resources its operators actually select.
`PageBuilder::clone_page` walks the operators and pulls in the fonts, colour
spaces and XObjects they name, so

* an object used by several selected pages is copied once (the `Importer`
  keys its copies by the source reference), and
* an object nothing selected uses — a resource a page lists but never selects,
  the content of a page that was not taken — is not written at all.

`tests/dedup.rs` pins both down: it merges two hand-built pages that share one
font and checks the font appears once, that a font only an unselected page uses
does not appear, and that a font listed in `/Resources` but never selected by
an operator does not appear either.

## Building against a local pdf checkout

`Cargo.toml` carries a `[patch.crates-io]` that points `pdf` at
`/home/hematite/task-clones/pdf/pdf`, the local checkout with the fixes this
tool was developed against. Drop that section when the fixes are upstream and
released; nothing else depends on it.

## pdf 0.9 vs 0.10

This crate now builds against `pdf` 0.10, where `/Font` entries of a page's
resources are `Lazy<Font>` instead of `MaybeRef<Font>`; `src/lib.rs` loads them
with `Lazy::load`.
