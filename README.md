# PyMCU Documentation

Source of **[docs.pymcu.org](https://docs.pymcu.org)**, the documentation for
[PyMCU](https://github.com/PyMCU/PyMCU), a compiler that turns a statically-typed,
allocation-free subset of Python into bare-metal firmware for AVR, ARM (RP2040 / RP2350)
and PIC.

Built with [Astro](https://astro.build) and [Starlight](https://starlight.astro.build).

## Getting started

Node **22.12 or newer** is required: Astro 6 refuses to start below that.

```bash
npm ci --legacy-peer-deps
npm run dev
```

The `--legacy-peer-deps` is not optional. See [Known rough edges](#known-rough-edges).

| Command           | What it does                     |
| ----------------- | -------------------------------- |
| `npm run dev`     | dev server on `localhost:4321`   |
| `npm run build`   | production build into `dist/`    |
| `npm run preview` | serve the built site locally     |
| `npm run check`   | astro check, eslint and prettier |
| `npm run fix`     | apply eslint and prettier fixes  |

## Writing docs

Pages are `.md` and `.mdx` files under `src/content/docs/`, and each one becomes a route
named after its path. The sidebar is **not** generated from the file tree: add new pages
to the `sidebar` array in `astro.config.ts` or they will be reachable but unlisted.

Images go in `src/assets/` and are embedded with relative links.

## Deployment

Pushing to `main` publishes the site. Cloudflare Workers Builds is connected to this
repository and runs:

```bash
npm ci --legacy-peer-deps && npm run build   # build command
npx wrangler deploy                          # deploy command
```

It builds with `NODE_VERSION=22.12.0`, set as a build variable in the Cloudflare
dashboard. The target is the `pymcu-docs` worker, which serves `dist/` as static assets;
there is no `_worker.js` and no server-side rendering.

CI on GitHub builds and checks every push but does **not** deploy, so there is one
publishing path and no API token stored in this repository.

To publish by hand, `npm run deploy` still works and needs a Cloudflare login.

## Known rough edges

**`npm ci` needs `--legacy-peer-deps`.** The site is on Astro 6 while `@astrojs/tailwind`
still declares a peer of `astro ^3 || ^4 || ^5`, and its latest release (6.0.2) has not
moved. The package has no Astro 6 successor: the fix is migrating to Tailwind 4 with
`@tailwindcss/vite`, which touches the styles and has not been done yet. Until then a
plain `npm ci` fails.

**`npm run check` fails on formatting.** Prettier reports roughly 54 files that predate
the check being enforced. It is reported by CI but does not block anything. Run
`npm run fix` on files you touch rather than reformatting the tree in one go.

## History

This site lived on the `starlight-docs` branch of `PyMCU/website` until August 2026,
sharing a repository with the pymcu.org landing page. The history before the split is
preserved here, which is why early commits mention an AstroWind landing.
