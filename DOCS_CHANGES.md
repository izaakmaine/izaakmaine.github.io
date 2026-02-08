# Documentation Changes

## Fixes Applied

### Slideshow Rendering Issue in `01-the-wedding-at-cana.md`
- **Issue**: The slideshow shortcode was indented by 4 spaces, causing Hugo to interpret it as a code block instead of a shortcode.
- **Fix**: Removed the indentation to allow proper rendering.

**File**: `content/docs/01-the-wedding-at-cana.md`
**Change**:
```diff
-    {{< slideshow path="images/01-the-wedding-at-cana" >}}
+{{< slideshow path="images/01-the-wedding-at-cana" >}}
```
