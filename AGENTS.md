# Working on this repository (Base44 sandbox notes)

## What this app is

GDevelop is a C++ game engine with a JavaScript/React IDE. Only one part is
served in the sandbox preview: **the editor, `newIDE/app`** (Create React App,
`react-scripts start`, port 3000). The C++ `Core`/`Extensions` code, the desktop
`electron-app` and the `web-app` deploy scripts are not needed to see the IDE run.

## Running it

```bash
docker compose -f docker-compose.base44.yml up -d --build
docker compose -f docker-compose.base44.yml logs -f web
```

First boot takes several minutes. `newIDE/app`'s `npm ci` postinstall is what does
the real work: it installs `GDJS/` dependencies, then `import-resources`:

* downloads the **pre-built `libGD.js` + `libGD.wasm`** (the emscripten build of
  `Core`/`GDJS`) from `https://s3.amazonaws.com/gdevelop-gdevelop.js/<branch>/commit/<HEAD sha>`,
  falling back to `HEAD~1..HEAD~3` then `master/latest`. This is why the container
  needs the `.git` directory and outbound network access;
* builds the GDJS runtime + extensions with esbuild (`GDJS/scripts/build.js`);
* copies the Monaco editor out of `node_modules` and extracts the zipped
  Piskel/Jfxr/Yarn editors from GitHub releases.

All of these outputs (`newIDE/app/resources/`, `public/libGD.js`,
`public/external/…`, `src/Version/VersionMetadata.js`) are gitignored by
`newIDE/app/.gitignore`, so generated files never pollute `git status`.

## Things that are not obvious

* `node_modules` for both `newIDE/app` and `GDJS/` are **named volumes** in
  `docker-compose.base44.yml`, not files in the repo. To install a new dependency
  after editing a `package.json`, remove the marker and restart, or run
  `docker compose -f docker-compose.base44.yml exec web sh -c 'cd /repo/newIDE/app && npm install'`.
  The startup command re-runs `npm ci` automatically whenever `package-lock.json` changes.
* `newIDE/app/package.json` `start` runs `import-resources` again, so the GDJS
  runtime is built twice on a cold first boot. Later restarts skip the install and
  build it once.
* **Source-map warnings:** `GENERATE_SOURCEMAP=false` is set for the sandbox dev
  server. Monaco, `GDJS-for-web-app-only` and `qr-creator` reference `.map` files
  they don't ship, so CRA's `source-map-loader` used to print ~21
  `Failed to parse source map … ENOENT` warnings on every compile. It costs
  original-source mapping in browser devtools; remove that env var to get it back.
* **Memory:** the dev server (webpack + Monaco + the whole editor) sits at ~3.5GB
  RSS and, with Node's ~4GB default heap on this 8GB sandbox, aborts with
  `FATAL ERROR: … JavaScript heap out of memory` a few minutes after starting —
  the container then restarts itself (`restart: on-failure`) and the preview page
  needs a refresh. `NODE_OPTIONS=--max-old-space-size=5120` in
  `docker-compose.base44.yml` gives it headroom; `ulimit -c 0` in the startup
  command stops a crash from dropping a multi-GB `core` file into the repository.
* **No outbound network from the previewed page:** requests the IDE makes from the
  browser to `api-dev.gdevelop.io` fail with `TypeError: Failed to fetch`
  (announcements, template/asset stores, shop, tutorials, login), so the home page
  shows "Can't load the announcements…". This is an environment restriction, not an
  app bug — the API is reachable from the sandbox itself (`curl` gets 200) and
  answers with `access-control-allow-origin: *`. Local project editing is unaffected.
* No credentials are required to boot: the Firebase/cloud-project config is baked
  into the source and is only used for optional online features. Without a real
  account the editor still opens, and local projects work.
* `patches/` are applied by `patch-package` during `npm ci`; keep them mounted with
  `package-lock.json` (both are in the bind mount).
* The dev server host firewall is off by default: CRA only enables
  `allowedHosts: [lanUrlForConfig]` when a `proxy` field exists in `package.json`,
  and there is none — so the preview hostname is accepted without extra config.

## Verifying it works

```bash
curl -fsS http://localhost:3000/ | grep 'id="root"'      # IDE HTML shell
curl -fsS -o /dev/null -w '%{http_code}\n' http://localhost:3000/libGD.js
docker compose -f docker-compose.base44.yml ps           # web: healthy
```

In the browser, the IDE should reach its "Start a new project" / project browser
screen; a blank page usually means `libGD.js` failed to download (check the
`web` logs for the S3 download step).

## Tests (not run in the sandbox by default)

```bash
docker compose -f docker-compose.base44.yml exec web sh -c 'cd /repo/newIDE/app && npm test'
docker compose -f docker-compose.base44.yml exec web sh -c 'cd /repo/newIDE/app && npm run flow'
```
