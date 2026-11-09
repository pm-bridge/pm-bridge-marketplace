---
name: one-off-task
description: How to do a one-off job the owner needs now and no tool covers - edit or sign a PDF, convert a document, resize or convert images, pull a table out of a file - by writing and running a small throwaway script on this Mac, then sending the result back. No plugin is needed for it.
metadata:
  tools: run_command read_file write_file list_dir read_attachment save_attachment send_file ask_owner recall
---

# Do a one-off job with a throwaway script

Use this when the owner needs something done **now**, once, and no plugin tool does it: add a
signature image to a PDF, merge or split PDFs, fill in a PDF form, turn a Word file into text,
convert or shrink a batch of images, pull a table out of a document into a spreadsheet. You do it
yourself with `run_command`: a small script, run once, its result sent back with `send_file`.

Everything happens on this Mac. Never upload the owner's files to an online converter or any
website to get the job done.

## 1. Before you start

- Say in one line what you will do and what you will hand back (for example: "I'll put your
  signature on page 3, bottom right, and send back contract-signed.pdf").
- Ask with `ask_owner` only for what you cannot see: which page, where on the page, which file is
  the signature, what size. Do not guess anything the result depends on.
- You need the **workspace** level or higher to write files. On the read-only level you cannot do
  this job: tell the owner it needs their permission raised to Workspace on your profile.

## 2. Where things go

Two places, so the owner's project never fills up with leftovers:

- **The job's files** (the owner's inputs, your script, the results):
  `<workspace>/.pmb-quick/<job>/`, where `<job>` is a short name such as `sign-contract`.
  They must be inside the workspace: `save_attachment` writes there and `send_file` sends only from
  there.
- **The throwaway environment** (a Python virtual environment and the libraries in it):
  `${TMPDIR%/}/pmb-quick/<job>/`. It is outside the workspace and is deleted when you finish.

Get the owner's attachments in with `save_attachment` (for example to
`.pmb-quick/sign-contract/contract.pdf`). Never change an original file: write the result under a new
name such as `contract-signed.pdf`.

## 3. Tools already on this Mac (no install)

Check with `command -v <name>` before you rely on one.

| Job | Command |
|---|---|
| Convert documents: txt, rtf, html, doc, docx, odt | `textutil -convert docx in.html -output out.docx`, `textutil -convert txt in.docx -output out.txt` |
| Resize or convert images | `sips -Z 1600 in.jpg --out out.jpg` (longest side 1600 px), `sips -s format png in.heic --out out.png`, `sips -r 90 in.png` (rotate) |
| Image size and other facts | `sips -g pixelWidth -g pixelHeight in.png`, `mdls in.pdf` |
| Audio and video | `ffmpeg`, only if `command -v ffmpeg` finds it: it is not part of macOS |

For many files, loop in one command: `for f in in/*.jpg; do sips -Z 1600 "$f" --out "out/$(basename "$f")"; done`.

## 4. A throwaway Python environment

For PDFs, spreadsheets and anything the built-in tools do not cover:

```sh
J="${TMPDIR%/}/pmb-quick/sign-contract"
mkdir -p "$J"
python3 -m venv "$J/venv"
"$J/venv/bin/pip" -q --disable-pip-version-check install pymupdf
"$J/venv/bin/python" .pmb-quick/sign-contract/sign.py
```

- PyPI is on the usual developer sites, so the install needs no approval. A package from another
  site, or a command that needs the network for something else, makes the owner's card appear:
  give a one-line reason.
- If `python3` is missing, or asks to install the command line developer tools, stop and tell the
  owner to run `xcode-select --install` once. You cannot install it yourself.
- Write the script into the job's folder with `write_file`, then run it. Keep it short and plain:
  it is used once.

Libraries that suit common jobs:

| Job | Library |
|---|---|
| PDFs: read text, add text or an image (a signature), merge, split, rotate, render a page as an image, fill form fields | `pymupdf` (imported as `fitz`) |
| Excel files | `openpyxl` |
| Word files, beyond what `textutil` does | `python-docx` |
| Tables out of a PDF | `pdfplumber` |

### Example: a signature image on a PDF page

```python
import fitz  # pymupdf

doc = fitz.open(".pmb-quick/sign-contract/contract.pdf")
page = doc[2]                                  # page 3 (counting from 0)
w, h = page.rect.width, page.rect.height       # in points, origin top left
box = fitz.Rect(w - 220, h - 120, w - 40, h - 60)
page.insert_image(box, filename=".pmb-quick/sign-contract/signature.png", keep_proportion=True)
doc.save(".pmb-quick/sign-contract/contract-signed.pdf")
```

A signature on a white background shows as a white box over the text: ask for a PNG with a
transparent background, or tell the owner that is what they will see.

Before you send a result whose layout matters, look at it yourself first. Render the page as an
image with `page.get_pixmap(dpi=110).save(".pmb-quick/sign-contract/page3.png")`, and send that
preview with the result.

A signature image placed this way is **not** a certified digital signature: it carries no
certificate and no audit trail. Say so when the document is a contract or any other legal paper.

## 5. Hand it back and clean up

- Send each result with `send_file` (the path in the workspace, and a short caption): what you
  changed, where, and the original's name. Say if anything could not be done.
- Then delete the throwaway environment: `rm -rf "${TMPDIR%/}/pmb-quick/<job>"`.
- Leave `.pmb-quick/<job>/` in place until the owner has the result, and tell them where it is. Ask
  whether to delete it.

## 6. When the same kind of job comes back

Before you start, `recall` whether you did this kind of job before. If this is the second time,
ask the owner once whether to keep the script as a skill of its own, so the next time takes a
minute. Only if they say yes: a job done once does not need to be kept.
