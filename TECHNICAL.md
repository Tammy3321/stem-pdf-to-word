# Technical overview

This page explains what the converter does and how. For everyday use, see the [README](README.md).

## Contents of the notebook

| Cell | Purpose |
|---|---|
| 1 | Checks for a GPU and Python 3.11–3.13, then asks for the PDFs |
| 3 | Installs pinned dependencies |
| 5 | Writes the converter, `pdf2word_hybrid.py` (the whole engine, one file) |
| 7 | Runs the converter once per PDF, each in its own work folder |
| 9 | Downloads the Word file(s) and prints the per-page check report |
| 11 | Optional: zips the per-page records for debugging |

The converter is a normal command-line script, so you can also run it outside Colab: run cell 5 in Jupyter, or copy its contents into `pdf2word_hybrid.py`.

## Pipeline

```
PDF
 └─ render each page at 300 dpi (PyMuPDF)
     └─ layout boxes: text, titles, formulas, figures, tables (PP-DocLayout_plus-L)
         └─ coverage pass: add print the layout missed (option labels, "8.", small formulas, rings)
             └─ read each region: text + inline/display LaTeX in one pass (PaddleOCR-VL-1.5, GPU)
                 └─ validate every read; look again where it is implausible
                     └─ clean up: tags, running heads, reading order, figure crops
                         └─ per-page record (JSON "IR")  ← cached; can be edited by hand
                             └─ build: question/option structure, chemistry, pandoc → Word equations
                                 └─ output.docx + a report of anything doubtful
```

### 1. Layout and coverage

PP-DocLayout_plus-L proposes boxes. Layout models routinely skip small items, so a coverage pass looks for ink no box covers (option labels, hanging question numbers, short formulas) and turns it into regions. Two more detectors work on the page image:

- **Drawn structures.** A formula box that holds a vertical bond between rows, or a circled charge over an atom, is a 2-D drawing that no linear formula can express. It is kept as a picture.
- **Rings outside any box.** Thin-stroke closed shapes (benzene rings) are found and grown along their row to include their substituents.

All size rules are relative to the page's measured **glyph height** (the median height of letter-sized marks), so they work at any font size or scan resolution.

### 2. Reading

Each region is cropped and read by **PaddleOCR-VL-1.5** (0.9B parameters, fp16 on a CUDA GPU). It returns text with LaTeX for maths. Figure and table areas are blanked from the image the reader sees, so their labels don't leak into the text. Table regions are read separately with the model's table prompt and become Word tables; a table that cannot be parsed is read as text. Very tall regions are split so the model doesn't downscale small print.

### 3. Validation and retry

The reader is checked rather than trusted.

| Check | How |
|---|---|
| Length vs. print | Compares the characters read with the letter marks actually printed in the region. The ratio is calibrated per page, so it adapts to font and scan. |
| Loops | Detects runaway repetition ("A A A A …") and collapses it. |
| Failure sentences | Catches answers like "The image is too blurry …". |
| PDF text layer | When the PDF carries real text, compares the read with the PDF's own words, and corrects numbers that disagree. |

A read that fails a check gets a **second look**: the same print on a larger canvas, or split into smaller pieces. The better of the two attempts is kept. Exam tags that sit alone at the right end of a line, or on the line below, are re-read on their own.

Whatever still fails is never dropped silently. It goes into the document as a picture of that region or is flagged, and every flagged page is listed in the final `CHECK against the PDF` line.

### 4. Assembly

- **Clean-up:** running heads and page numbers are dropped; the chapter title is recognised by letter size; exam tags are attached to their question.
- **Reading order:** column-aware, with fragments the layout split re-joined.
- **Figures:** cropped tightly from ink, then polished: cut-off glyphs, stray specks and edge rules are removed and the paper is levelled to white.
- **Questions and options:** rebuilt from the printed markers (`12.`, `(A)`, `i.`, `OR`) instead of from layout boxes, which often cut a question in two. A picture printed among the options becomes that option.

### 5. Word output

- **Equations:** all LaTeX goes to **pandoc** in one batch and is converted to native OMML. Anything pandoc cannot convert is written as readable text, never as a picture of text.
- **Chemistry:** chemical formulas become text with Unicode subscripts and charges (H₂SO₄, Fe²⁺(aq)), because Word equations would set every letter in italic as a variable. Reactions stay equations, but upright. When a document contains three or more formulas it is treated as chemistry, which also converts unmarked formulas (CaO), common molecules (H₂, O₃, P₄) and electron configurations. Gas-law variables such as PV or P₂ are never touched.
- **Layout:** python-docx writes questions with hanging numbers, options in a grid, and tags right-aligned.

## Robustness

- **Page retry:** a page that crashes (for example a GPU memory spike) is retried once.
- **Page fallback:** if a page still fails, the PDF's own text layer is used; if the PDF has none, the page goes in as a picture. The report says which pages.
- **Resume:** every page's result is saved to `work/<pdf>/ir/page_NNN.json` atomically, so re-running the Convert cell resumes at the first unfinished page. A different PDF in the same work folder starts clean.
- **Rebuild:** after hand-editing a page record, `--rebuild` regenerates the Word file without reading again.

## Engines

| Engine | When | What it uses |
|---|---|---|
| `vlm` (default with a GPU) | CUDA available | PP-DocLayout + PaddleOCR-VL-1.5 |
| `hybrid` (CPU fallback) | No GPU, or forced | PP-OCRv5 text + Pix2Text MFD/MFR formula models run as ONNX |

The CPU engine is much slower and weaker on scans. The notebook refuses to run without a GPU unless `ALLOW_CPU = True` is set in cell 1.

## Command line

```bash
python pdf2word_hybrid.py input.pdf -o output.docx --work work
python pdf2word_hybrid.py input.pdf -o output.docx --work work --pages 1-5
python pdf2word_hybrid.py --rebuild -o output.docx --work work      # after editing work/ir/*.json
python pdf2word_hybrid.py --check-env
```

| Option | Meaning |
|---|---|
| `--work DIR` | Folder for page images and per-page records (the cache) |
| `--pages 1-3,7` | Convert only these pages |
| `--force` | Re-read pages even if cached |
| `--rebuild` | Build the Word file from the cached records only |
| `--engine auto\|vlm\|hybrid` | Choose the recognition engine |
| `--cache DIR` | Model cache folder (for example on Google Drive) |
| `--tables`, `--subject`, `--rules`, `--accurate`, `--verify`, `--mfd-weights` | Tuning for the CPU engine (`--tables` loads PP-StructureV3 for tables; slower) |

## Environment

Tested on Google Colab (Python 3.13, T4 GPU). The dependencies are pinned in cell 3 because newer releases have broken things:

- `paddlepaddle==3.2.2` (3.3.0 crashes CPU inference in its oneDNN path)
- `paddleocr[doc-parser]==3.3.2`
- `transformers==5.22.2`, `accelerate==1.15.0`

Colab's preinstalled `langchain` is removed, because PaddleX imports a module that langchain 1.x deleted. Models download from Hugging Face rather than Baidu's CDN, which often times out.

## Known limitations

- Handwriting is not supported.
- Drawn chemical structures stay pictures. Structure-recognition models were considered but not used: a silently wrong molecule in a question bank is worse than a clean picture.
- Complex tables (merged cells, formulas in cells) may need touching up.
- Accuracy follows scan quality. The validation step reports doubtful pages, but proofreading is still recommended.
