# STEM PDF to Editable Word

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Tammy3321/stem-pdf-to-word/blob/main/pdf2word_hybrid.ipynb)

Turn maths, physics and chemistry PDFs (question banks, worksheets, textbook pages, even scans) into a Word document you can actually edit. It runs for free in Google Colab, with no installation on your computer.

## Why

Retyping a PDF full of equations is slow and error-prone. Ordinary PDF-to-Word converters either paste each page in as a picture or turn every formula into scrambled text. This notebook rebuilds the document properly:

| In the PDF | In the Word file |
|---|---|
| Text, questions, options | Editable text, laid out as numbered questions with (A)–(D) options |
| Equations, fractions, matrices | Native Word equations, editable in Word's equation editor |
| Chemical formulas (H₂SO₄, Fe²⁺, CH₃–CH₂–Br) | Proper text with real subscripts and charges |
| Chemical reactions | Upright equations with conditions on the arrow |
| Diagrams, graphs, drawn structures | Clean, tightly cropped pictures |
| Exam tags such as `[Mar 18]` or `[3 marks]` | Kept at the end of their question |
| Page numbers, running headers | Removed (the chapter title is kept) |

## Who it is for

Teachers, tutors, coaching centres and content teams who need to edit, reuse or reformat STEM material that exists only as a PDF.

## How to use it

1. Click **Open in Colab** above. Sign in with any Google account.
2. In Colab, choose **Runtime → Change runtime type → T4 GPU**, then **Save**.
3. Choose **Runtime → Run all**.
4. A **Choose Files** button appears under the first cell. Pick one or more PDFs.
5. Wait, and **keep the tab open** (free Colab can stop a notebook whose tab is closed). The Word file downloads by itself at the end; several PDFs arrive as one `.zip`. If your browser asks, allow the download.

Installing takes a few minutes the first time. After that, a page typically takes 15–90 seconds on the free T4 GPU.

## Checking the result

When the conversion finishes, the last lines under the **Download** cell tell you how it went:

- `Nothing was left out, pasted as a picture or flagged as doubtful.` The whole document was read cleanly.
- `CHECK against the PDF -> page 3: 1 doubtful read(s) ...` lists every page where something could not be read with confidence. Compare those pages with the original.

The converter never drops content silently. Whatever it could not read cleanly is either kept as a picture or listed in that line.

> **Always proofread.** Recognition is very good on clean print but not perfect, especially on low-quality scans.

## Good to know

- **Works best on:** printed (not handwritten) STEM material in one or two columns, as digital PDFs or reasonable scans.
- **Drawn chemical structures** (rings, branched molecules) come out as pictures, not editable drawings.
- **Tables** become Word tables. A table that cannot be parsed is kept as text, never as a picture.
- **Privacy:** your PDFs are processed inside your own Colab session. The notebook downloads its recognition models but uploads your files nowhere.
- **Free Colab limits:** Google sometimes has no GPU to give. If so, try again later. Very long documents may hit Colab's session time limit.

## Troubleshooting

| Problem | What to do |
|---|---|
| "No GPU on this runtime" | Runtime → Change runtime type → T4 GPU → Save, then Run all again. |
| It stopped halfway | Run the **Convert** cell again while the runtime is still connected. Pages already read are kept. |
| The download did not start | Allow downloads for colab.research.google.com, then run the **Download** cell again. |
| A page looks wrong | Set `DIAGNOSTICS = True` in the last cell and run it. It downloads a file showing what the converter saw on each page. |

## How it works

The notebook combines a page-layout model, a vision-language model that reads text and equations together, and a set of checks that catch and retry doubtful reads. The result is assembled into a Word file with native equations. See [TECHNICAL.md](TECHNICAL.md) for the details.

## Acknowledgements

Built on [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) (PP-DocLayout and PaddleOCR-VL), [PyMuPDF](https://github.com/pymupdf/PyMuPDF), [pandoc](https://pandoc.org/), [python-docx](https://github.com/python-openxml/python-docx) and [Pix2Text](https://github.com/breezedeus/Pix2Text) models.

## License

This project's code is released under the [MIT License](LICENSE). You are free to use it, change it and share it.

The libraries and models it installs keep their own licenses. Most are permissive (MIT, Apache-2.0, BSD). Two are **AGPL-3.0**: PyMuPDF, used to read PDFs, and Ultralytics, used only by the CPU fallback. Running the notebook for your own work is unaffected. If you ship a product or run a public service built on this project, check the AGPL terms for those two (PyMuPDF is also sold under a commercial license by Artifex). See [TECHNICAL.md](TECHNICAL.md#third-party-licenses) for the full list.
