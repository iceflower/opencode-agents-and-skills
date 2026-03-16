---
name: pdf-handling
description: >-
  PDF file reading and processing rules.
  Use when analyzing PDF files or extracting content from them.
---
# PDF File Handling Rules

## Conversion Procedure

- When reading PDF files, always follow this procedure:
  1. Convert the required pages to PNG images using `pdftoppm`
  2. Analyze the converted images to extract text
- Never include a PDF file directly in an API request body

## Prerequisite

- `poppler` must be installed
  - macOS: `brew install poppler`
  - openSUSE Tumbleweed: `sudo zypper install poppler-tools`
  - Ubuntu (LTS): `sudo apt install poppler-utils`
  - Windows: `choco install poppler` or `scoop install poppler`

## Page Range Strategy

- Read table of contents pages first to understand structure
- Convert only the pages needed for the current task, not the entire document
- For large PDFs (100+ pages), work in batches of 10-20 pages

## Temporary File Management

- Use `/tmp/` for converted PNG images
- Use descriptive prefixes for output files (e.g., `pdftoppm -png -f 1 -l 5 input.pdf /tmp/project_toc`)
- Clean up temporary PNG files after extraction is complete using `rm /tmp/<prefix>*.png`

## Large PDF Handling

- If the Read tool fails due to file size, always fall back to `pdftoppm`
- Estimate the PDF page offset by comparing displayed page numbers with actual PDF page numbers
- When searching for specific content, use the table of contents to calculate target page numbers rather than scanning sequentially
