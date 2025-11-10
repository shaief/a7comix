# A4 to A7 Document Splitter

A Vue.js web application that splits A4 scanned documents into 8 A7-sized JPG images.

## Features

- Upload A4 documents in various formats (JPG, PNG, PDF - max 2 pages)
- Automatically splits each A4 page into 8 A7-sized pieces
- Select whether the page represents "First Page" or "Second Page" of your document
- Generates JPG images numbered accordingly (1-8 for first page, 9-16 for second page)
- Rotate documents before splitting (0°, 90°, 180°, 270°)
- Drag-and-drop file upload support
- Real-time preview of the document
- Batch download all generated images
- Automatic orientation detection (portrait/landscape)

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
2. Preview the document and select if it's your "First Page" or "Second Page"
3. Rotate if needed using the rotation buttons
4. Click "Split into A7 Images" to process
5. Preview the generated images
6. Click "Download All Images" to save them to your computer

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

Each piece is exported as a properly proportioned A7-sized JPG image (74mm × 105mm ratio).

**File Naming:**
- **First Page** selected: `originalname_1.jpg` through `originalname_8.jpg`
- **Second Page** selected: `originalname_9.jpg` through `originalname_16.jpg`

This allows you to process both pages of a two-page document separately and get correctly numbered output files.

## Deployment

This project is configured for automatic deployment to GitHub Pages.

### Live Demo
Visit: **https://shaief.github.io/a7comix/**

### Automatic Deployment

The app automatically deploys to GitHub Pages when you push to the main/master branch. The GitHub Actions workflow handles the build and deployment process.

### Manual Deployment Setup

If this is your first deployment, enable GitHub Pages in your repository:

1. Go to your GitHub repository settings
2. Navigate to **Settings** → **Pages**
3. Under **Source**, select **GitHub Actions**
4. Push your code to trigger the deployment

The workflow will automatically build and deploy the site.

## Build for Production

```bash
npm run build
```

The built files will be in the `dist` directory.
