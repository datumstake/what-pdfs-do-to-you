# What PDFs actually do to you

**A defect log from three document-extraction tools. Every entry below produced a clean
build, a successful run, and a wrong answer.**

Document extraction has a failure mode that almost nothing else in software has. When a web
server breaks, it 500s. When a parser breaks, it throws. When a document pipeline breaks, it
writes a row — a confident, well-formed, completely plausible row — and nobody finds out for
six months.

I have spent a while building three of these: a tool that turns mechanical drawings into
AS9102 inspection forms, a single-header C++ PDF text extractor, and a pipeline that posts
recorded legal documents to a database. What follows is every defect I found in them that a
successful run did not. I publish it because the thing a buyer is actually afraid of is not
incompetence. It is **confident incompetence**, and a failure log is the only thing that
addresses it.

The defects fall into three groups, and they get progressively worse.

---

## Part 1 — The text layer lies to you

### The characters in the PDF are not the characters on the page

This is the single most common reason someone says "my PDF has no text." It does have text.
You are reading the wrong bytes.

When a PDF embeds a *subset* font — which almost every CAD exporter, invoicing system and
typesetting tool does, because it keeps the file small — the bytes inside the string
operators are **glyph indices in a custom per-document order**, not character codes. Byte
`0x01` is not `SOH`. It is "whatever glyph the producer decided to put in slot 1 of this
particular font."

The real text lives in a `/ToUnicode` CMap, which is two object hops from the content
stream:

```
content stream:   /FB 12 Tf  <0102030405> Tj
page resources:   /Font << /FB 7 0 R >>
object 7:         /ToUnicode 8 0 R
object 8:         the CMap -- bfchar and bfrange sections
```

Decode `<0102030405>` as WinAnsi and you get control characters. Decode it through the CMap
and you get `WIDTH`.

**The expensive part:** a naive implementation matches the CMap's own `/CMapName` against
the name used by `Tf`. That is wrong, and it *appears* to work on files where the two
coincide. `/CMapName` is the CMap's own name; `Tf` names a *resource*, which points to a
font *object*, whose `/ToUnicode` points to the CMap. Two hops, and there is no way around
them.

### Filters chain, and the first one is invisible

`/Filter [/ASCII85Decode /FlateDecode]` is what ReportLab writes, among others. The bytes
between `stream` and `endstream` are ASCII85 *text* wrapping the deflate data.

Inflate them directly and you get garbage. Not an error — garbage. Your decompressor
returns `false`, you log "stream not compressed", you fall through to treating it as plain
text, and your extractor quietly finds nothing on that page.

I found this by looking at the output rather than assuming the first run was correct.

### Rotated text comes back at the origin, or negative

A vertical dimension callout is painted inside a transform:

```
q  0.7071 0.7071 -0.7071 0.7071  300 400 cm
BT /FA 12 Tf 1 0 0 1 0 0 Tm (rotated callout) Tj ET
Q
```

The text matrix alone says this text is at `(0, 0)`. It is actually at `(300, 400)`. If you
do not track the current transformation matrix and the `q`/`Q` stack, every rotated callout
on the sheet lands somewhere meaningless — and on a drawing, *most* dimension text is
rotated.

This one at least fails visibly, as long as you are looking at coordinates. If you only
collect strings, you will never notice, and then the first time someone asks "which of these
is the width?" you have no answer.

### Multi-sheet packages merge into a plausible lie

A three-sheet PDF with no page attribution flattens into one undifferentiated list of
fragments. That does not look like an error. It looks like a complete document.

On an AS9102 Form 3, the Reference Location field is sheet *and* zone. Merged sheets produce
a form where every reference points at the wrong drawing — a form that is fully populated,
internally consistent, and wrong in a way that only shows up when an inspector walks to the
wrong place on the wrong sheet.

`/Kids` gives the authoritative page order. File order is the fallback, and it is usually
but not always the same.

### The font resource is not page content, and it will happily look like it

A `/ToUnicode` CMap is a stream. If your traversal is a brute-force scan over every
`stream`/`endstream` pair — which is a legitimate strategy, and the one I chose, because it
sidesteps implementing cross-reference streams and object streams — then the CMap itself
gets scanned for text.

It contains hex strings. `<00> <ff>`, the codespace range, and every `bfchar` pair. They
come out as text fragments, unattributed to any page, sitting at the top of your output.

