# Migrating Images to Tigris Data (S3) & Updating the Slideshow — V3

> **Objective**: Move all images out of the GitHub repository into Tigris Data (S3-compatible object storage with built-in global CDN) and update the Hugo slideshow shortcode to serve images from the remote URL instead of the local `static/` directory.
>
> **Version**: V3 — Clarified all directory paths and script execution context relative to the Hugo project root.

---

## Table of Contents

1. [Project Directory Map](#1-project-directory-map)
2. [Current Architecture](#2-current-architecture)
3. [Why Tigris Data?](#3-why-tigris-data)
4. [Confirmed Tigris URL Pattern](#4-confirmed-tigris-url-pattern)
5. [Setting Up Tigris Data](#5-setting-up-tigris-data)
6. [Uploading Images](#6-uploading-images)
7. [Solving the `readDir` Problem — Image Manifests](#7-solving-the-readdir-problem--image-manifests)
8. [Hugo Configuration Changes](#8-hugo-configuration-changes)
9. [Shortcode Modification](#9-shortcode-modification)
10. [Content File Changes](#10-content-file-changes)
11. [Removing Images from the Repository](#11-removing-images-from-the-repository)
12. [Ongoing Workflow](#12-ongoing-workflow)
13. [Rollback Plan](#13-rollback-plan)
14. [Quick Reference](#quick-reference)

---

## 1. Project Directory Map

All paths in this document are relative to the **Hugo project root**:

```
/home/chaschel/Documents/hugo/izaakmaine.github.io-main/
```

Here is the full directory layout showing **existing** files and **new** files/folders that this migration will create:

```
izaakmaine.github.io-main/          ← PROJECT ROOT (you must cd here to run commands)
│
├── hugo.yaml                       ← Hugo config (we add cdn params here)
├── go.mod
├── go.sum
├── .gitignore                      ← We add static/images/ here later
│
├── content/                        ← Markdown content (no changes needed)
│   └── explorer/
│       ├── 01-columbus-pitch.md    ← uses {{< slideshow path="images/columbus-pitch" >}}
│       ├── 02-magellan.md          ← uses {{< slideshow path="images/magellan" >}}
│       └── ...
│
├── layouts/
│   └── shortcodes/
│       └── slideshow.html          ← MODIFY: update image source logic
│
├── static/
│   ├── images/                     ← CURRENT image storage (will be removed from git)
│   │   ├── columbus-pitch/
│   │   ├── elcano/
│   │   ├── luther/
│   │   ├── magellan/
│   │   ├── manalo/                 ← ✅ Already uploaded to Tigris
│   │   │   ├── manalo_a.png
│   │   │   ├── manalo_b.png
│   │   │   └── ... (12 files)
│   │   ├── paul/
│   │   └── valladolid/
│   └── js/
│
├── data/                           ← 🆕 CREATE THIS FOLDER (Hugo Data Files)
│   └── slideshows/                 ← 🆕 CREATE THIS SUBFOLDER
│       ├── columbus-pitch.json     ← 🆕 one manifest per gallery
│       ├── elcano.json
│       ├── luther.json
│       ├── magellan.json
│       ├── manalo.json
│       ├── paul.json
│       └── valladolid.json
│
├── scripts/                        ← Already exists
│   └── generate_manifests.sh       ← Already exists (needs to run from PROJECT ROOT)
│
├── docs/                           ← Documentation
│   ├── DOCS_S3.MD
│   ├── DOCS_S3_OPUS46.MD
│   └── DOCS_S3_OPUS46_V2.MD
│
└── public/                         ← Hugo build output (auto-generated)
```

> [!IMPORTANT]
> ### Where does `data/slideshows/` go?
>
> The `data/` folder goes **directly inside the project root**, at the same level as `content/`, `layouts/`, `static/`, and `hugo.yaml`. This is a [standard Hugo directory](https://gohugo.io/getting-started/directory-structure/). Hugo automatically reads JSON/YAML/TOML files from `data/` and makes them available in templates via `site.Data`.
>
> **Absolute path**: `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/data/slideshows/`
>
> **Create it with**:
> ```bash
> mkdir -p /home/chaschel/Documents/hugo/izaakmaine.github.io-main/data/slideshows
> ```

> [!IMPORTANT]
> ### Where do I run `scripts/generate_manifests.sh`?
>
> You **must** run the script from the **project root** directory, because the script uses **relative paths** (`data/slideshows` and `static/images`). If you run it from any other directory, it will create folders in the wrong location.
>
> ```bash
> # Step 1: cd to the project root
> cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main
>
> # Step 2: Run the script (from the project root!)
> bash scripts/generate_manifests.sh
> ```
>
> **What happens when you run it:**
> - The script scans each subfolder inside `static/images/` (e.g., `static/images/manalo/`).
> - For each subfolder, it finds all `.jpg`, `.jpeg`, `.png`, `.webp` files.
> - It writes a sorted JSON array of filenames to `data/slideshows/<gallery>.json`.
>
> **Example**: Running from the project root, the script reads `static/images/manalo/` and creates `data/slideshows/manalo.json` — which ends up at the absolute path:
> ```
> /home/chaschel/Documents/hugo/izaakmaine.github.io-main/data/slideshows/manalo.json
> ```

> [!CAUTION]
> ### Common Mistake: Running from Wrong Directory
>
> The existing script was previously run from inside `scripts/`, which created `scripts/data/slideshows/` instead of the intended `data/slideshows/`. That location is **wrong** — Hugo does not read data files from `scripts/data/`.
>
> **Wrong** (run from `scripts/`):
> ```
> cd scripts/
> bash generate_manifests.sh
> # Creates: scripts/data/slideshows/manalo.json  ← Hugo can't see this!
> ```
>
> **Correct** (run from project root):
> ```
> cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main
> bash scripts/generate_manifests.sh
> # Creates: data/slideshows/manalo.json  ← Hugo reads this via site.Data.slideshows.manalo
> ```

---

## 2. Current Architecture

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

### The Problem

- Images bloat the Git repository (binary files don't diff well).
- GitHub Pages has limited bandwidth/CDN capabilities compared to dedicated object storage.
- Every `git clone` downloads every image ever committed.

---

## 3. Why Tigris Data?

[Tigris Data](https://www.tigrisdata.com/) is an S3-compatible, globally distributed object storage service with:

- **Built-in CDN**: Objects are cached at edge locations automatically — no separate CDN setup.
- **S3 API Compatibility**: Works with AWS CLI, SDKs, and any S3-compatible tool.
- **Public Buckets**: Files can be made publicly accessible via HTTP URLs.
- **Free Tier**: Generous free tier suitable for static site assets.

---

## 4. Confirmed Tigris URL Pattern

Based on a confirmed working upload, the public URL pattern is:

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
3. **The public CDN domain is `t3.storage.dev`** — Different from the S3 API endpoint (`fly.storage.tigris.dev`) used for `aws s3` commands.

### URL Mapping (Local → S3)

| Local Path (in project) | S3 Key | Public URL |
|--------------------------|--------|------------|
| `static/images/manalo/manalo_a.png` | `manalo/manalo_a.png` | `https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png` |
| `static/images/magellan/001.jpg` | `magellan/001.jpg` | `https://izaakmaine-assets.t3.storage.dev/magellan/001.jpg` |

---

## 5. Setting Up Tigris Data

### 5.1 Create an Account

Sign up at [console.tigris.dev](https://console.tigris.dev/).

### 5.2 Create a Bucket

Create a bucket (e.g., `izaakmaine-assets`). Enable **public access**.

### 5.3 Obtain Credentials

| Credential | Description |
|------------|-------------|
| `Access Key ID` | Your AWS-style access key |
| `Secret Access Key` | Your AWS-style secret key |
| `S3 API Endpoint` | `https://fly.storage.tigris.dev` (for `aws s3` commands) |
| `Public CDN Domain` | `https://izaakmaine-assets.t3.storage.dev` (for browser/img tags) |

> [!NOTE]
> The **S3 API endpoint** (`fly.storage.tigris.dev`) is for CLI operations. The **public CDN domain** (`<bucket>.t3.storage.dev`) is what goes in `<img src="...">` tags and `hugo.yaml`.

### 5.4 Configure AWS CLI

```bash
aws configure --profile tigris
# AWS Access Key ID: <your_access_key>
# AWS Secret Access Key: <your_secret_key>
# Default region name: auto
# Default output format: json
```

### 5.5 Verify Access

```bash
aws s3 ls s3://izaakmaine-assets/ --endpoint-url https://fly.storage.tigris.dev --profile tigris
```

---

## 6. Uploading Images

All upload commands should be run from the **project root**:

```bash
cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main
```

### 6.1 Upload ALL Galleries at Once

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

### 6.2 Upload a Single Gallery

```bash
aws s3 sync static/images/magellan/ s3://izaakmaine-assets/magellan/ \
    --endpoint-url https://fly.storage.tigris.dev \
    --profile tigris
```

### 6.3 Verify Public Access

```
https://izaakmaine-assets.t3.storage.dev/magellan/<some-image>.jpg
```

### Upload Progress

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

## 7. Solving the `readDir` Problem — Image Manifests

### The Core Challenge

Hugo's `readDir` function only reads from the **local filesystem** at build time. It **cannot** query an S3 bucket. We need to tell Hugo which images belong to each slideshow via a **data file**.

### The Solution: Hugo Data Files

Hugo reads JSON/YAML/TOML files from the `data/` directory and makes them available in templates via `site.Data`. We create one JSON file per gallery:

### 7.1 Create the Directory

```bash
# Run from the project root
cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main
mkdir -p data/slideshows
```

This creates:
```
izaakmaine.github.io-main/
├── data/                  ← same level as content/, layouts/, hugo.yaml
│   └── slideshows/
```

### 7.2 Example Manifest: `data/slideshows/manalo.json`

**Absolute path**: `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/data/slideshows/manalo.json`

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

Hugo will make this accessible as `site.Data.slideshows.manalo` in templates — returning the array above.

### 7.3 Generate All Manifests with the Script

The script `scripts/generate_manifests.sh` already exists in the project. It scans `static/images/` and creates one JSON file per gallery inside `data/slideshows/`.

**How to run it:**

```bash
# STEP 1: cd to the project root — THIS IS CRITICAL
cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main

# STEP 2: Make the script executable (one time only)
chmod +x scripts/generate_manifests.sh

# STEP 3: Run it
bash scripts/generate_manifests.sh
```

**What it does:**

```
Reads from:     static/images/manalo/          (looks for .jpg .jpeg .png .webp)
                static/images/magellan/
                static/images/elcano/
                ... (all subdirectories)

Writes to:      data/slideshows/manalo.json    (sorted list of filenames)
                data/slideshows/magellan.json
                data/slideshows/elcano.json
                ... (one JSON per gallery)
```

**Expected output:**

```
Generating manifest for: columbus-pitch
  -> data/slideshows/columbus-pitch.json (N images)
Generating manifest for: elcano
  -> data/slideshows/elcano.json (N images)
Generating manifest for: luther
  -> data/slideshows/luther.json (N images)
Generating manifest for: magellan
  -> data/slideshows/magellan.json (N images)
Generating manifest for: manalo
  -> data/slideshows/manalo.json (12 images)
Generating manifest for: paul
  -> data/slideshows/paul.json (N images)
Generating manifest for: valladolid
  -> data/slideshows/valladolid.json (N images)
Done! Manifests written to data/slideshows/
```

**After running**, verify the files were created in the right place:

```bash
ls -la data/slideshows/
# Should show: columbus-pitch.json  elcano.json  luther.json  magellan.json  manalo.json  paul.json  valladolid.json

cat data/slideshows/manalo.json
# Should show the JSON array of filenames
```

### 7.4 Alternative: Generate from S3 Directly

If images are already removed from local and only exist on S3:

```bash
#!/usr/bin/env bash
# Run from PROJECT ROOT: cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main

MANIFEST_DIR="data/slideshows"
BUCKET="izaakmaine-assets"
ENDPOINT="https://fly.storage.tigris.dev"

mkdir -p "$MANIFEST_DIR"

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

## 8. Hugo Configuration Changes

**File**: `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/hugo.yaml`

Add CDN parameters under `params`:

```yaml
params:
  # ... existing params (search, theme, page, navbar, footer) ...

  cdn:
    enabled: true
    url: "https://izaakmaine-assets.t3.storage.dev"
```

> [!TIP]
> Set `cdn.enabled: false` during local development to fall back to local images. This lets you work offline without S3 access.

### Resulting Image URLs

With this config, the shortcode constructs:

```
<cdn.url>/<gallery>/<filename>
= https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png  ✅
```

---

## 9. Shortcode Modification

**File**: `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/layouts/shortcodes/slideshow.html`

### Current Code (Key Lines)

```go-html-template
{{ $path := .Get "path" }}
{{ $files := readDir (print "static/" $path) }}
{{ range $files }}
  <img src="/{{ $path }}/{{ .Name }}" ...>
{{ end }}
```

### New Code (Dual-Mode)

| Mode | Condition | Image Source |
|------|-----------|-------------|
| **S3/CDN** | `params.cdn.enabled == true` AND manifest exists | Remote URL from Tigris CDN |
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

{{/* Check for a data manifest: site.Data.slideshows.<galleryKey> */}}
{{ $manifest := index site.Data.slideshows $galleryKey }}

{{ if and $cdnEnabled $manifest }}
  {{/* === S3/CDN MODE === */}}
  {{/* URL: https://izaakmaine-assets.t3.storage.dev/<gallery>/<filename> */}}
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

### Example Output (CDN Mode for Manalo)

```html
<img src="https://izaakmaine-assets.t3.storage.dev/manalo/manalo_a.png" alt="manalo_a.png">
<img src="https://izaakmaine-assets.t3.storage.dev/manalo/manalo_b.png" alt="manalo_b.png">
...
```

> [!NOTE]
> The rest of the shortcode (navigation arrows, dots, CSS, JavaScript) stays **exactly the same**. Only the image source logic changes.

---

## 10. Content File Changes

**None required.** Content files keep the same shortcode call:

```markdown
{{< slideshow path="images/manalo" >}}
```

The shortcode derives the gallery key (`manalo`) from the path and looks it up in `site.Data.slideshows.manalo`.

---

## 11. Removing Images from the Repository

Once you have confirmed that the S3/CDN setup is working correctly:

### 11.1 Add to `.gitignore`

**File**: `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/.gitignore`

```gitignore
# Images are now hosted on Tigris S3
static/images/
```

### 11.2 Remove from Git Tracking

```bash
cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main

# Remove from Git index (not from disk)
git rm -r --cached static/images/

# Commit
git commit -m "chore: move images to Tigris S3, remove from repo"
```

### 11.3 (Optional) Purge from Git History

```bash
pip install git-filter-repo
git filter-repo --path static/images/ --invert-paths
```

> [!WARNING]
> This rewrites Git history. Coordinate with collaborators before doing this.

---

## 12. Ongoing Workflow

All commands below assume you are in the **project root**:
```bash
cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main
```

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
│  3. Generate or update manifest (RUN FROM PROJECT ROOT!)     │
│     └── bash scripts/generate_manifests.sh                   │
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

## 13. Rollback Plan

To revert to local images instantly:

1. Set `cdn.enabled: false` in `hugo.yaml`.
2. Ensure `static/images/` still exists locally (or restore from S3).
3. Rebuild. The shortcode falls back to `readDir` and local files.

No content file changes needed.

---

## Quick Reference

| Item | Value |
|------|-------|
| **Project Root** | `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/` |
| **Manifest Location** | `data/slideshows/<gallery>.json` (relative to project root) |
| **Manifest Absolute Path** | `/home/chaschel/Documents/hugo/izaakmaine.github.io-main/data/slideshows/` |
| **Script Location** | `scripts/generate_manifests.sh` (relative to project root) |
| **Run Script From** | **Project root** (`cd /home/chaschel/Documents/hugo/izaakmaine.github.io-main`) |
| **Bucket Name** | `izaakmaine-assets` |
| **S3 API Endpoint** (CLI) | `https://fly.storage.tigris.dev` |
| **Public CDN URL** (img src) | `https://izaakmaine-assets.t3.storage.dev` |
| **S3 Key Pattern** | `<gallery>/<filename>` |
| **Hugo Config Key** | `params.cdn.url: "https://izaakmaine-assets.t3.storage.dev"` |
| **Hugo Template Access** | `site.Data.slideshows.<gallery>` |
