# Migrating Images to Tigris Data (S3-Compatible) & Updating the Slideshow

> **Objective**: Move all images out of the GitHub repository into Tigris Data (an S3-compatible object storage with built-in CDN) and update the Hugo slideshow shortcode to serve images from the remote URL instead of the local `static/` directory.

---

## Table of Contents

1. [Current Architecture](#1-current-architecture)
2. [Why Tigris Data?](#2-why-tigris-data)
3. [Setting Up Tigris Data](#3-setting-up-tigris-data)
4. [Uploading Images](#4-uploading-images)
5. [Solving the `readDir` Problem — Image Manifests](#5-solving-the-readdir-problem--image-manifests)
6. [Hugo Configuration Changes](#6-hugo-configuration-changes)
7. [Shortcode Modification](#7-shortcode-modification)
8. [Content File Changes](#8-content-file-changes)
9. [Removing Images from the Repository](#9-removing-images-from-the-repository)
10. [Ongoing Workflow](#10-ongoing-workflow)
11. [Rollback Plan](#11-rollback-plan)
12. [Appendix: Full Modified Shortcode](#appendix-full-modified-shortcode)

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

| Directory | Purpose |
|-----------|---------|
| `static/images/columbus-pitch` | Columbus Pitch slideshow |
| `static/images/elcano` | Elcano slideshow |
| `static/images/luther` | Luther slideshow |
| `static/images/magellan` | Magellan slideshow |
| `static/images/manalo` | Manalo slideshow |
| `static/images/paul` | Paul slideshow |
| `static/images/valladolid` | Valladolid slideshow |

### The Problem

- Images bloat the Git repository (binary files don't diff well).
- GitHub Pages has limited bandwidth/CDN capabilities compared to dedicated object storage.
- Every `git clone` downloads every image ever committed.

---

## 2. Why Tigris Data?

[Tigris Data](https://www.tigrisdata.com/) is an S3-compatible, globally distributed object storage service with:

- **Built-in CDN**: Objects are cached at edge locations automatically — no separate CDN configuration needed.
- **S3 API Compatibility**: Works with the AWS CLI, SDKs, and any S3-compatible tool.
- **Public Buckets**: Files can be made publicly accessible via HTTP URLs.
- **Free Tier**: Generous free tier suitable for static site assets.

The public URL of an object typically follows this pattern:
```
https://fly.storage.tigris.dev/<bucket-name>/<key>
```

---

## 3. Setting Up Tigris Data

### 3.1 Create an Account

Sign up at [console.tigris.dev](https://console.tigris.dev/).

### 3.2 Create a Bucket

Create a bucket (e.g., `izaakmaine-assets`). Enable **public access** so images can be loaded by browsers without authentication.

### 3.3 Obtain Credentials

From the Tigris dashboard, obtain:

| Credential | Description |
|------------|-------------|
| `Access Key ID` | Your AWS-style access key |
| `Secret Access Key` | Your AWS-style secret key |
| `Endpoint URL` | `https://fly.storage.tigris.dev` |
| `Region` | `auto` (or as specified by Tigris) |

### 3.4 Configure AWS CLI

```bash
# Install AWS CLI (if not already installed)
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

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

### 3.5 Verify Access

```bash
aws s3 ls --endpoint-url https://fly.storage.tigris.dev --profile tigris
```

You should see your bucket listed.

---

## 4. Uploading Images

### 4.1 Mirror the Directory Structure

Upload images preserving the existing folder structure so that image paths remain consistent:

```bash
# Sync ALL image directories at once
aws s3 sync static/images/ s3://izaakmaine-assets/images/ \
  --endpoint-url https://fly.storage.tigris.dev \
  --profile tigris \
  --acl public-read
```

### 4.2 Upload a Single Gallery

```bash
# Sync only the magellan gallery
aws s3 sync static/images/magellan/ s3://izaakmaine-assets/images/magellan/ \
  --endpoint-url https://fly.storage.tigris.dev \
  --profile tigris \
  --acl public-read
```

### 4.3 Verify Upload

```bash
# List uploaded files
aws s3 ls s3://izaakmaine-assets/images/magellan/ \
  --endpoint-url https://fly.storage.tigris.dev \
  --profile tigris
```

### 4.4 Verify Public Access

Open a browser and navigate to:
```
https://fly.storage.tigris.dev/izaakmaine-assets/images/magellan/<some-image>.jpg
```

The image should load. If it returns a 403, double-check that `--acl public-read` was applied or that the bucket has a public access policy.

---

## 5. Solving the `readDir` Problem — Image Manifests

### The Core Challenge

Hugo's `readDir` function only reads from the **local filesystem** at build time. It **cannot** query an S3 bucket to discover which files exist. Therefore, we need an alternative mechanism to tell Hugo which images belong to each slideshow.

### The Solution: Hugo Data Files

Hugo supports [Data Files](https://gohugo.io/templates/data/) — JSON, YAML, or TOML files placed in the `data/` directory that are accessible via `site.Data` in templates.

We create one JSON file per gallery in `data/slideshows/`, listing the image filenames in order.

### 5.1 Directory Structure

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

### 5.2 Example Manifest: `data/slideshows/magellan.json`

```json
[
  "001-magellan-portrait.jpg",
  "002-ship-trinidad.jpg",
  "003-strait-of-magellan.jpg",
  "004-pacific-ocean.jpg",
  "005-philippines.jpg"
]
```

> **Important**: The filenames must exactly match the files uploaded to S3.

### 5.3 Generating Manifests Automatically

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

If images are already on S3 and removed from local, you can generate manifests from the bucket:

```bash
#!/usr/bin/env bash
MANIFEST_DIR="data/slideshows"
BUCKET="izaakmaine-assets"
PREFIX="images"
ENDPOINT="https://fly.storage.tigris.dev"

mkdir -p "$MANIFEST_DIR"

# List top-level "directories" in the images prefix
galleries=$(aws s3 ls "s3://$BUCKET/$PREFIX/" \
    --endpoint-url "$ENDPOINT" --profile tigris \
    | awk '/PRE/ {print $2}' | sed 's|/||')

for gallery in $galleries; do
    manifest_file="$MANIFEST_DIR/${gallery}.json"
    
    files=$(aws s3 ls "s3://$BUCKET/$PREFIX/$gallery/" \
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

## 6. Hugo Configuration Changes

Add CDN parameters to `hugo.yaml` so the shortcode knows where to find images:

```yaml
params:
  # ... existing params ...

  cdn:
    enabled: true
    url: "https://fly.storage.tigris.dev/izaakmaine-assets"
```

> **Tip**: Set `cdn.enabled: false` during local development to fall back to local images. This dual-mode approach lets you work offline.

---

## 7. Shortcode Modification

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
| **S3/CDN** | `params.cdn.enabled == true` AND a manifest file exists in `data/slideshows/` | Remote URL from Tigris |
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
  {{ range $manifest }}
    {{ $imgUrl := printf "%s/%s/%s" $cdnUrl $path . }}
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

> **Key Point**: The rest of the shortcode (navigation, dots, CSS, JavaScript) stays exactly the same. Only the image source logic changes.

---

## 8. Content File Changes

**None required.** The content files continue to use the same shortcode call:

```markdown
{{< slideshow path="images/magellan" >}}
```

The shortcode internally decides whether to use local or CDN images based on the Hugo config and the presence of a manifest file.

---

## 9. Removing Images from the Repository

Once you have confirmed that the S3/CDN setup is working correctly:

### 9.1 Add to `.gitignore`

```gitignore
# Images are now hosted on Tigris S3
static/images/
```

### 9.2 Remove from Git Tracking

```bash
# Remove from Git index (not from disk)
git rm -r --cached static/images/

# Commit
git commit -m "chore: move images to Tigris S3, remove from repo"
```

### 9.3 (Optional) Purge from Git History

If you want to reduce the repository size retroactively:

```bash
# Using git-filter-repo (recommended over filter-branch)
pip install git-filter-repo
git filter-repo --path static/images/ --invert-paths
```

> ⚠️ **Warning**: This rewrites Git history. Coordinate with collaborators before doing this.

---

## 10. Ongoing Workflow

After the migration, follow this workflow when adding new slideshows or images:

```
┌──────────────────────────────────────────────────────────┐
│  1. Prepare images locally                               │
│     └── Place in static/images/<gallery-name>/           │
│                                                          │
│  2. Upload to S3                                         │
│     └── aws s3 sync static/images/<gallery-name>/        │
│         s3://izaakmaine-assets/images/<gallery-name>/    │
│         --endpoint-url https://fly.storage.tigris.dev    │
│         --profile tigris --acl public-read               │
│                                                          │
│  3. Generate or update manifest                          │
│     └── ./scripts/generate_manifests.sh                  │
│     └── OR manually create data/slideshows/<name>.json   │
│                                                          │
│  4. Add content                                          │
│     └── {{< slideshow path="images/<gallery-name>" >}}   │
│                                                          │
│  5. Test locally                                         │
│     └── hugo server (with cdn.enabled: false for local)  │
│                                                          │
│  6. Deploy                                               │
│     └── git add data/slideshows/ content/                │
│     └── git push                                         │
└──────────────────────────────────────────────────────────┘
```

---

## 11. Rollback Plan

If something goes wrong, you can revert to local images instantly:

1. Set `cdn.enabled: false` in `hugo.yaml`.
2. Ensure `static/images/` still exists locally (or restore from S3).
3. Rebuild. The shortcode will fall back to `readDir` and local files.

No content file changes are needed for rollback.

---

## Appendix: Full Modified Shortcode

See the [Full shortcode diff in Step 7](#7-shortcode-modification). The complete file would combine:

- The **new image-source logic** (S3 vs local) at the top.
- The **existing HTML structure** (slides, navigation, dots) — unchanged.
- The **existing CSS** — unchanged.
- The **existing JavaScript** — unchanged.

The only lines that change are **lines 1–11** of the original `slideshow.html` (the image discovery logic). Everything else remains identical.