**I only caught this because a test asserted a line *count*.** Asserting that the expected
lines appear *somewhere* would have passed.

### Short streams were never detected as ASCII85 at all

My ASCII85 detector samples the first 64 bytes and requires every one to be in the ASCII85
alphabet. On a stream shorter than 64 bytes, the `~>` terminator falls *inside* its own
sample window — and `~` is above `u`, so the check fails against itself.

Every real-world stream is long enough to hide this. A test fixture is not.

---

## Part 2 — The semantics lie to you

Getting the characters out is the easy half. Now you have to decide what they *mean*, and
this is where the wrong answers start looking professional.

### Every diameter on the sheet became a length

The classifier ran on raw PDF bytes and tested for the UTF-8 diameter symbol. The bytes were
not UTF-8 yet. So `Ø0.3125` matched nothing, fell through to the default, and was recorded
as a LINEAR dimension.

Silent. Every diameter on every sheet. On an inspection record, "this hole is 0.3125 inches
long" is not a typo — it is a measurement instruction nobody can follow.

### A tolerance that lost its plus-or-minus

`±` written as a raw WinAnsi byte into a UTF-8 file becomes a replacement character.

**A tolerance that has lost its `±` still looks like a number.** That is the worst kind of
wrong: it passes every structural check you have, it renders fine, and the only way to catch
it is to compare against what was actually printed on the page.

### A flatness callout became a linear dimension

A geometric tolerance of `.002` came out as `LINEAR ".002"`.

Nothing on a mechanical drawing is two thousandths of an inch long. The number is right; the
*control it belongs to* is geometry — a symbol and datum references that live in the vector
layer, not the text layer. So the value comes through and its meaning does not.

### The title block became measurable characteristics

Alloy designation, part number, and scale ratio, all emitted as dimensions to inspect.

Fixed by requiring a numeric decimal point: every dimension on a drawing has one, and no
part number, revision letter, alloy code or scale ratio does. That rule is shaped exactly
like the domain and would be nonsense in any other.

### Hole charts became 36 fake characteristics out of 87 rows

A hole chart is a table: `H01`, `6.680`, `3.046`, row after row. Each cell came out as its
own inspection line with its own designator.

**Content cannot fix this.** `6.680` is either an X ordinate in a table or a 6.68 inch
length, and the bytes are identical. The only thing that distinguishes them is *where they
sit*: a table has an identifier column — three or more short ID-shaped tokens sharing an x
ordinate at different heights. Find that, and everything on one of its rows and to its right
is a cell.

This is the general lesson of the whole exercise. **When content cannot decide, geometry
can.** A label and its value are related by position and by nothing else; the file does not
record that they belong together.

### Then the fix for over-capture caused under-capture, which is worse

Suppressing whole table rows moved **twelve real toleranced diameters** out of the form.

A hole chart's diameter column reads `Ø0.3125 ±.003` — a nominal, a symbol and a tolerance.
That is a complete requirement wherever it happens to be printed, table or not.

> A spurious row costs two seconds to delete.
> A missing characteristic is one nobody inspects.

Those two errors are not worth trading one-for-one, and a pipeline tuned as though they were
will look like it is performing well right up until it matters.

### A legal description truncated at a line wrap

Most recent, and my favourite, because of how harmless it looks.

A label-and-next-line extraction strategy took only the first line of a value that wraps:

```
want:  Lot 13, Block 4 of KESSLER ADDITION, a subdivision in Section 15,
       Township 11 North, Range 17 West, according to the recorded plat thereof
got:   Lot 13, Block 4 of KESSLER ADDITION, a subdivision in Section 15,
       Township 11 North
```

That is not a parse error. It is a **legal description of a different piece of land**,
posted unattended at 95% confidence, into a title plant that people search to find out who
owns what.

---

## Part 3 — Your measurements lie to you, and this is the one that should scare you

Everything above was found by measuring. Which raises the obvious question: what if the
measurement is wrong?

It was. Three times.

### 100% recall, by coincidence

I scored recall by string-matching ground truth against output. It returned 100%. It was
matching bare geometric-tolerance values against unrelated numbers that happened to be
identical. The number was meaningless and it looked excellent.

### "36 of 42 characteristics were never drawn"

I then "discovered" that most of the ground truth did not appear in the PDF at all — by
grepping the file for ASCII numerals.

On subset fonts. Where bytes are glyph indices. The exact thing I had fixed the week before.

