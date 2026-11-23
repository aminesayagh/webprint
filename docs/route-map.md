# route-map: a site's route families

A second command in this repo. It turns one website into a compact map of **route families**: 1,000
product pages become one route, `/product/{segment_2}`, instead of 1,000 entries. Single pages such as
`/about` stay on their own. It infers routes from the URLs it sees. It does not read the site's source
router and does not prove that every value works. Its output is not an SEO XML sitemap.

The plan is `vault/AI-Memory/02_projects/quickshop/sitemap-route-generator-plan.md`. All six milestones
are built: discovery from the seed URL, route families, their validation on sampled pages, page roles, the
export views, and the plan's test scenarios with measurements (see `evaluation/README.md`, "route-map").

```bash
uv run route-map https://shop.example/ -o route-map.json --summary route-map.md   # or: python -m route_map ...
uv run route-map https://shop.example/ --inventory urls.txt -o route-map.json     # a URL list, nothing fetched
```

**Worked example** (masayagh.com, default budgets; `--summary` output):

| Route | Status | Role | URLs | Samples |
| --- | --- | --- | --- | --- |
| `/contact/?service={service}` | validated_on_sample | contact | 6 | 6 |
| `/` | individual | home | 1 | 1 |
| `/contact/` | individual | contact | 1 | 1 |

The six "Start a project" links (`?service=Product discovery`, `?service=Dashboards`, ...) are one
family: their pages share a template and carry a contact form. The map took 12 requests.

- **Discovery:** the crawler (Scrapy) obeys robots.txt and reads it for `Sitemap:` declarations. If there
  are none, it tries `/sitemap.xml` and `/sitemap_index.xml`, then relies on links.
  - **Sitemaps:** indexes are followed recursively, gzipped files included. Limits stop runaway files:
    a seen set (cycles), `--max-sitemap-files`, 50 MB per file after decompression, and safe XML parsing
    (no entity expansion). **Sitemap URLs only go into the inventory**: 1,000 products listed in a sitemap
    cost no page requests.
  - **Pages:** HTML pages are fetched from the seed outward. Half of `--max-html-pages` goes to discovery;
    the rest goes to validation samples, then to template sampling (see Templates). Pages are fetched in waves: a wave is chosen only once every
    page in flight has answered, so the map does not depend on which response comes back first. A wave
    takes one waiting page per path shape not fetched yet (one product page is enough to show the product
    family, so the budget reaches the blog, pages and so on), then a few more members when every shape is
    known. Ties go to the shallower page, then the URL in sorted order.
  - **Redirects:** they are recorded, not followed blindly. The old URL keeps a `redirect_to`, the target
    is queued like any link, and a redirect never takes the crawl to another host. `--max-depth` bounds how far from the seed links are followed.
  - **Never requested:** forms, action links (logout, add to cart, delete, ...), binary assets (images,
    PDFs, fonts, ...), other hosts (listed in `skipped_hosts`; `--include-subdomains` widens this), and
    pages disallowed by robots.txt (`blocked_urls`).
  - **Throttling:** requests are throttled (Scrapy AutoThrottle, at most 4 at a time), time out after 20 s
    and are retried twice, except page requests: a retry would be a page outside `--max-html-pages`. `--deadline-seconds` (300 by default) bounds the whole discovery.
  - **Partial maps:** a budget that runs out yields a partial map, with the reason in
    `crawl_report.stop_reasons`. A seed page without links in its HTML is reported as a possible
    JavaScript shell; pages are not rendered in this version.
- **Input:** `--inventory` is a text file with one URL per line (`#` lines are skipped). URLs on another
  host than the seed, or that are not HTTP(S), are listed under `crawl_report.rejected_urls`.
- **URL identity (conservative):** the scheme, default ports, ordinary `#anchors` and tracking parameters
  (`utm_*`, `gclid`, `fbclid`, ...) are ignored. Everything else is kept as written: path case, `%2F`,
  extensions, the trailing slash, and unknown query parameters with their order, blank values and repeated
  keys. Duplicates become `aliases` of one URL record.
- **Families:** URLs of the same shape (host, depth, trailing slash, extension, query keys) that differ in
  exactly one slot become a family once that slot takes at least 3 different values, e.g.
  `/blog/{segment_2}`, `/p/{segment_2}.html` or `/search?q={q}`. When a URL fits two candidates, a varying
  path segment wins over a query value and the rightmost segment over earlier ones (URLs vary most at their
  end); then the larger family wins.
  - **Two parameters:** one-parameter families that differ only in one other path segment, at least 3 of
    them, merge into one family. For example, `/collections/men/products/{x}`, `/collections/women/products/{x}`
    and so on become `/collections/{segment_2}/products/{segment_4}`. URLs of the same shape left out of
    every family (a collection with one product) join it. Samples are spread over the outer value (one per
    collection first). Language prefixes never merge (`/en/...`, `/fr/...` stay separate), and the merged
    pattern must keep a constant path segment.
  - **Rules that hold:**
    - `/blog/{slug}` and `/blog/category/{slug}` are different families.
    - Locale prefixes stay separate (`/en/product/...`, `/fr/product/...`).
    - Root pages (`/about`, `/contact`, `/privacy`) are never folded into a catch-all `/{segment_1}`.
