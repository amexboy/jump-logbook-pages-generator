# PDF Logbook Generator

This is a Vue 3 application built with Vite that allows users to generate customized PDF logbooks. It provides a web interface to select various options for the logbook, including cover pages, page layouts, and page ranges.

## Features

*   **Customizable Cover Page**:
    *   Option to include or exclude a cover page.
    *   Select from predefined background images or upload your own.
    *   Select from predefined SVG cover designs or upload your own SVG file.
*   **Logbook Page Layouts**:
    *   Choose from multiple predefined logbook page layouts (e.g., Base Design, Skydiving Design).
*   **Page Range Selection**:
    *   Specify the starting and ending page numbers for the logbook.
*   **PDF Generation**:
    *   Generates a PDF document based on the selected customizations.
    *   The generated PDF is displayed in an iframe for preview.
*   **Dynamic Content**:
    *   Page numbers are automatically inserted into the logbook pages.
    *   Layouts are designed for double-sided printing with appropriate gutter margins.

## How to Use

1.  **Setup**:
    *   Clone the repository.
    *   Install dependencies: `npm install`
    *   Run the development server: `npm run dev`
2.  **Access the Application**:
    *   Open your browser and navigate to the local development URL provided by Vite (usually `http://localhost:5173`).
3.  **Configure Your Logbook**:
    *   **Include Cover Page**: Check the "Include Cover Page" box if you want a cover.
    *   **Logbook Page Layout**: Select your preferred layout for the logbook pages.
    *   **Cover Options** (if cover is included):
        *   **Cover Background Image**: Choose between "Use Predefined" (select from a dropdown) or "Upload Image" (upload your own image file).
        *   **Cover SVG Design**: Choose between "Use Predefined" (select from a dropdown) or "Upload SVG" (upload your own SVG file).
    *   **Page Range**: Set the "From Page" and "To Page" numbers.
4.  **Generate PDF**:
    *   Click the "Generate PDF" button. The button will be enabled once all required assets (SVG designs, images if selected) are loaded and page numbers are valid.
    *   The generated PDF will appear in an iframe below the controls.

## Recommended IDE Setup

*   [VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur if installed).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```

## Customize Configuration

See [Vite Configuration Reference](https://vite.dev/config/).