Both numbers were nonsense, and both looked plausible enough to act on.

### 100% precision at a confidence threshold of zero

The most recent one, and the most instructive. My scorer reported, for the deed pipeline:

```
 threshold   automation   precision   complete
      0.00         100%      100.0%        92%
      0.80          67%      100.0%       100%
```

At threshold zero — post everything, check nothing — precision was **100%**.

Because precision was only measured over fields that were *present*. **An absent value
cannot be wrong.** The pipeline was posting rows with null grantors and scoring perfectly
for it, and a title plant is searched *by* grantor. Those rows are not a partial win; they
are unsearchable records occupying a primary key.

The fix was a third number — completeness — which is what that `92%` is. But the
generalisable rule is harsher:

> **A metric that cannot go down is not measuring anything.**

If you cannot describe the input that makes your number worse, you do not have a metric. You
have a thermometer that reads 98.6 regardless.

---

## What I actually do about it

Not a methodology. Just the rules that came out of the defects above, each one paid for.

**1. Ground truth before extraction code.** Hand-key the correct answer for a handful of
real documents and commit it. Without it, "it works" is an opinion, and you cannot tell a
real improvement from a lucky run. This is the first hour of any engagement and it is never
the fun hour.

**2. The generator writes the expectations, not the parser.** My test corpora are assembled
byte by byte by a script that knows what it drew, and that script emits the expected output
beside each file. The parser never grades its own homework. CI regenerates the corpus and
fails on any diff, so the expectations cannot quietly drift into a rubber stamp.

**3. Reject rather than coerce.** `book` is a string, not an int, because `00412` is a real
book number and `int("00412")` is `412`, which indexes a different book. An unparseable date
returns null rather than something that sorts — a recording date silently defaulted to
1970-01-01 is a bug found years later.

**4. A value that fails validation is dropped, not down-weighted.** A malformed instrument
number is not a low-confidence answer. It is a wrong one.

**5. Confidence is per field, not per document.** A deed where the instrument number is
crisp and the legal description is half-cut-off is not "0.6 confident." It is certain about
one thing and unsure about another, and collapsing that to one score throws away the only
thing a reviewer needs: *which field to look at*.

**6. Report at least three numbers and refuse to average them.** How much posted without a
human. How much of that was right. How complete those rows were. One accuracy figure hides
the trade you are actually making.

**7. The exception queue carries the reason.** A folder of rejected files makes a human
re-do the triage the pipeline already did. Route on the weakest field, name it, record the
threshold it failed against.

**8. When content cannot decide, use geometry — and when geometry cannot decide either,
refuse.** My deed pipeline posts 8 of 12 documents unattended. The 4 it queues are one
county layout that prints no field labels at all, where the only thing separating grantor
from grantee is which line they sit on. Getting that backwards inverts the conveyance, and a
reversed deed reads as perfectly valid forever. So it does not guess.

---

## The code

All MIT, all run in one command.

| | |
|---|---|
| **[deedpipe](https://github.com/datumstake/deedpipe)** | Recorded-document extraction with the exception queue described above. Typed schema, per-field confidence, idempotent loader, three-number scoring. 100% posted precision and completeness against generator-written ground truth; CI fails on a single wrongly posted value. |
| **[pdftext](https://github.com/datumstake/pdftext)** | PDF text extraction in one dependency-free C++ header — the DEFLATE decoder, the ASCII85 chain and the `/ToUnicode` CMap handling from Part 1. 132 checks, green on g++/clang/MSVC at `-Werror`, ASan + UBSan, 312-file mutation sweep. |
| **[firstarticle](https://github.com/datumstake/firstarticle)** | The drawing tool most of Part 2 came out of. Ships with [its accuracy report](https://github.com/datumstake/firstarticle/blob/master/docs/ACCURACY.md) and the original defect log. |

---

## One honest limitation

Every number here was measured on synthetic corpora with known ground truth, plus a small
amount of real CAD output cross-checked against an independent parser. **A 100% on a
generated document is not a promise about yours.** That is stated in each repo rather than
buried, because a benchmark without its limits is marketing — which is, in the end, the same
failure as a pipeline without a scorer.

---

*Written by [Ken Deibel](https://github.com/datumstake). I build document extraction that
refuses to write a row it cannot stand behind. If you have documents doing something in this
list, I would genuinely like to hear about it.*
