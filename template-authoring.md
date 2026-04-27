# Posterama Template Authoring Guide

Templates are YAML files that drive ImageMagick to produce styled poster images. The engine normalises the base poster to **1000×1500 px**, then runs each step in order, passing the resulting argument list to ImageMagick.

---

## Table of Contents

1. [Directory layout](#1-directory-layout)
2. [template.yml structure](#2-templateyml-structure)
3. [Context variables reference](#3-context-variables-reference)
4. [Expressions and interpolation](#4-expressions-and-interpolation)
5. [Transform reference](#5-transform-reference)
6. [Variables block](#6-variables-block)
7. [Steps](#7-steps)
8. [Conditions](#8-conditions)
9. [skipBaseNormalization](#9-skipbasenormalization)
10. [Resources folder](#10-resources-folder)
11. [Complete worked examples](#11-complete-worked-examples)
12. [Tips and common patterns](#12-tips-and-common-patterns)
13. [Validation errors](#13-validation-errors)

---

## 1. Directory layout

### How template repos work

Posterama's **Repositories** page lets you add any Git repository URL. When you sync it, Posterama does a shallow clone and scans the repo for templates. There are two supported structures:

---

#### Option A — Single-template repo

One `template.yml` lives at the root of the repository. The entire repo is one template.

```
your-repo/                    ← GitHub repository root
  template.yml                ← required
  resources/                  ← optional
    fonts/
      MyFont-Bold.ttf
    overlays/
      gradient.png
  README.md                   ← ignored by Posterama, useful for humans
```

**Template ID after sync:** `repos/your-github-username/your-repo`

This is the recommended structure when publishing a single polished template.

---

#### Option B — Multi-template repo

Multiple templates live in named sub-directories. Posterama detects each one independently.

```
your-repo/                    ← GitHub repository root
  TemplateOne/
    template.yml              ← required per template
    resources/
      overlays/
        gradient.png
  TemplateTwo/
    template.yml
    resources/
      fonts/
        Bold.ttf
  README.md
```

**Template IDs after sync:**
- `repos/your-github-username/your-repo/TemplateOne`
- `repos/your-github-username/your-repo/TemplateTwo`

Use this structure to group thematic variants (e.g. Light, Dark, Minimal) in one repo.

---

#### Local templates (no Git)

You can also drop templates directly into the Posterama config volume at `/config/templates/local/<name>/template.yml`. No repo registration needed.

```
/config/templates/local/
  MyCustomTemplate/
    template.yml
    resources/
```

**Template ID:** `local/MyCustomTemplate`

---

### Notes on syncing

- Posterama performs a **shallow clone** (`--depth 1`) on first add, then `git pull` on subsequent syncs.
- Any Git hosting works: GitHub, GitLab, Gitea, self-hosted, etc.
- Private repos are supported as long as the URL includes credentials or the host is accessible without authentication.
- Sync can be triggered manually from the Repositories page or runs on the configured schedule.
- Templates from a synced repo can be **copied to local** from the Templates page, letting you customise them without affecting the upstream repo.

---

## 2. template.yml structure

```yaml
# ── Required ────────────────────────────────────────────────────
name: "Human-readable template name"

# ── Optional metadata ───────────────────────────────────────────
description: "Short description shown in the UI"
version: "1.0"
author: "your-github-handle"

# ── Skip the automatic 1000×1500 normalisation step ─────────────
# Default: false — the engine resizes and crops the base poster
# before running any steps. Set to true only if your first step
# handles the geometry itself.
skipBaseNormalization: false

# ── Variables (computed before any step runs) ───────────────────
variables:
  myVar:
    type: expr
    value: "'prefix-' + title|lower"

# ── Steps (run in order) ─────────────────────────────────────────
steps:
  - name: "unique-step-name"
    condition: "isTv"          # optional — step runs only when truthy
    args:
      - "-argument"
      - "{{myVar}}"
```

All string values inside `args` support `{{expression}}` interpolation.

---

## 3. Context variables reference

These are available inside every expression, condition, and `{{…}}` interpolation.

### Engine-injected (always present)

| Variable | Type | Description |
|---|---|---|
| `logoPath` | `string \| null` | Absolute path to the selected logo file |
| `hasLogo` | `boolean` | `true` when a logo was selected |
| `posterWidth` | `number` | Always `1000` |
| `posterHeight` | `number` | Always `1500` |
| `templateResources` | `string` | Absolute path to `<templateDir>/resources` |

### Media metadata

| Variable | Type | Description |
|---|---|---|
| `title` | `string \| null` | Movie / show / collection title |
| `originalTitle` | `string \| null` | Original language title |
| `year` | `number \| null` | Release year |
| `overview` | `string \| null` | Plot summary |
| `tagline` | `string \| null` | Tagline |
| `voteAverage` | `number \| null` | TMDB rating (e.g. `8.73`) |
| `voteAverageRounded` | `number \| null` | Rounded to 1 decimal (e.g. `8.7`) |
| `runtime` | `number \| null` | Runtime in minutes |
| `certification` | `string \| null` | Age rating string (e.g. `"PG-13"`) |
| `imdbId` | `string \| null` | IMDb ID |
| `tmdbId` | `number \| null` | TMDB ID |
| `mediaType` | `"movie" \| "tv" \| null` | |
| `status` | `string \| null` | e.g. `"Returning Series"`, `"Ended"`, `"Canceled"`, `"In Production"` |
| `originalLanguage` | `string \| null` | ISO 639-1 code e.g. `"en"` |
| `popularity` | `number \| null` | TMDB popularity score |
| `firstAirDate` | `string \| null` | ISO date string e.g. `"2022-06-23"` |
| `lastAirDate` | `string \| null` | ISO date string |
| `posterPath` | `string \| null` | TMDB poster path |
| `backdropPath` | `string \| null` | TMDB backdrop path |

### Derived / computed

| Variable | Type | Description |
|---|---|---|
| `isTv` | `boolean` | `true` when mediaType is `"tv"` |
| `isMovie` | `boolean` | `true` when mediaType is `"movie"` |
| `isCollection` | `boolean` | `true` for Plex collections |
| `isEnded` | `boolean` | `status === "Ended"` |
| `isReturning` | `boolean` | `status === "Returning Series"` |
| `isCanceled` | `boolean` | `status === "Canceled"` |
| `isInProduction` | `boolean` | in-production flag from TMDB |
| `inProduction` | `boolean` | same as `isInProduction` |
| `isSeason` | `boolean` | `true` when a season number was provided |
| `seasonNumber` | `number \| null` | The selected season number |
| `seasonName` | `string \| null` | Season name from TMDB |
| `seasonEpisodeCount` | `number \| null` | Episodes in the season |
| `seasonCount` | `number \| null` | Total seasons (`numberOfSeasons`) |
| `episodeCount` | `number \| null` | Total episodes (`numberOfEpisodes`) |
| `collectionMemberCount` | `number \| null` | Items in a Plex collection |

### Lists

| Variable | Type | Description |
|---|---|---|
| `genreList` | `string[]` | Ordered genre names e.g. `["Action", "Drama"]` |
| `genre1` | `string \| null` | First genre |
| `genre2` | `string \| null` | Second genre |
| `networkName` | `string \| null` | First broadcast network name |
| `seasonList` | `{seasonNumber, name, episodeCount}[]` | All seasons |

---

## 4. Expressions and interpolation

### `{{expression}}` in args

Any step arg can contain one or more `{{expr}}` placeholders. The result is converted to a string. `null` becomes `""`.

```yaml
args:
  - "-annotate"
  - "+0+100"
  - "{{title}}"                     # → "Inception"
  - "{{voteAverageRounded}}/10"     # → "8.7/10"
  - "{{genre1|default('Unknown')}}" # → "Action" or "Unknown"
```

Multiple placeholders in one string are all replaced:

```yaml
  - "{{title}} ({{year}})"  # → "The Bear (2022)"
```

### Conditions

Step `condition` fields are full Jexl expressions returning a truthy/falsy value:

```yaml
condition: "isTv"
condition: "isMovie and voteAverage > 8"
condition: "isSeason and seasonNumber == 1"
condition: "hasLogo and not isCollection"
condition: "status == 'Ended' or status == 'Canceled'"
```

### Expression syntax (Jexl)

| Syntax | Example |
|---|---|
| Comparison | `voteAverage >= 8.0` |
| Logical | `isTv and not isEnded` |
| Ternary | `isTv ? 'TV' : 'Movie'` |
| String concat | `'★ ' + voteAverageRounded` |
| Array access | `genreList[0]` |
| Null check | `tagline != null` |

---

## 5. Transform reference

Transforms are applied to a value using the pipe `|` syntax inside expressions.

```yaml
"{{title|upper}}"           # → "THE BEAR"
"{{title|lower}}"           # → "the bear"
"{{title|truncate(20)}}"    # → "The Marvelous Mrs. M…"
"{{title|trim}}"            # → trims whitespace
```

| Transform | Input | Output | Notes |
|---|---|---|---|
| `upper` | string | string | Uppercase |
| `lower` | string | string | Lowercase |
| `trim` | string | string | Strip leading/trailing whitespace |
| `truncate(n)` | string | string | Truncate to `n` chars, appends `…` |
| `round` | number | number | `Math.round` |
| `floor` | number | number | `Math.floor` |
| `ceil` | number | number | `Math.ceil` |
| `abs` | number | number | `Math.abs` |
| `default(x)` | any | any | Returns `x` when value is `null`, `undefined`, or `""` |
| `year` | string | string | Extracts first 4 chars: `"2022-06-23"` → `"2022"` |
| `join(sep)` | string[] | string | Joins array: `genreList\|join(' / ')` → `"Action / Drama"` |
| `words` | number | string | `2` → `"two"` |
| `ordinal` | number | string | `2` → `"2nd"` |
| `wordsOrdinal` | number | string | `2` → `"second"` |

Transforms chain:

```yaml
"{{title|trim|upper|truncate(25)}}"
"{{seasonNumber|wordsOrdinal|upper}}"   # → "SECOND"
```

---

## 6. Variables block

Variables are resolved **before any step runs** and injected into the context. Later variables can reference earlier ones.

### `type: expr` — compute a value

```yaml
variables:
  ratingLabel:
    type: expr
    value: "'★ ' + voteAverageRounded|default('N/A')"

  seasonLabel:
    type: expr
    value: "isSeason ? 'Season ' + seasonNumber : ''"
```

### `type: map` — map a context value to a string

```yaml
variables:
  statusColor:
    type: map
    source: "status"         # expression whose result is used as the lookup key
    mappings:
      "Returning Series": "#22c55e"
      "Ended":            "#ef4444"
      "Canceled":         "#6b7280"
      "In Production":    "#f59e0b"
    default: "#ffffff"       # used when key is absent or source fails

  overlayFile:
    type: map
    source: "status"
    mappings:
      "Returning Series": "border-green.png"
      "Ended":            "border-red.png"
    default: "border-white.png"
    prefix: "{{templateResources}}/overlays/"
    # → "/config/templates/local/MyTemplate/resources/overlays/border-green.png"
```

`prefix` is itself interpolated, so you can use `{{templateResources}}` in it.

If the `source` expression throws or the key is not found in `mappings`, `default` is used. An explicit empty-string mapping **is** valid and will not fall through to `default`.

---

## 7. Steps

Each step is an ImageMagick argument fragment. The engine concatenates all enabled step fragments and calls:

```
magick <base-args> <step1-args> <step2-args> ... -quality 95 -strip <outPath>
```

### Referencing resource files

```yaml
args:
  - "{{templateResources}}/overlays/gradient.png"
  - "-composite"
```

### Using the logo

```yaml
- name: "add-logo"
  condition: "hasLogo"
  args:
    - "("
    - "{{logoPath}}"
    - "-resize"
    - "700x300"      # fit inside 700×300, preserve ratio
    - ")"
    - "-gravity"
    - "south"
    - "-geometry"
    - "+0+175"
    - "-composite"
```

### Drawing text

```yaml
- name: "draw-title"
  condition: "title != null"
  args:
    - "-font"
    - "{{templateResources}}/fonts/MyFont-Bold.ttf"
    - "-pointsize"
    - "60"
    - "-fill"
    - "white"
    - "-gravity"
    - "south"
    - "-annotate"
    - "+0+200"
    - "{{title|truncate(30)}}"
```

### Fonts — bundled vs system

You can reference fonts two ways:

**Bundled font** (recommended — portable, no host dependency):
```yaml
- "-font"
- "{{templateResources}}/fonts/MyFont-Bold.ttf"
```
Place the `.ttf` or `.otf` file in `resources/fonts/` and commit it to the repo.

**System font** (depends on what ImageMagick can find on the host):
```yaml
- "-font"
- "Helvetica"
```
Available system fonts vary by installation. Use `magick -list font` inside the container to see what's available. Bundled fonts are strongly preferred for shareable templates.

### Gravity reference

| Value | Anchor |
|---|---|
| `center` | Middle of image |
| `north` | Top centre |
| `south` | Bottom centre |
| `east` | Middle-right |
| `west` | Middle-left |
| `northeast` | Top-right corner |
| `northwest` | Top-left corner |
| `southeast` | Bottom-right corner |
| `southwest` | Bottom-left corner |

### `-geometry` offsets

`+X+Y` — positive X moves right, positive Y moves **down** from the anchor point (for `south`, Y increases upward from the bottom).

### YAML quoting rules for args

Each item in `args` is a YAML block sequence scalar. A few characters need care:

| Situation | Rule | Example |
|---|---|---|
| `(` or `)` alone | Quote or leave bare — both work | `"("` or `(` |
| `%` in level strings | Quote the value | `"-40%,85%"` |
| `#` starting a value | **Must** quote — bare `#` starts a YAML comment | `"#ffffff"` |
| `{{expr}}` interpolation | No quoting needed; engine handles it | `"{{title}}"` |
| Spaces inside a value | Quote to be safe | `"rectangle 4,4 995,1495"` |

When in doubt, double-quote the argument. Excess quotes never cause problems.

---

## 8. Conditions

Steps without a `condition` always run. Steps with a condition run only when the expression is truthy.

```yaml
# Conditional examples
condition: "hasLogo"
condition: "isTv and not isSeason"
condition: "isSeason and seasonNumber != null"
condition: "isCollection"
condition: "isEnded or isCanceled"
condition: "genre1 != null"
condition: "voteAverage >= 8"
condition: "collectionMemberCount > 1"
```

The user can also **force-enable or force-disable** individual steps from the editor UI using step overrides. Name your steps descriptively — the name is shown in the UI.

---

## 9. skipBaseNormalization

By default the engine prepends these args before your steps:

```
<posterPath> -resize 1000x1500^ -gravity center -extent 1000x1500 +repage
```

This crops and fills the source image to exactly 1000×1500. Set `skipBaseNormalization: true` only if:

- Your template fully controls the canvas geometry (e.g. you're compositing from scratch)
- The base image is already 1000×1500

When set to `true`, the `<posterPath>` argument is **not** prepended — your first step must open the image.

---

## 10. Resources folder

Place any static assets in `resources/`. The `{{templateResources}}` variable always points to this folder, so your YAML is portable regardless of install path.

Recommended layout:

```
resources/
  fonts/
    MyFont-Bold.ttf
  overlays/
    gradient.png
    collection-badge.png
    border-green.png
    border-red.png
```

> **File formats:** ImageMagick reads PNG, JPG, TIFF, and most other raster formats. PNG with transparency is recommended for overlays.

### Template thumbnail

To display a preview image in the Posterama Templates gallery, place a PNG at:

```
resources/
  metadata/
    thumb.png    ← shown as the template card thumbnail
```

If absent, the card shows a placeholder. Recommended size: **300×450 px** (2:3 ratio matching a poster).

---

## 11. Complete worked examples

### Example A — Minimal template (no logo, no text)

```yaml
name: "Dark Vignette"
description: "Simple darkening vignette effect"
version: "1.0"

steps:
  - name: "darken"
    args:
      - "-level"
      - "-30%,90%"

  - name: "vignette"
    args:
      - "-vignette"
      - "0x50"
```

---

### Example B — Gradient overlay with logo

```yaml
name: "DarkMatte Style"
description: "Dark gradient with logo placement"
version: "1.1"
author: "alsharad"

steps:
  - name: "darken-base"
    args:
      - "-level"
      - "-54%,80%"
      - "-colorspace"
      - "RGB"

  - name: "gradient-overlay"
    args:
      - "{{templateResources}}/overlays/gradient.png"
      - "-composite"

  - name: "collection-badge"
    condition: "isCollection"
    args:
      - "{{templateResources}}/overlays/collection.png"
      - "-composite"

  - name: "add-logo"
    condition: "hasLogo"
    args:
      - "("
      - "{{logoPath}}"
      - "-resize"
      - "600x260"
      - ")"
      - "-gravity"
      - "south"
      - "-geometry"
      - "+0+175"
      - "-composite"
```

---

### Example C — Status-based coloured border

```yaml
name: "Status Border"
description: "Coloured border changes colour by show status"
version: "1.0"

variables:
  borderColor:
    type: map
    source: "status"
    mappings:
      "Returning Series": "#22c55e"
      "Ended":            "#ef4444"
      "Canceled":         "#6b7280"
    default: "#94a3b8"

steps:
  - name: "gradient-overlay"
    args:
      - "{{templateResources}}/overlays/gradient.png"
      - "-composite"

  - name: "status-border"
    args:
      - "-fill"
      - "none"
      - "-stroke"
      - "{{borderColor}}"
      - "-strokewidth"
      - "8"
      - "-draw"
      - "rectangle 4,4 995,1495"

  - name: "add-logo"
    condition: "hasLogo"
    args:
      - "("
      - "{{logoPath}}"
      - "-resize"
      - "600x260"
      - ")"
      - "-gravity"
      - "south"
      - "-geometry"
      - "+0+175"
      - "-composite"
```

---

### Example D — Season label with ordinal text

```yaml
name: "Season Label"
description: "Adds season number as text below the logo"
version: "1.0"

variables:
  seasonLabel:
    type: expr
    value: "isSeason ? seasonNumber|wordsOrdinal|upper + ' SEASON' : ''"

steps:
  - name: "gradient-overlay"
    args:
      - "{{templateResources}}/overlays/gradient.png"
      - "-composite"

  - name: "add-logo"
    condition: "hasLogo"
    args:
      - "("
      - "{{logoPath}}"
      - "-resize"
      - "600x220"
      - ")"
      - "-gravity"
      - "south"
      - "-geometry"
      - "+0+225"
      - "-composite"

  - name: "season-text"
    condition: "isSeason and seasonLabel != ''"
    args:
      - "-font"
      - "{{templateResources}}/fonts/CoreSansA65Bold.otf"
      - "-pointsize"
      - "38"
      - "-fill"
      - "white"
      - "-stroke"
      - "none"
      - "-gravity"
      - "south"
      - "-annotate"
      - "+0+155"
      - "{{seasonLabel}}"
```

---

### Example E — Genre pill composite

```yaml
name: "Genre Pill"
description: "Shows first two genres as a label strip"
version: "1.0"

variables:
  genreText:
    type: expr
    value: "genre2 != null ? genre1 + '  ·  ' + genre2 : genre1|default('')"

steps:
  - name: "darken"
    args:
      - "-level"
      - "-40%,85%"

  - name: "gradient"
    args:
      - "{{templateResources}}/overlays/gradient.png"
      - "-composite"

  - name: "add-logo"
    condition: "hasLogo"
    args:
      - "("
      - "{{logoPath}}"
      - "-resize"
      - "600x220"
      - ")"
      - "-gravity"
      - "south"
      - "-geometry"
      - "+0+215"
      - "-composite"

  - name: "genre-label"
    condition: "genre1 != null"
    args:
      - "-font"
      - "{{templateResources}}/fonts/Light.ttf"
      - "-pointsize"
      - "28"
      - "-fill"
      - "rgba(255,255,255,0.75)"
      - "-gravity"
      - "south"
      - "-annotate"
      - "+0+155"
      - "{{genreText|upper}}"
```

---

### Example F — Rating badge overlay

```yaml
name: "Rating Badge"
description: "Composites a rating badge from a computed path"
version: "1.0"

variables:
  ratingTier:
    type: map
    source: "voteAverage >= 8 ? 'high' : voteAverage >= 6 ? 'mid' : 'low'"
    mappings:
      "high": "badge-gold.png"
      "mid":  "badge-silver.png"
      "low":  "badge-grey.png"
    default: "badge-grey.png"
    prefix: "{{templateResources}}/badges/"

steps:
  - name: "darken"
    args:
      - "-level"
      - "-40%,85%"

  - name: "gradient"
    args:
      - "{{templateResources}}/overlays/gradient.png"
      - "-composite"

  - name: "rating-badge"
    condition: "voteAverage != null"
    args:
      - "("
      - "{{ratingTier}}"
      - "-resize"
      - "120x120"
      - ")"
      - "-gravity"
      - "northeast"
      - "-geometry"
      - "+20+20"
      - "-composite"

  - name: "rating-number"
    condition: "voteAverage != null"
    args:
      - "-font"
      - "{{templateResources}}/fonts/Bold.ttf"
      - "-pointsize"
      - "32"
      - "-fill"
      - "white"
      - "-gravity"
      - "northeast"
      - "-annotate"
      - "+30+44"
      - "{{voteAverageRounded}}"

  - name: "add-logo"
    condition: "hasLogo"
    args:
      - "("
      - "{{logoPath}}"
      - "-resize"
      - "600x260"
      - ")"
      - "-gravity"
      - "south"
      - "-geometry"
      - "+0+175"
      - "-composite"
```

---

## 12. Tips and common patterns

### Transparent overlays

PNGs with an alpha channel composite cleanly:

```yaml
args:
  - "{{templateResources}}/overlays/gradient.png"
  - "-composite"
```

### Multi-line annotations

ImageMagick supports `\n` in annotate strings. Use single quotes in YAML to preserve it:

```yaml
args:
  - "-annotate"
  - "+0+100"
  - '{{title}}\n{{year}}'
```

### Coloured text shadow

```yaml
- name: "title-shadow"
  args:
    - "-fill"
    - "rgba(0,0,0,0.6)"
    - "-annotate"
    - "+2+102"
    - "{{title}}"
- name: "title"
  args:
    - "-fill"
    - "white"
    - "-annotate"
    - "+0+100"
    - "{{title}}"
```

### Conditional logo sizing

```yaml
variables:
  logoSize:
    type: expr
    value: "isCollection ? '500x200' : '600x260'"

# In the logo step:
args:
  - "("
  - "{{logoPath}}"
  - "-resize"
  - "{{logoSize}}"
  - ")"
  ...
```

### Safe chaining of nullable values

```yaml
variables:
  displayTitle:
    type: expr
    value: "title|default(originalTitle)|default('Unknown')"
```

### Year in corner

```yaml
- name: "year-label"
  condition: "firstAirDate != null"
  args:
    - "-pointsize"
    - "26"
    - "-fill"
    - "rgba(255,255,255,0.6)"
    - "-gravity"
    - "northeast"
    - "-annotate"
    - "+20+20"
    - "{{firstAirDate|year}}"
```

### Avoid upscaling logos

Use `>` flag to only shrink, never enlarge:

```yaml
args:
  - "-resize"
  - "600x260>"
```

### Step naming convention

Name steps for what they do, not what they are. The name is shown in the UI when users override steps:

```
✓ add-logo
✓ status-border
✓ season-label
✗ step1
✗ composite
```

---

> **Testing:** After authoring a template, sync its repo (or place it in `local/`) and use the poster editor to generate a preview before saving. The step override controls in the UI let you toggle individual steps on or off to debug.

---

## 13. Validation errors

Posterama validates every `template.yml` at load time using a strict schema. When a template fails validation it still appears in the Templates list, but with an error badge. The error message is shown in the card tooltip.

### Common errors and fixes

| Error | Cause | Fix |
|---|---|---|
| `Required` on `name` | Missing `name` field | Add `name: "My Template"` at the top |
| `Step names must be unique` | Two steps share the same `name` | Rename one of the duplicate steps |
| `Expected array, received string` on `args` | Wrote `args: "-resize 100"` as a string | Use a YAML list: `args:` then `- "-resize"` / `- "100"` |
| `Invalid discriminator value` on a variable | `type` is not `expr` or `map` | Check spelling: only `type: expr` and `type: map` are valid |
| `Template not found` | `template.yml` is missing or path is wrong | Confirm the file is at the root (Option A) or inside the named sub-directory (Option B) |
| YAML parse error | Unquoted `#` colour value, bad indentation, or tab characters | Quote colour strings (`"#ffffff"`); use spaces not tabs |

If your template silently produces a blank or unexpected output with no error, the most likely cause is a step condition that never evaluates to `true`, or an expression that returns `null` (which becomes an empty string in an arg). Use the step override panel in the editor to force-enable individual steps and isolate the issue.