- **Validation on samples:** once discovery has nothing left to fetch, each family's planned samples are
  fetched. The plan spreads them over different value shapes (`item-12` and `compare` are different
  shapes), and families take turns so a 1,000-product family cannot use up the budget. Pages fetched
  during discovery count as samples for free. A family whose samples disagree gets up to 3 more.
  - **What is compared:** each page's main content (`<main>`, `[role=main]` or `<article>`; otherwise the
    body without header, nav, footer and aside, marked as weaker evidence) as a set of tag paths such as
    `ul>li>a`. Texts, prices, generated ids and how many cards repeat do not count; a different component
    does. Pages whose JSON-LD names different things (Product vs Article) never agree. Signals read outside
    the main content (`og:type` or article dates in `<head>`, a search box in the header) do not count.
  - **Agreement:** two pages agree when their structure overlaps by at least 50% (Jaccard). A page joins a
    group only when it agrees with every member, so a page halfway between two unlike pages does not chain
    them together. Pages are taken in URL path order, so the groups do not change with the host or port.
  - **Unusable samples:** samples that failed, were blocked, redirected elsewhere, or read as an error,
    sign-in or challenge page are set aside (`unusable_sample` evidence). They never count against their
    family.
- **Status of each route:**
  - `validated_on_sample`: at least 3 usable samples agree. Samples that differ become their own
    `individual` routes (`/product/compare`), listed in the family's `excluded_literals` and
    `exception_route_ids`. Unsampled members stay inferred (`inferred_unsampled_url_count`), not verified.
  - **Two templates under one pattern** (3+ samples each): one route per template (`template: "a"`, `"b"`).
    The unsampled members' URLs cannot tell which template they use, so they are `ambiguous`, with their
    `candidate_route_ids`.
  - `unresolved`: enough usable samples, but no 3 agree. The observed pages become individual routes; the
    rest stay in the unresolved family.
  - `provisional`: too few usable samples, for example when the page budget ran out (said in
    `stop_reasons`), or with `--inventory`, which fetches nothing.
  - `individual`: a URL in no family.
- **Evidence:** each validated route lists what its samples share: `main_structure` (e.g. `5/5`), the
  JSON-LD type, functional signals (`purchase`, `listing`, `article_meta`, `search`, `password`,
  `contact_form`), and a `canonical` that points elsewhere.
- **Page roles:** each route gets a `role` (`home`, `product_detail`, `product_listing`, `article`,
  `article_listing`, `search_results`, `contact`, `authentication`, `cart`, `checkout`, `informational`
  or `unknown`) and a `role_certainty`. The rules are small and listed in `src/route_map/roles.py`; the
  reason is kept as `role` evidence.
  - **`supported`:** at least 80% of the usable samples show the role's cues. For example, a product page
    has Product JSON-LD and a purchase control; a product listing has a grid whose links reach product
    pages; a login page has a password field in its main content (a sign-in box in the header does not
    count).
  - **`weak`:** cues on at least half the samples, or only words in the URL (`/products/...`,
    `/blog/...`, `/cart`).
  - **`unknown`:** no cue, an unresolved family, or two supported roles that contradict each other (the
    reason is given). Two overlaps are settled on purpose: a product page with a related-products grid
    stays a product page, and a search page listing products is `search_results`. An exception split off
    a family (`/product/compare`) does not take a role from its URL words: they describe the family.
- **Parameter names:** parameters are renamed only when the route's role is `supported`. Otherwise they
  keep their neutral names (`{segment_2}`).
  - **The last path parameter** is named after the role: `{product_slug}` or `{product_id}`,
    `{article_slug}` or `{article_id}`, `{category_slug}` for listings.
  - **An earlier one** is named after the word before it: `{category_slug}` after `/collections/` or
    `/category/`, as in `/collections/{category_slug}/products/{product_slug}`.
- **Output:** versioned JSON (`schema_version`) holding:
  - `routes`, each with at most 3 `examples`;
  - `urls`: the full inventory, each with its `route_id` (or `assignment: "ambiguous"` / `"unresolved"`),
    its discovery `sources` and any `redirect_to`;
  - `observations`: the pages fetched, each with its status, final URL, title, links, structure features,
    JSON-LD types, breadcrumbs, signals and canonical (a 200 that reads as an error or a challenge page is
    marked `unsuitable`);
  - `tree`: the URL prefix hierarchy. A node without a route of its own (`/product` when only
    `/product/{product_slug}` exists) is `synthetic`: a grouping, not a discovered page;
  - `links`: route-to-route links actually seen on fetched pages (`from`, `to`, how many links, from how
    many pages), kept apart from the prefix tree, so no category-to-product relation is invented;
  - `crawl_report`: budgets, requests used, sitemap results, skipped/blocked/failed URLs, stop reasons and
    limitations.
