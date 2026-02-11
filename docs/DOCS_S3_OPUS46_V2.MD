# Migrating Images to Tigris Data (S3) & Updating the Slideshow — V2

> **Objective**: Move all images out of the GitHub repository into Tigris Data (S3-compatible object storage with built-in global CDN) and update the Hugo slideshow shortcode to serve images from the remote URL instead of the local `static/` directory.
>
> **Version**: V2 — Updated with confirmed Tigris URL pattern from a working upload.

---

## Table of Contents

1. [Current Architecture](#1-current-architecture)
2. [Why Tigris Data?](#2-why-tigris-data)
3. [Confirmed Tigris URL Pattern](#3-confirmed-tigris-url-pattern)
4. [Setting Up Tigris Data](#4-setting-up-tigris-data)
5. [Uploading Images](#5-uploading-images)
6. [Solving the `readDir` Problem — Image Manifests](#6-solving-the-readdir-problem--image-manifests)
7. [Hugo Configuration Changes](#7-hugo-configuration-changes)
8. [Shortcode Modification](#8-shortcode-modification)
9. [Content File Changes](#9-content-file-changes)
10. [Removing Images from the Repository](#10-removing-images-from-the-repository)
11. [Ongoing Workflow](#11-ongoing-workflow)
12. [Rollback Plan](#12-rollback-plan)
13. [Appendix: Full Modified Shortcode](#appendix-full-modified-shortcode)

---

## 1. Current Architecture

### How the Slideshow Works Today

The slideshow shortcode (`layouts/shortcodes/slideshow.html`) currently:

1. Receives a `path` parameter from the content file (e.g., `images/magellan`).
2. Calls Hugo's **`readDir`** on `static/<path>` to list all files in that local directory.
3. Filters for image extensions (`.jpg`, `.jpeg`, `.png`, `.webp`).
4. Renders each image as an `<img>` tag with `src="/<path>/<filename>"`.

### Content File Usage

```markdown
{{< slideshow path="images/magellan" >}}
```

### Current Image Directories

All images live inside `static/images/` in the Git repo:

| Directory | Gallery | Files |
|-----------|---------|-------|
| `static/images/columbus-pitch` | Columbus Pitch slideshow | `.jpg`/`.png` |
| `static/images/elcano` | Elcano slideshow | `.jpg`/`.png` |
| `static/images/luther` | Luther slideshow | `.jpg`/`.png` |
| `static/images/magellan` | Magellan slideshow | `.jpg`/`.png` |
| `static/images/manalo` | Manalo slideshow | 12 `.png` files (~2 MB each) |
| `static/images/paul` | Paul slideshow | `.jpg`/`.png` |
| `static/images/valladolid` | Valladolid slideshow | `.jpg`/`.png` |

### The Problem

- Images bloat the Git repository (binary files don't diff well).
- GitHub Pages has limited bandwidth/CDN capabilities compared to dedicated object storage.
- Every `git clone` downloads every image ever committed.

---

## 2. Why Tigris Data?

[Tigris Data](https://www.tigrisdata.com/) is an S3-compatible, globally distributed object storage service with:

- **Built-in CDN**: Objects are cached at edge locations automatically — no separate CDN setup.
- **S3 API Compatibility**: Works with AWS CLI, SDKs, and any S3-compatible tool.
- **Public Buckets**: Files can be made publicly accessible via HTTP URLs.
- **Free Tier**: Generous free tier suitable for static site assets.

---

## 3. Confirmed Tigris URL Pattern

> [!IMPORTANT]
> The actual public URL pattern for Tigris differs from the S3 API endpoint. Based on a confirmed working upload, the pattern is:

```
https://<bucket-name>.t3.storage.dev/<key>
```

### Working Example

| Component | Value |
|-----------|-------|
| **Bucket** | `izaakmaine-assets` |
| **Key** | `manalo/manalo_a.png` |
| **Full URL** | `https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png` |

### Key Observations

1. **The bucket name is in the subdomain** — `izaakmaine-assets.t3.storage.dev`, not `t3.storage.dev/izaakmaine-assets`.
2. **No `images/` prefix** — The `manalo` folder was uploaded to the bucket root, so the key is `manalo/manalo_a.png`, not `images/manalo/manalo_a.png`.
3. **The public CDN domain is `t3.storage.dev`** — This is different from the S3 API endpoint (`fly.storage.tigris.dev`) used for `aws s3` commands.

### URL Mapping (Local → S3)

| Local Path | S3 Key | Public URL |
|------------|--------|------------|
| `static/images/manalo/manalo_a.png` | `manalo/manalo_a.png` | `https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png` |
| `static/images/magellan/001.jpg` | `magellan/001.jpg` | `https://izaakmaine-assets.t3.storage.dev/magellan/001.jpg` |
| `static/images/elcano/slide1.png` | `elcano/slide1.png` | `https://izaakmaine-assets.t3.storage.dev/elcano/slide1.png` |

> The `images/` directory prefix is stripped when uploading. Each gallery folder sits directly at the bucket root.

---

## 4. Setting Up Tigris Data

### 4.1 Create an Account

Sign up at [console.tigris.dev](https://console.tigris.dev/).

### 4.2 Create a Bucket

Create a bucket (e.g., `izaakmaine-assets`). Enable **public access** so images can be loaded by browsers without authentication.

### 4.3 Obtain Credentials

From the Tigris dashboard, obtain:

| Credential | Description |
|------------|-------------|
| `Access Key ID` | Your AWS-style access key |
| `Secret Access Key` | Your AWS-style secret key |
| `S3 API Endpoint` | `https://fly.storage.tigris.dev` (for `aws s3` commands) |
| `Public CDN Domain` | `https://izaakmaine-assets.t3.storage.dev` (for browser access) |
| `Region` | `auto` (or as specified by Tigris) |

> [!NOTE]
> The **S3 API endpoint** (`fly.storage.tigris.dev`) used with the AWS CLI is different from the **public CDN domain** (`<bucket>.t3.storage.dev`) you put in browser URLs and `<img>` tags.

### 4.4 Configure AWS CLI

```bash
# Configure a named profile for Tigris
aws configure --profile tigris
# AWS Access Key ID: <your_access_key>
# AWS Secret Access Key: <your_secret_key>
# Default region name: auto
# Default output format: json
```

Alternatively, set environment variables:
```bash
export AWS_ACCESS_KEY_ID="<your_access_key>"
export AWS_SECRET_ACCESS_KEY="<your_secret_key>"
export AWS_ENDPOINT_URL_S3="https://fly.storage.tigris.dev"
```

### 4.5 Verify Access

```bash
aws s3 ls s3://izaakmaine-assets/ --endpoint-url https://fly.storage.tigris.dev --profile tigris
```

---

## 5. Uploading Images

### Upload Strategy

Strip the `images/` prefix when uploading so that gallery folders sit at the bucket root. This produces cleaner S3 keys and matches the confirmed working pattern.

### 5.1 Upload ALL Galleries at Once

```bash
# Each subdirectory under static/images/ becomes a top-level folder in the bucket
for dir in static/images/*/; do
    gallery=$(basename "$dir")
    echo "Uploading: $gallery"
    aws s3 sync "$dir" "s3://izaakmaine-assets/$gallery/" \
        --endpoint-url https://fly.storage.tigris.dev \
        --profile tigris
done
```

### 5.2 Upload a Single Gallery

```bash
# Example: upload the magellan gallery
aws s3 sync static/images/magellan/ s3://izaakmaine-assets/magellan/ \
    --endpoint-url https://fly.storage.tigris.dev \
    --profile tigris
```

### 5.3 Verify Upload

```bash
# List files in a gallery
aws s3 ls s3://izaakmaine-assets/magellan/ \
    --endpoint-url https://fly.storage.tigris.dev \
    --profile tigris
```

### 5.4 Verify Public Access

Open a browser and navigate to:
```
https://izaakmaine-assets.t3.storage.dev/magellan/<some-image>.jpg
```

The image should load. If it returns a 403, check that the bucket is configured for public access in the Tigris dashboard.

### Already Uploaded ✅

| Gallery | Status | Example URL |
|---------|--------|-------------|
| `manalo` | ✅ Uploaded | `https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png` |
| `columbus-pitch` | ⬜ Pending | — |
| `elcano` | ⬜ Pending | — |
| `luther` | ⬜ Pending | — |
| `magellan` | ⬜ Pending | — |
| `paul` | ⬜ Pending | — |
| `valladolid` | ⬜ Pending | — |

---

## 6. Solving the `readDir` Problem — Image Manifests

### The Core Challenge

Hugo's `readDir` function only reads from the **local filesystem** at build time. It **cannot** query an S3 bucket to discover which files exist. We need an alternative mechanism to tell Hugo which images belong to each slideshow.

### The Solution: Hugo Data Files

Hugo supports [Data Files](https://gohugo.io/templates/data/) — JSON, YAML, or TOML files placed in the `data/` directory that are accessible via `site.Data` in templates.

We create one JSON file per gallery in `data/slideshows/`, listing the image filenames in order.

### 6.1 Directory Structure

```
data/
  slideshows/
    columbus-pitch.json
    elcano.json
    luther.json
    magellan.json
    manalo.json
    paul.json
    valladolid.json
```

### 6.2 Example Manifest: `data/slideshows/manalo.json`

Based on the actual files in the `manalo` gallery:

```json
[
  "manalo_a.png",
  "manalo_b.png",
  "manalo_c.png",
  "manalo_d.png",
  "manalo_e.png",
  "manalo_f.png",
  "manalo_g.png",
  "manalo_h.png",
  "manalo_i.png",
  "manalo_j.png",
  "manalo_k.png",
  "manalo_l.png"
]
```

> **Important**: The filenames must exactly match the files uploaded to S3.

### 6.3 Generating Manifests Automatically

Create a helper script to generate these manifests from your local `static/images/` directories **before** you remove them from the repo.

#### `scripts/generate_manifests.sh`

```bash
#!/usr/bin/env bash
# Generate Hugo data files from static/images subdirectories

MANIFEST_DIR="data/slideshows"
IMAGE_DIR="static/images"

mkdir -p "$MANIFEST_DIR"

for dir in "$IMAGE_DIR"/*/; do
    gallery_name=$(basename "$dir")
    manifest_file="$MANIFEST_DIR/${gallery_name}.json"
    
    echo "Generating manifest for: $gallery_name"
    
    # Find image files, sort, and format as JSON array
    files=$(find "$dir" -maxdepth 1 -type f \
        \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" -o -iname "*.webp" \) \
        -printf '%f\n' | sort)
    
    # Build JSON array
    echo "[" > "$manifest_file"
    first=true
    while IFS= read -r file; do
        [ -z "$file" ] && continue
        if [ "$first" = true ]; then
            echo "  \"$file\"" >> "$manifest_file"
            first=false
        else
            echo "  ,\"$file\"" >> "$manifest_file"
        fi
    done <<< "$files"
    echo "]" >> "$manifest_file"
    
    echo "  -> $manifest_file ($(echo "$files" | grep -c .) images)"
done

echo "Done! Manifests written to $MANIFEST_DIR/"
```

Run it:
```bash
chmod +x scripts/generate_manifests.sh
./scripts/generate_manifests.sh
```

#### Alternative: Generate from S3 Directly

If images are already on S3 and removed locally, you can generate manifests from the bucket:

```bash
#!/usr/bin/env bash
MANIFEST_DIR="data/slideshows"
BUCKET="izaakmaine-assets"
ENDPOINT="https://fly.storage.tigris.dev"

mkdir -p "$MANIFEST_DIR"

# List top-level "directories" in the bucket
galleries=$(aws s3 ls "s3://$BUCKET/" \
    --endpoint-url "$ENDPOINT" --profile tigris \
    | awk '/PRE/ {print $2}' | sed 's|/||')

for gallery in $galleries; do
    manifest_file="$MANIFEST_DIR/${gallery}.json"
    
    files=$(aws s3 ls "s3://$BUCKET/$gallery/" \
        --endpoint-url "$ENDPOINT" --profile tigris \
        | awk '{print $4}' \
        | grep -iE '\.(jpg|jpeg|png|webp)$' \
        | sort)
    
    echo "[" > "$manifest_file"
    first=true
    while IFS= read -r file; do
        [ -z "$file" ] && continue
        if [ "$first" = true ]; then
            echo "  \"$file\"" >> "$manifest_file"
            first=false
        else
            echo "  ,\"$file\"" >> "$manifest_file"
        fi
    done <<< "$files"
    echo "]" >> "$manifest_file"
    
    echo "Generated: $manifest_file"
done
```

---

## 7. Hugo Configuration Changes

Add CDN parameters to `hugo.yaml` so the shortcode knows where to find images:

```yaml
params:
  # ... existing params ...

  cdn:
    enabled: true
    url: "https://izaakmaine-assets.t3.storage.dev"
```

> [!TIP]
> Set `cdn.enabled: false` during local development to fall back to local images. This dual-mode approach lets you work offline.

### Resulting Image URLs

With this configuration, the shortcode will construct URLs like:

```
<cdn.url>/<gallery>/<filename>
= https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png  ✅ matches working example
```

---

## 8. Shortcode Modification

### Current Code (Key Lines)

```go-html-template
{{ $path := .Get "path" }}
{{ $files := readDir (print "static/" $path) }}
{{ range $files }}
  <img src="/{{ $path }}/{{ .Name }}" ...>
{{ end }}
```

### New Code (Dual-Mode)

The updated shortcode supports two modes:

| Mode | Condition | Image Source |
|------|-----------|-------------|
| **S3/CDN** | `params.cdn.enabled == true` AND manifest exists in `data/slideshows/` | Remote URL from Tigris CDN |
| **Local** | Otherwise | Local `static/` directory via `readDir` |

```go-html-template
{{ $seed := delimit (shuffle (seq 1 9)) "" }}
{{ $id := cond (isset .Params "id") (.Get "id") (substr (md5 $seed) 0 8) }}
{{ $path := .Get "path" }}

{{ $cdnEnabled := site.Params.cdn.enabled | default false }}
{{ $cdnUrl := site.Params.cdn.url | default "" }}
{{ $images := slice }}

{{/* Derive gallery key from path (e.g., "images/magellan" -> "magellan") */}}
{{ $galleryKey := path.Base $path }}

{{/* Check for a data manifest */}}
{{ $manifest := index site.Data.slideshows $galleryKey }}

{{ if and $cdnEnabled $manifest }}
  {{/* === S3/CDN MODE === */}}
  {{/* URL: <cdn.url>/<gallery>/<filename> */}}
  {{ range $manifest }}
    {{ $imgUrl := printf "%s/%s/%s" $cdnUrl $galleryKey . }}
    {{ $images = $images | append (dict "Name" . "URL" $imgUrl) }}
  {{ end }}
{{ else }}
  {{/* === LOCAL MODE (fallback) === */}}
  {{ $files := readDir (print "static/" $path) }}
  {{ range $files }}
    {{ if or (strings.HasSuffix .Name ".jpg") (strings.HasSuffix .Name ".jpeg")
            (strings.HasSuffix .Name ".png") (strings.HasSuffix .Name ".webp") }}
      {{ $images = $images | append (dict "Name" .Name "URL" (printf "/%s/%s" $path .Name)) }}
    {{ end }}
  {{ end }}
{{ end }}

{{/* Render slides */}}
<div class="slideshow-container" id="slideshow-{{ $id }}">
    <div class="slides-wrapper">
        {{ range $index, $img := $images }}
        <div class="slide">
            <img src="{{ $img.URL }}" alt="{{ $img.Name }}" onclick="toggleFullscreen('{{ $id }}')">
            <div class="caption">{{ $index | add 1 }} / {{ len $images }}</div>
        </div>
        {{ end }}
    </div>
    {{/* ... navigation buttons, dots, styles, and script remain unchanged ... */}}
</div>
```

### Example Output (CDN Mode)

For `{{< slideshow path="images/manalo" >}}` with `cdn.enabled: true`:

```html
<img src="https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png" alt="manalo_a.png">
<img src="https://izaakmaine-assets.t3.storage.dev/manalo/manalo_b.png" alt="manalo_b.png">
...
```

> [!NOTE]
> The rest of the shortcode (navigation arrows, dots, CSS, JavaScript) stays **exactly the same**. Only the image source logic changes.

---

## 9. Content File Changes

**None required.** The content files continue to use the same shortcode call:

```markdown
{{< slideshow path="images/manalo" >}}
```

The shortcode internally decides whether to use local or CDN images based on the Hugo config and the presence of a manifest file.

---

## 10. Removing Images from the Repository

Once you have confirmed that the S3/CDN setup is working correctly:

### 10.1 Add to `.gitignore`

```gitignore
# Images are now hosted on Tigris S3
static/images/
```

### 10.2 Remove from Git Tracking

```bash
# Remove from Git index (not from disk)
git rm -r --cached static/images/

# Commit
git commit -m "chore: move images to Tigris S3, remove from repo"
```

### 10.3 (Optional) Purge from Git History

If you want to reduce the repository size retroactively:

```bash
# Using git-filter-repo (recommended over filter-branch)
pip install git-filter-repo
git filter-repo --path static/images/ --invert-paths
```

> [!WARNING]
> This rewrites Git history. Coordinate with collaborators before doing this.

---

## 11. Ongoing Workflow

After the migration, follow this workflow when adding new slideshows or images:

```
┌──────────────────────────────────────────────────────────────┐
│  1. Prepare images locally                                   │
│     └── Place in static/images/<gallery-name>/               │
│                                                              │
│  2. Upload to S3 (note: no images/ prefix in S3 key)         │
│     └── aws s3 sync static/images/<gallery-name>/            │
│         s3://izaakmaine-assets/<gallery-name>/               │
│         --endpoint-url https://fly.storage.tigris.dev        │
│         --profile tigris                                     │
│                                                              │
│  3. Generate or update manifest                              │
│     └── ./scripts/generate_manifests.sh                      │
│     └── OR manually create data/slideshows/<name>.json       │
│                                                              │
│  4. Add content (shortcode call stays the same)              │
│     └── {{< slideshow path="images/<gallery-name>" >}}       │
│                                                              │
│  5. Test locally                                             │
│     └── hugo server (with cdn.enabled: false for local)      │
│                                                              │
│  6. Deploy                                                   │
│     └── git add data/slideshows/ content/                    │
│     └── git push                                             │
└──────────────────────────────────────────────────────────────┘
```

---

## 12. Rollback Plan

If something goes wrong, you can revert to local images instantly:

1. Set `cdn.enabled: false` in `hugo.yaml`.
2. Ensure `static/images/` still exists locally (or restore from S3).
3. Rebuild. The shortcode will fall back to `readDir` and local files.

No content file changes are needed for rollback.

---

## Appendix: Full Modified Shortcode

The complete updated shortcode combines:

- The **new image-source logic** (S3 vs local) at the top — see [Section 8](#8-shortcode-modification).
- The **existing HTML structure** (slides, navigation, dots) — unchanged.
- The **existing CSS** — unchanged.
- The **existing JavaScript** — unchanged.

Only the image discovery logic changes. Everything else remains identical.

---

## Appendix: Quick Reference

| Item | Value |
|------|-------|
| **Bucket Name** | `izaakmaine-assets` |
| **S3 API Endpoint** (for `aws s3` commands) | `https://fly.storage.tigris.dev` |
| **Public CDN URL** (for `<img>` src) | `https://izaakmaine-assets.t3.storage.dev` |
| **S3 Key Pattern** | `<gallery>/<filename>` |
| **Public URL Pattern** | `https://izaakmaine-assets.t3.storage.dev/<gallery>/<filename>` |
| **Hugo Config Key** | `params.cdn.url: "https://izaakmaine-assets.t3.storage.dev"` |
| **Data Manifest Location** | `data/slideshows/<gallery>.json` |
