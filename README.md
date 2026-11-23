# webprint

**Turn a website into structured data: its sections, its layouts and its routes.**

webprint opens a page in a real browser and describes what a visitor sees. It splits the page into its
major sections (header, hero, pricing, footer…), explains why it placed each boundary, and exports each
section's content, position and screenshot as JSON. A second tool maps a whole site into route families
and page layouts, so you can capture each layout once instead of every page.

> **Status: experimental.** This is a working research tool, not a production library. The heuristics
> are tuned on a handful of real sites and test fixtures, the output formats may still change, and there
> is no stable Python API yet; use it through the command line. Known gaps are listed in
> [`notice.md`](notice.md).

## What it does

| Tool | Input | Output |
|---|---|---|
| **`section-scan`** | One URL | The page's sections at laptop, tablet and mobile sizes: boundaries with evidence, content as a tree, one screenshot per section, an HTML report |
| **`section-scan classify`** | A captured page | A label per section (`hero`, `pricing`, `faq`, …) and for the page (industry, page type), proposed by an AI model and confirmed by you |
| **`section-scan board`** | Captured pages | One Excalidraw board with every page side by side, per breakpoint, sections annotated |
| **`route-map`** | A site's home URL | Its route families (`/product/{product_slug}` instead of 1,000 URLs), page roles and distinct layouts |

## Quick start

Requires Python 3.12+, [uv](https://docs.astral.sh/uv/) and a Chromium build for Playwright.

```bash
git clone https://github.com/aminesayagh/webprint.git && cd webprint
uv sync                                  # install dependencies into .venv
uv run playwright install chromium       # one-time browser download

uv run section-scan capture https://example.com -o runs/example   # laptop, tablet and mobile
uv run section-scan detect runs/example
uv run section-scan report runs/example --open
```

Then review and export one breakpoint's sections, or map a whole site:

```bash
uv run section-scan review runs/example/laptop   # interactive: rename, merge, split, confirm
uv run section-scan export runs/example/laptop -o sections.json
uv run route-map https://example.com -o route-map.json --summary route-map.md
```

Only `capture` and `route-map` visit the site. Everything else works offline on the saved run, so you can
rerun detection with different settings without loading the page again.

## Example output

`report.json` describes each section in page order:

```json
{
  "link": "https://example.com/",
  "title": "Page title",
  "sections": [
    {
      "id": "pricing",
      "role": "flow",
      "position": { "x": 0, "y": 900, "width": 1440, "height": 982 },
      "message": [
        { "tag": "group", "children": [ { "tag": "h2", "text": "Pricing" }, { "tag": "p", "text": "Pick one of these plans." } ] },
        { "tag": "ul", "children": [ { "tag": "li", "text": "Starter" }, { "tag": "li", "text": "Pro" } ] }
      ]
    }
  ]
}
```

## How it works

1. **Capture:** Playwright loads the page at each breakpoint and waits for fonts, images and lazy content.
   It records every rendered element (box, styles, text, DOM path) together with a full-page screenshot
   and an offline copy of the page.
2. **Detect:** the elements are simplified into visual groups. Candidate boundaries come from containers,
   landmarks, spacing gaps and background changes, and each boundary is marked split, kept or ambiguous,
   with the evidence behind that decision.
3. **Review:** nothing is confirmed automatically. You merge, split, rename and confirm sections, and
   every step can be undone.
4. **Merge breakpoints:** content is matched across the three loads (same text first, then same DOM
   slot), so each section knows where it sits, and what it hides, at each size.
5. **Map the site (route-map):** a polite crawler (it obeys robots.txt, is throttled and stays within a
   budget) reads sitemaps and links, groups URLs into families, and checks sampled pages to confirm that a
   family really shares one layout.

## Documentation

- [`docs/section-scan.md`](docs/section-scan.md): capture, detection, review, breakpoints, screenshots,
  the board, classification, `report.json` and tuning (`detector.toml`).
- [`docs/route-map.md`](docs/route-map.md): discovery, route families, validation, page roles, templates
  and the output schema.
- [`evaluation/README.md`](evaluation/README.md): how the detector and route-map are measured.
- [`notice.md`](notice.md): known issues and postponed decisions.

## Project layout

```
src/section_scan/   page capture, section detection, review, report, classification, board
src/route_map/      site crawling, route families, validation, roles, templates
tests/              pytest suite (uses a real Chromium; about 2 minutes)
fixtures/           small HTML pages the tests capture
evaluation/         annotations and scripts that score the detector against baselines
detector.toml       detector settings (optional; defaults apply without it)
```

Run the tests with `uv run pytest`.

## Configuration

- **Classification** uses TypeSafe's Jev model through `typesafe-sdk`. Put `TYPESAFE_API_KEY=…` in a
  `.env` file (git-ignored). Every other command works without it.
- **Detection** reads `./detector.toml` if present. The keys and their defaults are listed in
  [`docs/section-scan.md`](docs/section-scan.md#tuning-detectortoml).

## Limits

- Vertically flowing pages work best. App shells, canvas-heavy pages and scroll-driven animations are
  only partly understood, and areas the tool cannot inspect are recorded as limitations rather than
  reported as empty.
- iframes, closed shadow roots and canvas pixels are not inspected.
- route-map reads served HTML and does not render JavaScript, so a client-rendered site shows up as a
  possible JavaScript shell.

## Roadmap

- **Change tracking:** keep a snapshot per visit and report what changed between two visits, and in
  which section.
- Stable element references across captures, so reviews and labels carry over to a recapture.
