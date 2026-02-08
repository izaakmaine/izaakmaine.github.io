# Implementation Plan: Slideshow for Static Images

## 1. User Choices
- **Strategy**: Option B (Static Directory).
- **Display**: Slideshow (Carousel).

## 2. Directory Structure
You will need to organize your images in the `static` folder.
**Action**: Create `static/images/`.

**Usage Example**:
For the document `content/docs/01-the-wedding-at-cana.md`:
1.  Create folder: `static/images/01-the-wedding-at-cana/`
2.  Place images inside: `01.jpg`, `02.jpg`, etc.
3.  In the markdown file, use the shortcode:
    ```markdown
    {{< slideshow path="images/01-the-wedding-at-cana" >}}
    ```

## 3. Shortcode Implementation (`layouts/shortcodes/slideshow.html`)

We will build a responsive, touch-friendly slideshow.

### Logic
1.  **Input**: `path` argument (relative to `static/`).
2.  **Process**:
    -   Use `readDir` to list files in `static/` + `path`.
    -   Filter by image extensions (`.jpg`, `.png`, `.webp`, etc.).
    -   Sort files by name.
3.  **Output**: HTML structure with embedded CSS/JS.

### HTML Structure
```html
<div class="slideshow-container" id="slideshow-{{ .Ordinal }}">
  <div class="slides-wrapper">
    <!-- Loop images -->
    <div class="slide">
        <img src="/path/to/image.jpg" loading="lazy">
    </div>
  </div>
  <button class="prev">❮</button>
  <button class="next">❯</button>
  <div class="dots-container">
    <!-- Dots for navigation -->
  </div>
</div>
```

### CSS & JS
-   **CSS**: Flexbox/Grid for layout, absolute positioning for controls. Transitions for sliding effect.
-   **JS**: Handle Next/Prev clicks, auto-advance (optional), touch swiping.
-   **Optimization**: Use `loading="lazy"` on images.

## 4. Next Steps
1.  Create the `static` directory structure.
2.  Implement the `slideshow.html` shortcode.
3.  (User Action) You will confirm when you have added images to valid paths so we can verify.
