# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Requires Node >= 24 / npm >= 11 (see `.nvmrc`).

```sh
npm install
npm start                    # webpack dev server (webpack.dev.js)
npm run build:development    # dev build with sourcemaps
npm run build:production     # production build into dist/
npm run build:check          # tsc --noEmit type check
npm run lint                 # eslint
npm run stylelint            # stylelint on src/**/*.{css,scss}
npm test                     # vitest, single run
npm run test:watch
npx vitest --watch=false --config vite.config.ts src/utils/url.test.ts   # single test file
npm run build:es-check       # production build + es-check that output is valid for legacy browsers
```

CI (`.github/workflows/__quality_checks.yml`) runs `build:es-check`, `lint`, `stylelint`, `build:check` and `test`. All must pass.

Tests run under jsdom. They live next to the code as `*.test.ts`, and import paths resolve through `vite-tsconfig-paths`.

## Contribution rules (from CONTRIBUTING.md)

- New files **must** be TypeScript. New API calls **must** use the Jellyfin TypeScript SDK (`@jellyfin/sdk`), not the legacy ApiClient.
- New app pages **must** be React components built on `src/components/Page.tsx`.
- Use dynamic imports only at the page level. Leave the rest of the code-splitting to webpack.
- Don't reference browser globals (`window.Events` and similar exist only for plugin and legacy compatibility). Import dependencies directly instead.
- Only edit `src/strings/en-us.json`. Other languages come from Weblate. Don't rename existing translation keys.
- Don't create new files in `src/scripts/` (deprecated). `src/elements/` (the `emby-*` webcomponents) is legacy and needs cleanup.
- Don't use Conventional Commits for PR titles or commit messages. Keep each PR to a single focus, and avoid whitespace-only changes in code you aren't otherwise changing.
- The code must run on old TV browser engines. The TS target is ES5, and the `browserslist` in `package.json` defines which JS and CSS features are allowed (`eslint-plugin-compat` and `es-check` enforce this).

## Architecture

Imports are resolved from `src/` (`baseUrl: "src"`), e.g. `import Page from 'components/Page'`. The code is moving toward a Bulletproof React structure: new code goes under the relevant `src/apps/<app>/` (`features/`, `routes/`, `components/`) unless it is shared.

### Boot and routing

`src/index.jsx` loads legacy polyfills, then initializes `ServerConnections` (`lib/jellyfin-apiclient`), plugins, globalize and the many side-effect modules, and renders `RootApp`. `src/RootAppRouter.tsx` builds a single **hash router**. It combines:

- **either** the Modern app routes **or** the Legacy app routes, depending on `layoutManager.modern`,
- the Dashboard routes (admin pages; `isNewLayoutPath` hides the legacy `AppHeader` on these),
- the Wizard routes (first-run setup).

`RootAppLayout` always renders the legacy `Backdrop` and `AppHeader`. The app crashes if they are removed from the DOM.

### Two kinds of routes

Each app's `routes/` folder defines two route lists:

- **Async routes** (`AsyncRoute`, `{ path, page }`): React pages, lazy-loaded by `components/router/AsyncRoute.tsx` from `apps/<app>/routes/${page}`.
- **Legacy routes** (`LegacyRoute`, `{ path, pageProps: { controller, view } }`): old HTML views plus JS controllers. `components/viewManager/ViewManagerPage.tsx` loads them from `apps/<app>/controllers/`. They go through the legacy view manager and `components/router/appRouter.js`.

User routes are wrapped in `ConnectionRequired`; public routes use `<ConnectionRequired level='public' />`. Migrating a page means replacing a legacy entry with an async React page.

### Apps

- `apps/legacy`: the main user UI and the only one that supports the TV layout.
- `apps/modern`: an MUI rewrite of the user UI. It still reuses legacy controllers through its own `legacyRoutes`. No TV layout.
- `apps/dashboard`: admin and metadata pages, mostly MUI. Feature data hooks live in `features/<feature>/api/` (TanStack Query over the SDK).
- `apps/wizard`: first-run setup, all legacy controllers.

### Data access

Use `useApi()` (`hooks/useApi.tsx`) to get the SDK `api` and current user, then call `@jellyfin/sdk` `get*Api(api)` helpers inside TanStack Query hooks. Shared hooks are in `src/hooks/` and `src/hooks/api/`. Legacy code still uses the `ApiClient` from `ServerConnections`. `utils/jellyfin-apiclient/compat` keeps the SDK instance in sync with it.

### Plugins

`src/plugins/` contains client-side modules loaded at runtime by `components/pluginManager` (the name has nothing to do with server plugins). Every media player (HTML video/audio, Chromecast, book/comic/PDF/photo, SyncPlay, etc.) is a plugin, and native app wrappers can override them through `apphost`.

### Legacy stack being replaced

Emby webcomponents (`src/elements`) are being replaced by MUI (dashboard and modern only). `appRouter` and `viewManager` are being replaced by React Router, the legacy ApiClient by the TypeScript SDK, and jQuery by plain TS.

## Fork workflow

This repo is a personal fork for Adam's own customizations. There are no current plans to contribute upstream, but keep that option open.

- `origin` is the fork (`adamfogg/jellyfin-web`). `upstream` is `jellyfin/jellyfin-web` and is fetch-only.
- `master` mirrors upstream and gets no local commits. Sync it with `git fetch upstream && git checkout master && git merge --ff-only upstream/master && git push origin master`.
- Customizations go on `my-changes` (or a short branch off it that gets merged back). Pull in upstream by merging `master` into `my-changes`. Don't rebase it, since it's pushed.
- Keep diffs against upstream files small so upstream merges stay easy: add new files and hook them in with minimal edits, and don't reformat code you aren't otherwise changing. Only edit `src/strings/en-us.json`, never the other language files.
- The upstream contribution rules above (TypeScript, SDK, `Page.tsx`) still apply. The no-Conventional-Commits rule is only an upstream PR requirement.
- Run `lint`, `build:check` and `test` before committing.
- Changes are developed locally and then deployed to a NAS for testing. Jellyfin runs there in Docker: build with `npm run build:production`, copy `dist/` to the NAS, and bind-mount it over the container's web directory (`/jellyfin/jellyfin-web` in the official `jellyfin/jellyfin` image, `/usr/share/jellyfin/web` in the linuxserver.io image). Restart the container and hard-refresh the browser after deploying.
