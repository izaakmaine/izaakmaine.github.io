# Disable Right Click Analysis and Plan

## Goal
Prevent users from right-clicking on any page to discourage downloading images.

## Analysis
The site is built with Hugo using the `hextra` theme (imported as a module).
The `hextra` theme supports custom code injection via partials. Specifically, it looks for:
- `layouts/partials/custom/head-end.html`: To add scripts to the end of the `<head>` tag.
- `layouts/partials/custom/body-end.html`: To add scripts to the end of the `<body>` tag.

## Plan

### 1. Create JavaScript File
Create a new file `static/js/disable-right-click.js` containing the logic to disable the context menu and drag events.

**Content:**
```javascript
document.addEventListener('contextmenu', event => event.preventDefault());

// Optional: specific image protection
document.addEventListener('dragstart', event => {
  if (event.target.tagName === 'IMG') {
    event.preventDefault();
  }
});
```

### 2. Inject Script via Theme Override
Create the file `layouts/partials/custom/head-end.html`.

**Content:**
```html
<script src="{{ "js/disable-right-click.js" | relURL }}"></script>
```

### 3. Verification
1.  Run `hugo server`.
2.  Open the website in a browser.
3.  Try to right-click on the background and images.
4.  Try to drag images.
5.  Verify that the context menu does not appear and images cannot be dragged.

## Notes
- This method disables the right-click menu globally.
- Technical users can still download images by inspecting the source or disabling JavaScript. This is a deterrent, not a secure prevention method.
