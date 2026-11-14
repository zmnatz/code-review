# code-review

Shared tooling for reviewing pull requests. At the moment that is one reusable
workflow: screenshots of a PR's pages, posted as a single comment that updates in place.

## PR screenshots

On each pull request the workflow builds your site, starts it, and screenshots pages at
desktop (1280px) and mobile (390px) widths. It reports console errors, page errors and
failed requests per page, stores the images on an orphan `pr-screenshots` branch so they
embed inline, and keeps one comment on the PR up to date.

Which pages: `/` always, then the routes your rules file maps from the PR's changed
files, then any paths listed in the PR description's `## Screenshots` section, one per
line. Add `wait: <selector>` or `delay: <ms>` after a path for pages that fill in late.

```markdown
## Screenshots

- /
- /reports/  wait: [data-testid=report]
```

### Use it

`.github/workflows/screenshots.yml`:

```yaml
name: Screenshots
on:
  pull_request:

jobs:
  screenshots:
    # A fork's token can't push the images or comment.
    if: github.event.pull_request.head.repo.full_name == github.repository
    uses: zmnatz/code-review/.github/workflows/screenshots.yml@v1
    permissions:
      contents: write
      pull-requests: write
    with:
      rules: screenshots/rules.yml
      noise: screenshots/noise.yml # optional
```

`.github/workflows/screenshots-cleanup.yml`, which deletes a PR's images when it closes:

```yaml
name: Clean up PR screenshots
on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    uses: zmnatz/code-review/.github/workflows/screenshots-cleanup.yml@v1
    permissions:
      contents: write
```

### Inputs

| Input | Default | |
|---|---|---|
| `rules` | required | YAML mapping changed files to routes (below). |
| `noise` | none | YAML of findings to leave out of the comment (below). |
| `build` | `bun run build` | Builds the site. |
| `install` | `bun install --frozen-lockfile` | Installs its dependencies. |
| `start` | `node_modules/.bin/next start -H 127.0.0.1 -p {port}` | Serves the build. `{port}` is a free port, also in `$PORT`. |
| `vars` | none | Shell command printing `name=value` lines; each fills `{name}` in a route. |
| `bun-version` | `latest` | |
| `node-version-file` | none (Node 24) | A file such as `.nvmrc` naming the Node the site builds with. The tool runs on it too, so it needs Node 22.18 or later. |
| `tool-ref` | `v1` | Which ref of this repo to run the tool from. |

The site is started inside the job from your checkout, so it needs no deploy and no
preview URL. Because it runs on a runner with nothing else in it, it shows the build,
not your production data or services.

### Rules file

```yaml
rules:
  - files:               # globs: ** crosses directories, * stays in one
      - src/components/Report/**
      - pages/report.tsx
    routes:
      - path: /report/
      - path: /player/{player}/   # {name} comes from the `vars` input
        waitFor: "[data-testid=player]"
        delayMs: 500
```

`vars` is how a route can name a page whose id changes with your data:

```yaml
with:
  vars: jq -r '"player=" + (.players | keys[0])' public/data/players.json
```

### Noise file

```yaml
noise:
  - kind: request            # console, pageerror or request; omit to match all
    pattern: "/_next/data/\\S+ 404$"   # a regular expression
    reason: Why this is safe to ignore.
```

### Running it by hand

```sh
npm install && npx playwright-core install chromium
node screenshots/capture.ts --rules rules.yml --app ../my-site --start 'node_modules/.bin/next start -p {port}' --page '/reports/'
```

It writes `screenshots/<page>-<viewport>.png` and `report.json`, and exits non-zero when a
page fails to load or answers with a non-200 status.

## Development

`npm test` runs the tests on Node's built-in runner (Node 22.18 or later, which runs
TypeScript directly). It has no build step.
