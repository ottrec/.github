# ottrec

City of Ottawa drop-in recreation schedule data: a scraper, a protobuf dataset
served at data.ottrec.ca, and the ottrec.ca website. Most work happens in the
**website**.

## Workspace layout

This top-level directory is its own git repo (`github.com/ottrec/.github`), but
it tracks **only base files** (README, CONTRIBUTING, CLAUDE.md, .gitignore,
.github) — everything else below is gitignored. Each subfolder is a **separate
repo or worktree** of its own, tied together by `go.work`. So a `git` command at
the root touches the `.github` repo, not the subprojects; cd into the relevant
subfolder for its history.

- `website/` — module `github.com/ottrec/website`: the ottrec.ca site + the
  data.ottrec.ca data/API server. **Most changes go here.**
- `scraper/` — module `github.com/ottrec/scraper`: the scraper (`scraper/`
  subdir) and the protobuf data schema (`schema/`).
- `misc/`, `scripts/`, `infra/` — supporting tooling and deployment.
- `data/` + `cache/` — the dataset repo (a `--filter=blob:none` clone with a
  worktree); **never regenerate or commit here**.
- `data/data.pb` — a sample data protobuf for local testing without a live data
  server.

Make changes in the relevant worktree and run Go commands from that module's
directory.

## Working preferences

- **The user is an experienced developer with deep domain knowledge** of this
  project and its stack. Be concise and to-the-point; don't overexplain basics,
  restate the obvious, or pad with caveats and hand-holding. Lead with the
  answer or the change. Do call out problems you anticipate and alternative
  approaches worth considering.
- **Subtle, muted UI.** Default to quiet flexoki tints, bordered (not solid
  accent) buttons, small grey secondary text. Past corrections: "too
  prominent", "bold and distracting", "cluttered". Confirm before styling
  anything attention-grabbing.
- **Dry, concise comments.** Simple, to-the-point, concise, matter-of-fact, no
  cheesy/obvious phrasing, no em/en dashes (in copy or Go doc comments).
- **Don't run the dev server** or data-loading programs — the user runs those.
  Verify with build/generate/vet (see below), not by serving.
- **The user iterates in rapid small follow-ups.** Messages arriving mid-task
  are usually amendments; keep changes minimal and targeted.
- **Scope changes to the website worktree** unless told otherwise. In
  particular, don't touch the data.ottrec.ca rendering path unless explicitly
  asked; when shared code affects it, preserve its existing behavior.
- **Never touch the scraper repo** (`scraper/`, module
  `github.com/ottrec/scraper`). The one exception: if you genuinely need a
  helper on the schema types (`scraper/schema/`), ask the user first — don't add
  it unprompted.
- **Don't modify packages under `website/pkg/`** (`ottrecidx`, `ottrecql`,
  `ottrecexp`, `ottrecexph`, `ottregions`, etc.) unless explicitly asked, with
  clear instructions on the intended scope, edge cases, and semantics. These are
  load-bearing libraries with subtle behavior; treat them as consume-only by
  default. If a task seems to need a change there, surface it and ask rather
  than editing — and never change observable semantics as a side effect.
- Modern TypeScript only: no frameworks, no jQuery-isms. ASI style (no
  statement-terminating semicolons). Use modern JS/TS syntax and APIs freely, as
  long as it transpiles/runs down to the supported targets (~Chrome/Firefox 110,
  Safari 15) — esbuild handles syntax, but mind runtime API availability on
  those browsers.
- **Use modern Go features** (the workspace is on a current Go toolchain): range-
  over-func iterators, `min`/`max`, generics, `slices`/`maps`/`cmp`, structured
  `log/slog`, etc. Prefer them over older hand-rolled equivalents.
- For long-form text (docs, copy, posts, READMEs), don't write it out unprompted
  — give the user a short point-form list of recommendations on what may be
  worth adding and let them write it.
- **The user stages changes and creates commits**, not you. Don't `git add` or
  `git commit` unless explicitly asked. Changes made mostly by Claude get a
  `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>` trailer in the
  commit message.

## Website architecture

Adding/changing a page generally touches four things:

1. **Route** — register in `routes.Website()` (`routes/website.go`). Handlers
   embed `websiteHandlerBase` and call `h.render(w, r, func(data
   ottrecidx.DataRef) (templ.Component, int, error) {...})`, which handles
   ETags, content-encoding negotiation, and error pages. New indexable pages
   also go in the sitemap path list.
