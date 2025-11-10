# A4 to A7 Document Splitter

A Vue.js web application that splits A4 scanned documents into 8 A7-sized JPG images.

## Features

- Upload A4 documents in various formats (JPG, PNG, PDF, etc.)
- Automatically splits each A4 page into 8 A7-sized pieces (2 columns × 4 rows)
- Generates JPG images with original filename + number (1-8, 9-16, etc.)
- Drag-and-drop file upload support
- Real-time preview of generated images
- Batch download all generated images
- Supports multi-page PDFs (generates 8 images per page)

## Setup

1. Install dependencies:
```bash
npm install
```

2. Run development server:
```bash
npm run dev
```

3. Open your browser to the URL shown (typically http://localhost:5173)

## Usage

1. Click the upload area or drag and drop an A4 document
2. Click "Split into A7 Images" to process
3. Preview the generated images
4. Click "Download All Images" to save them to your computer

## How it works

The application automatically detects the orientation of your document and splits it accordingly:

### Portrait A4 (2×4 grid = 8 A7 landscape pieces):

```
┌─────┬─────┐
│  1  │  2  │
├─────┼─────┤
│  3  │  4  │
├─────┼─────┤
│  5  │  6  │
├─────┼─────┤
│  7  │  8  │
└─────┴─────┘
```

### Landscape A4 (4×2 grid = 8 A7 portrait pieces):

```
┌───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │
├───┼───┼───┼───┤
│ 5 │ 6 │ 7 │ 8 │
└───┴───┴───┴───┘
```

Each piece is exported as a properly proportioned A7-sized JPG image (74mm × 105mm ratio) with the naming format:
`originalname_1.jpg`, `originalname_2.jpg`, ... `originalname_8.jpg`

For multi-page PDFs, numbering continues: page 2 would be `originalname_9.jpg` through `originalname_16.jpg`, etc.

## Build for Production

```bash
npm run build
```

The built files will be in the `dist` directory.
