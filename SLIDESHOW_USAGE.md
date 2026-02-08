# Slideshow Feature Usage Guide

This guide explains how to use the automated sequential slideshow feature in your Hugo documentation site.

## Overview
The feature allows you to display a folder of images as an interactive slideshow within any Markdown content file. It includes:
-   **Sequential Navigation**: Arrows and dot indicators.
-   **Fullscreen Mode**: Click any image or the "⛶" icon to maximize.
-   **Keyboard Support**: Use `Left`/`Right` arrows to navigate and `Escape` to exit fullscreen (only works when fullscreen is active).

## Quick Start

1.  **Prepare Images**:
    -   Create a folder for your images in `static/images/`.
    -   Example: `static/images/my-slide-folder`
    -   Place your image files (`.jpg`, `.png`, `.webp`, `.jpeg`) inside this folder.
    -   *Note*: Images are sorted alphabetically by filename. Name them sequentially (e.g., `01.jpg`, `02.jpg`) to control the order.

2.  **Add to Content**:
    -   Open the Markdown file where you want the slideshow to appear (e.g., `content/docs/my-page.md`).
    -   Insert the following shortcode:

    ```markdown
    {{< slideshow path="images/my-slide-folder" >}}
    ```

    -   The `path` parameter must match the folder path inside `static/` (excluding `static/` itself).

## Configuration

The shortcode accepts the following parameter:

| Parameter | Description | Required | Example |
| :--- | :--- | :--- | :--- |
| `path` | The relative path to the image folder inside `static/` | Yes | `images/vacation` |
| `id` | (Optional) A unique ID for the slideshow. If omitted, one is generated automatically. Use this if you have multiple slideshows on one page and experience conflicts. | No | `my-unique-slider` |

## Example Directory Structure

```text
my-hugo-site/
├── content/
│   └── docs/
│       └── 01-the-wedding-at-cana.md  <-- Contains shortcode
├── static/
│   └── images/
│       └── 01-the-wedding-at-cana/    <-- Image folder
│           ├── 01.jpg
│           ├── 02.jpg
│           └── 03.jpg
```

## Troubleshooting

-   **Images not showing?**
    -   Verify the folder exists in `static/`.
    -   Check the `path` in the shortcode matches the folder name exactly.
    -   Ensure image filenames end in supported extensions (`.jpg`, `.jpeg`, `.png`, `.webp`). Case sensitivity matters on Linux servers!

-   **Slideshow unstyled?**
    -   The CSS is embedded in the shortcode. Ensure your Hugo theme isn't overriding key styles like `display: none` on `.slide`.
