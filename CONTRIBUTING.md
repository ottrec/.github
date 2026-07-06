# Contributing

## Development

### Setup

#### Dependencies

##### All

- Git
- Go
- NPM

##### Infra

- OpenTofu
- Ansible
- Bun
- Rust
- cargo-zigbuild

#### Workspace

```bash
# clone repositories
git clone https://github.com/ottrec/.github ottrec
git clone https://github.com/ottrec/scraper ottrec/scraper
git clone https://github.com/ottrec/website ottrec/website
git clone https://github.com/ottrec/data ottrec/data --filter=blob:none
git clone https://github.com/ottrec/misc ottrec/misc
git clone https://github.com/ottrec/infra ottrec/infra # private
git -C ottrec/data worktree add ../cache cache

# set up go workspace
go work init -C ottrec scraper website misc

# set up tofu
env -C ottrec/infra/terraform tofu init

# set up npm deps for ide integration
env -C ottrec/website npm ci --ignore-scripts
```

#### VSCode extensions

Recommended:

- [`golang.go`](https://marketplace.visualstudio.com/items?itemName=golang.Go)
- [`a-h.templ`](https://marketplace.visualstudio.com/items?itemName=a-h.templ)
- [`pbkit.vscode-pbkit`](https://marketplace.visualstudio.com/items?itemName=pbkit.vscode-pbkit)
- [`dnut.rewrap-revived`](https://marketplace.visualstudio.com/items?itemName=dnut.rewrap-revived) - for rewrapping comments (Alt+Q)
- [`opentofu.vscode-opentofu`](https://marketplace.visualstudio.com/items?itemName=OpenTofu.vscode-opentofu)

Optional:

- [`aaron-bond.better-comments`](https://marketplace.visualstudio.com/items?itemName=aaron-bond.better-comments)
- [`yo1dog.cursor-align`](https://marketplace.visualstudio.com/items?itemName=yo1dog.cursor-align) - for aligning cursors (Alt+A)
- [`hangxingliu.vscode-systemd-support`](https://marketplace.visualstudio.com/items?itemName=hangxingliu.vscode-systemd-support)
- [`PKief.material-icon-theme`](https://marketplace.visualstudio.com/items?itemName=PKief.material-icon-theme)
- [`mhutchie.git-graph`](https://marketplace.visualstudio.com/items?itemName=mhutchie.git-graph)

### Useful commands

Many of these are also available as vscode tasks.

#### Data

```bash
# pull and reset latest cache/data
git -C data fetch origin
git -C cache reset --hard @{u}
git -C data reset --hard @{u}
git -C cache clean -fdx
git -C data clean -fdx

# checkout specific date
git -C cache clean -fdx
git -C data clean -fdx
git -C cache reset --hard '@{u}@{2026-05-15}'
git -C data reset --hard '@{u}@{2026-05-15}'
```

#### Scraper

```bash
# test scraper on local data using cached pages
go run ./scraper/scraper -cache cache -geocodio -scrape -export.pretty -export.proto data/data.proto -export.pb data/data.pb -export.textpb data/data.textpb -export.json data/data.json

# lint schema
go run github.com/bufbuild/buf/cmd/buf@v1.66.1 lint scraper/schema/schema.proto

# check for breaking schema changes
go run github.com/bufbuild/buf/cmd/buf@v1.66.1 breaking --config '{"version":"v2","breaking":{"use":["WIRE_JSON"]}}' --against 'data/data.proto#include_package_files=false' 'scraper/schema/schema.proto#include_package_files=false'

# generate schema
go generate ./scraper/schema
```

#### Website

```bash
# run dev server
go run ./website/cmd/ottrec-data --repo-remote $PWD/data # http://localhost:8082
go run ./website/cmd/ottrec-website # http://localhost:8083
go run ./website/cmd/ottrec-timemachine # http://localhost:8084

# format and generate templates
go generate ./website/templates

# update vendored fonts and libs
go generate ./website/static

# check css/typescript compilation
go run ./website/cmd/ottrec-website-check

# check typescript
env -C website npm exec tsc

# sanity check ottrecidx against data.ottrec.ca dev server
go run ./website/pkg/ottrecidx/profile.go -check -base http://localhost:8082/

# profile ottrecidx
go run ./website/pkg/ottrecidx/profile.go -cpuprofile /tmp/cpu.pprof -memprofile /tmp/mem.pprof
go tool pprof -http :6060 /tmp/cpu.pprof
go tool pprof -http :6061 /tmp/mem.pprof
```

### Infra

Binaries are built locally as static executables. Everything is pinned except ottrec, which always builds the local clone or the latest commit.

```bash
# deploy
env -C infra/ansible ansible-playbook -i inventory.yml playbook.yml

# deploy ottrec
env -C infra/ansible ansible-playbook -i inventory.yml playbook.yml -t ottrec

# apply terraform
env -C infra/terraform tofu apply
```

## Guidelines

### Scraper schema updates

- Do not make backwards-incompatible protobuf changes.
- Avoid renaming fields unless absolutely necessary (this will break JSON users).
- Avoid adding fields which do not mirror the inherent structure of the website and could be consistently computed from the existing fields.
- Underscored fields can be used for computed fields where how they are parsed may change in the future, but the field itself is an inherent property (e.g., schedule date ranges from the caption).
- Do not change the semantic meaning of existing fields.
- Do not remove fields entirely; deprecate them, but continue to set them.
- Keep fields in sync with the website ottrecidx package.

If backwards-incompatible changes are ever necessary, create a new v2 subdir with the new schema, put stuff in there, create a new v2 api using the v2 schema, and create a new v2 branch in the data repo.

### Website

Be careful with the packages in pkg/* (data export, simplified schema, query language, indexer, heuristics, storage). Never use Claude on them, and don't touch them carelessly or do unnecessary refactoring either. They were very carefully designed to handle the schedule data correctly and efficiently, and to preserve backwards/forwards compatibility. Certain packages (e.g., ottrecql and ottrecidx) are also somewhat fragile and easy to break in subtle ways.

However, as a result of this careful backend work, Claude tends to do a good job with the frontend templates/scripts/styles as long as you have a clear vision.

CSS should be written for compatibility, but you can safely use most modern features since it's transpiled at startup using LightningCSS.

Scripts should be written in TypeScript. Prefer to write modern JS instead of importing libraries. All pages should also fall back gracefully without scripting. If you must import a dependency, choose a well-designed modern one without a crazy tree of dependencies.

Fonts are vendored into the repo and subsetted at runtime. See static.go.

You may wonder why I don't use a "proper" frontend framework or at least a separate build step, and it's because:

- I want server-rendered progressively-enhanced pages because SPAs suck (and SEO is also easier this way).
- I hate "modern" frontend framework-driven development.
- I can use templ for JSX-like syntax in Go (which also ties in really well with my custom indexer).
- Go already has a very nice and mature JS bundler/transpiler, esbuild.
- I ported LightningCSS (a nice CSS transpiler written in Rust) to Go using wasm2go.
- NPM is a security nightmare. Vendoring the deps and building them with esbuild sidesteps this entirely.
- Since LightningCSS and esbuild are very fast, I can build stuff at startup.
- As a result, I can build and run with just the Go toolchain, and also get a tight feedback loop during development.

And if you're wondering why I rolled my own indexer instead of using a separate database:

- Again, I control the entire stack, so I can have nice things.
- The custom indexer also allows heuristics to be computed efficiently and without needing to keep logic in sync with an external datastore.
- Since everything is in Go, I can do fancy stuff with iterators so the API feels natural.
- I can do interning to amortize the memory cost of loading many versions at the same time.
- Almost every page render needs to read almost an entire version of the data and filter it in some way. This design lets me do filtering with zero copies and very little overhead.
- Without this, a lot of cycles would be wasted normalizing and denormalizing the data in/out of the heirarchical form needed for rendering.
- It was very fun to design and write.

When doing URL parameters, handle them such that errors are obvious, yet still have them tolerant to unknown params, and also design them carefully for forwards/backwards compatibility.

All pages should have sensible caching, making use of ETags where possible.

Pages should also have canonical URLs set, pointing at the main page of that category to avoid being penalized by Google for the many generated pages.