2. **Templates** — [templ](https://templ.guide) files in `templates/`.
   Regenerate with **`go generate ./templates`** (not `go tool templ`
   directly). Pages share a navbar, theme head, and footer; reuse existing
   shared components rather than duplicating.
3. **Static assets** — files under `static/`, registered in `static/static.go`
   and referenced via `static.Path(...)` (content-hashed URLs). See the asset
   pipeline below.
4. **Data access** — everything reads through `pkg/ottrecidx` `DataRef` getters
   and iterators. Coordinates: `fac.GetLngLat()` returns **lng, lat, ok** (easy
   to swap). Timezone is `ottrecidx.TZ`; dataset timestamp is
   `data.Index().Updated()`.

Notable packages: `pkg/ottrecidx` (indexed read API over the dataset),
`pkg/ottrecql` (the schedule query language used by search), `pkg/ottrecexp(h)`
(simplified export / HTML rendering), `pkg/ottregions` (lat/lng → Ottawa
region + sector, generated from OSM), `internal/asset` (the asset pipeline).

### Rendering: server-first, progressive enhancement

The site is **server-rendered (templ) and works without JavaScript**. Client-side
TypeScript only **progressively enhances** an already-functional page — it never
renders core content or gates it behind JS.

- Render the full, usable page on the server; links, forms, and content must
  work with JS disabled.
- Scripts attach to server-rendered markup and degrade gracefully: each
  enhancement no-ops if its target elements are absent, and there's a sensible
  no-JS fallback (e.g. a plain form/link behind an enhanced control).
- Pass data to scripts via server-rendered markup (data attributes / embedded
  JSON islands), not by fetching and rendering on the client.
- Prefer this over client-side rendering, SPA patterns, or frameworks. Reach for
  client JS only when the behavior genuinely can't be done server-side.

### Asset pipeline (CSS / TS)

The pipeline lives in `internal/asset`; `static/static.go` wires it up. Assets
are content-addressed (hashed names) and compiled **lazily at runtime**, so
`go build` does **not** catch CSS/TS compile errors.

- **CSS** is minified by lightningcss (`compileCSS`), with `url()` references
  rewritten to hashed asset names. Targets ~Chrome/Firefox 110, Safari 15.
- **TypeScript** is bundled to a minified IIFE by esbuild (`internal/esbuild`,
  pure Go). `.ts` files can import each other and vendored npm packages
  (tree-shaken). tsc is **editor/CI only** — the server never type-checks.
- **npm deps** are vendored as uncompressed tars under `static/lib/` via
  `go generate ./static` (`internal/npm`); tracked in `package.json`. Type
  checking needs `npm install` first (`node_modules` is gitignored).
- **Fonts** (Source Sans 3 / Source Serif 4 / Asap / Material Symbols) are
  subset by explicit codepoint in `static/static.go`. Material Symbols render
  via CSS `::before` content escapes (ligature names don't work in the subset).
- **Theming** is a flexoki light/dark palette driven entirely by
  `color-scheme` + `light-dark()` CSS vars, with an anti-FOUC inline script.

### Verifying changes

Run from the `website/` module dir:

```sh
go generate ./templates              # after editing any .templ (check output for errors!)
go build ./... && go vet ./...
go run ./cmd/ottrec-website-check     # compile all CSS + TS (server-free; catches what go build can't)
npx -y -p typescript tsc -p tsconfig.json   # TS type checking (run npm install first)
```

`cmd/ottrec-website-check` builds every static asset and reports compile
failures without serving; `--css` / `--js` / `--all` narrow what it checks. It
does **not** type-check TS. The commands live in `cmd/`: `ottrec-website`
(site), `ottrec-data` (data/API), `ottrec-timemachine` (experimental history
viewer), `ottrec-website-check`.

### Gotchas

- A stale generated templ file still `go build`s — always check
  `go generate ./templates` output, not just the build.
- In `.templ` text, a run that **begins with `for`/`if`/`switch`** (or a bare
  `@`) is parsed as a Go statement, not literal text. Reword (e.g. avoid
  starting prose right after a closing tag with those words) or escape as
  `{ "for" }`.
- `routes.render`'s handler prologue expects `Vary: Accept-Encoding`
  (templates render panics without it). Follow an existing handler.

## Scraper & schema (`scraper/` module)

The scraper (`scraper/scraper/main.go`) scrapes City of Ottawa drop-in schedules
into a protobuf dataset. Design philosophy (see `scraper/README.md`): scrape
only basic facility + schedule info with **minimal processing** to keep the
scraper reliable and the schema stable long-term; freeform fields (notifications,
schedule changes, exceptions) are kept as best-effort raw HTML; parsing
problems become per-facility error messages rather than failures.

The schema is `scraper/schema/schema.proto` (`schema.pb.go` generated;
`schema.go` for helpers). It carries:

- **Raw** scraped fields (the source of truth, least lossy), plus
- **Optional best-effort parsed/normalized** fields (geocoded lng/lat,
  normalized activity/schedule names, parsed date/time ranges, reservation-
  requirement guesses). Parsing is unambiguous-only; overlapping/holiday
  schedules are not merged.

The dataset is published in raw (protobuf/textpb/json) and **simplified**
(json/csv) forms at data.ottrec.ca, updated daily. The website consumes it via
`pkg/ottrecidx` (raw protobuf), never re-scraping.

### Using the data — gotchas

- **Coordinates are lng, lat** (`fac.GetLngLat()` returns lng, lat, ok). Easy to
  swap; this exact bug shipped once historically.
- **Parsed/normalized fields are best-effort and optional.** Parsed date/time
  ranges, normalized names, and reservation guesses may be absent or wrong when
  the source was ambiguous — always have a fallback (e.g. raw label/caption) and
  check the "ok" return on parsed getters.
- **Schedule date ranges are negative-only.** Overlapping/holiday schedules are
  *not* merged, so a schedule's start/end range only reliably tells you an
  activity does *not* apply on a date — not that it does. Don't compute "what's
  on now" from a single schedule's range alone.
- **Fixed-date (holiday) schedules** can override regular ones for specific
  dates and may share weekly times across different date ranges; treat them
  separately from recurring schedules.
- **Freeform fields are raw HTML.** Notifications, schedule changes, and
  exceptions are unparsed best-effort HTML — render/sanitize accordingly, don't
  expect structure.
- **Per-facility errors exist and are meaningful.** Each facility carries an
  errors array for parsing problems; surface them rather than assuming clean
  data.
- **Names are normalized lowercase** in the parsed fields; the human-facing
  label/caption is separate.
- The schema may change in backwards-incompatible ways; the **raw protobuf** is
  the least-lossy source of truth, the **simplified** export is the easiest to
  consume.