- **Summary:** `--summary FILE` also writes a Markdown table of the routes (status, role, URL and sample
  counts, examples), their exceptions, and why the map is partial if it is. Its IDs and ordering depend only on the set of URLs, not on their order.
- **Limits (heuristics, see also `notice.md`):**
  - **At most two parameters per family:** a second one comes only from merging sibling families
    (below), and three or more are not inferred. A sibling family with fewer than 3 members stays apart
    unless its URLs fit a merged family.
  - **No rendering:** pages are read as served HTML. A JavaScript shell (a seed page without links) is
    reported, not rendered, and hash-router URLs are flagged, not followed.
  - **Word lists:** action links are skipped by URL words, so `/product/remove-tool` would be skipped too.
    Error-page titles, purchase and checkout buttons and the role URL words cover English plus a few
    French and German phrases.
  - **No pagination naming:** a numeric segment keeps a neutral name. There is no pagination evidence
    (such as `rel=next`) yet.
  - **Structure threshold:** 50% overlap of main-content structure decides whether two pages share a
    template. It was tuned on fixtures, not proven on every site.

## Templates

A **template** is a page layout: the checked pages whose main content has the same skeleton, whatever their
URL. A **route** groups URLs by pattern. The two are independent:

- **A template can span routes:** `/about` and `/privacy` built on one layout are one template over two
  routes, and so are a collection page and a search page that render the same product grid.
- **A route can hold several templates:** a provisional family whose few checked pages differ.

The purpose is screenshots: `templates` lists every layout seen on the site with one `representative` URL
(the member closest on average to the others), so section-scan captures each layout once. A false merge
(two layouts in one template) hides a layout; a false split only costs one extra screenshot. The rules lean
towards splitting.

- **Skeleton:** the same tag paths as validation, with one change. A **rich-text block** counts as one
  `rich_text` token, whatever it holds, so two posts on the same layout agree even when their bodies differ.
  - **What counts as rich text:** an element with at least 3 prose children (paragraphs, headings, lists,
    tables, figures, images), at least 2 of them `p`, prose making up at least 70% of its children, and no
    `section` or `form` inside. The main-content root itself is never collapsed.
  - **Why cards stay layout:** a product card (`img`, `h3`, one `p`) has one paragraph, so it is not rich
    text. A list counts as prose only when its items hold text: a `ul` of product cards (items holding a
    `div`, `img`, heading, `table` or `form`) is layout, so a page wrapper around an intro and a product
    grid is not collapsed. A different card is a different template, and it should get its own screenshot.
- **Clustering:** every usable checked page of the site is grouped by skeleton, with the same 50% rule and
  JSON-LD check as validation. A page that failed, redirected or read as an error page has no template.
- **Templates phase:** after validation, the rest of the page budget keeps fetching unchecked members of
  each family, in rounds, while that family's pages keep touching templates the family had not touched
  before. Such a template may already exist elsewhere on the site: a blog post on the home page's layout
  still counts as new for the blog family.
  - **Saturation:** each family counts the pages checked since it last touched a new template; the count
    is updated once per round (to 0 after any round in which the family touched a new template, otherwise
    up by the pages that round checked). The family is saturated once the count reaches
    `--saturation-window` (5 by default). A family validated on one template, with no exceptions, is
    saturated from the start, so a 1,000-product shop spends nothing more on products.
  - **Single pages:** each URL in no family is checked once, except pages that were already requested and
    were not HTML, and pages found only deeper than `--max-depth` (links of pages fetched after discovery).
  - **Budget:** families take turns, up to one window of pages each per round. When the budget runs out,
    `stop_reasons` names each family still showing new templates (with how many members were not checked)
    and how many individual pages were never checked.
  - **Options:** `--templates` (the default) runs this phase; `--no-templates` stops after validation, and
    the inventory then covers only the pages already fetched.
- **Output (`schema_version` "2"):**
  - `templates`: each with an `id`, its `representative`, a `status` (`repeated` or `single`),
    `checked_page_count`, `inferred_page_count` (members never fetched of a validated route whose samples
    all use this template; members fetched but unusable, such as redirects, are not counted), its `route_ids`, `page_ids` and `evidence` (`skeleton`, `rich_text`,
    `structured_data`);
  - `urls[].template_id`: set on checked pages only. Unchecked URLs have no template;
  - `observations[].rich_text_blocks`: how many rich-text blocks were collapsed on that page;
  - the `--summary` file adds a Templates table.
- **Limits:**
  - **No rendering:** the skeleton is read from served HTML, not from what the browser paints. Two layouts
    that differ only in CSS are one template.
  - **The rich-text rule is a heuristic:** a large custom component inside a post body (a pricing table
    built from divs, an embedded form) is not collapsed, so that post may count as a new template.
  - **Saturation is not proof:** a layout used by 1 page in 500 can be missed when no sampled page uses it.
    The stop reasons show where the budget ended before a family settled.
  - **The 50% threshold** was tuned on fixtures, not proven on every site.
