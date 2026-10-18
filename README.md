# Agroverse Wiki

An independently maintained [Wiki.js](https://github.com/requarks/wiki) fork for the **UC Davis Unofficial Wiki**. This README describes additions in this fork and how to use them. The inherited project README is preserved in [README.upstream.md](README.upstream.md); licensing and upstream credits remain unchanged.

## What this fork adds

### Native Aggie Bites homepage section

The `home` page includes a Vue/Vuetify dining section backed by the independent Aggie Bites menu service:

- Switch between Cuarto, Segundo, Tercero and Latitude, and choose the current/next meal or breakfast, lunch and dinner.
- Follow the Wiki page language: Chinese pages show translated names with the English originals; English pages show the originals. AI-translated names are marked on Chinese pages.
- Show the dining-zone name and color beside each highlighted dish, with colors adapted to the Wiki's dark theme.
- Display an AI-written meal summary when available. Rule-generated summaries are omitted because they repeat the dish list.
- Open a dish's details or the complete menu on the menu site.
- Show unavailable-menu and closure states. If the API fails or times out after five seconds, show a short message and a link to the menu site.

The component follows Vue's lifecycle, cancels obsolete requests, and prevents older responses from overwriting a newer selection. It inherits the Wiki's typography, surfaces and theme instead of embedding a separate page.

**No iframe, theme code injection, page-body rewriting, or menu commits to Git sync are involved.** Only component code lives in this repository. Menu data, translations, model calls, snapshots, administration and Bot delivery belong to the separate Aggie Bites service. A Wiki outage does not stop that service's worker or Bot.

## Using the homepage section

1. Open the Wiki's `home` page, for example `/zh/home` or `/en/home`.
2. Choose a dining hall and meal. The displayed date comes from the service's Davis-time schedule.
3. Use the colored zone label to locate the serving station. Select a dish for its next scheduled appearances, or **Full menu** for the complete daily menu.
4. Use the Wiki's normal language and theme controls. The dining section updates with them.

The component is mounted only when the page path is `home`; it is not automatically inserted into arbitrary pages named `index` or `welcome`. Site operators can change that condition in [page.vue](client/themes/default/components/page.vue). The section is placed above the existing page content and does not modify that content.

## Operator setup

### 1. Run Aggie Bites independently

Deploy its API and single worker using that service's deployment guide. The default menu-site links in this component point to `https://menu.ucdavis.top`. Change `menuLink` in [aggie-bites.vue](client/themes/default/components/aggie-bites.vue) when using another hostname.

Do not put model keys, Bot tokens or menu-service administrator credentials into the Wiki component. It consumes only a public, read-only API.

### 2. Connect the same-origin API route

The component fetches `/bites-api/today`. Add the following **inside the existing HTTPS server block for the Wiki hostname**. This example assumes Nginx and `bites-api` share a Docker network:

```nginx
resolver 127.0.0.11 valid=30s ipv6=off;

location /bites-api/ {
    set $bites_upstream bites-api:8000;
    rewrite ^/bites-api/(.*)$ /api/v1/$1 break;
    proxy_pass http://$bites_upstream;
    proxy_connect_timeout 2s;
    proxy_read_timeout 5s;
    proxy_set_header Host menu.ucdavis.top;
}
```

Validate with `nginx -t` before reloading Nginx. Reuse an existing resolver directive if the server block already has one. Docker's resolver address is specific to this example; use the correct resolver and backend address outside Docker.

Dynamic upstream resolution allows Nginx to start even if the menu container is unavailable. Same-origin routing avoids browser CORS configuration. If the Wiki is gated by an authentication proxy, ensure this public API path can be fetched without credentials; the component uses `credentials: 'omit'`.

### 3. Deploy this fork's frontend

Running the unmodified `ghcr.io/requarks/wiki:2` image will **not** include this component. Build this fork and replace only the Wiki application image, preserving the existing database configuration and persistent volumes.

For an isolated release checkout, set the inherited package's development flag to false before building:

```sh
git clone --branch main https://github.com/zhenmunse/wiki.git agroverse-wiki-release
cd agroverse-wiki-release
node -e "const fs = require('fs'); const p = require('./package.json'); p.dev = false; fs.writeFileSync('package.json', JSON.stringify(p, null, 2) + '\n')"
docker build -f dev/build/Dockerfile -t agroverse-wiki:local .
```

Build on the target architecture, or explicitly use a matching Docker Buildx platform. The inherited Dockerfile uses Node 24. This command is a deployment recipe, not a claim that a complete release build has been validated for every platform.

The inherited Build + Publish and Helm publication workflows are guarded to run only in `requarks/wiki`, because their destinations and credentials belong to upstream. **A merge in this fork does not automatically publish a fork image.** Configure a separate fork-owned image workflow before relying on automated releases.

### 4. Verify the integration

- Open `/bites-api/today` on the Wiki hostname and confirm JSON with `schema_version: "1"`.
- Open `/zh/home` and `/en/home`; confirm dining choices and language-specific names.
- Switch the Wiki's theme; confirm zone colors remain visible.
- Navigate away and return; confirm the section loads normally.
- Simulate an unavailable API in a test environment; the rest of the page must remain usable and the menu-site link must appear.

## API compatibility

The section consumes the Aggie Bites `/api/v1/today` contract, optionally with `?meal=breakfast|lunch|dinner`. It expects the response date, meal, hall states, highlights and bilingual summaries. Each highlight contains a stable dish ID, English/Chinese names and translation status; `zone` is optional.

Existing v1 fields must retain their types and meanings. Add optional fields for compatible changes; introduce a new API version for breaking changes. Missing zone metadata simply omits its label and colored accent. Unknown zone names remain readable without being assigned a misleading known color.

## Development and troubleshooting

Use the inherited Wiki.js development setup and database configuration; see [upstream documentation](https://docs.requarks.io/). This repository includes `yarn dev`, `yarn build`, and `yarn test` scripts. Keep local configuration and credentials out of commits.

| Symptom | Check |
| --- | --- |
| No dining section | Confirm the fork's rebuilt assets are deployed and the page path is `home`. |
| Menu unavailable message | Check `/bites-api/today`, the Nginx route, shared Docker network and Aggie Bites API health. |
| Names only in English | The service has no Chinese translation yet, or the Wiki page locale is English. |
| No summary paragraph | Expected when `summary_source` is `rule`; highlighted dishes still appear. |
| Empty or closed meal | Check the service's returned status and official menu; absence of data is not proof of closure. |
| Changed component not visible | Rebuild the frontend/image; changing server-side page text does not replace bundled Vue assets. |

The focused Vue compilation, rendering and request-lifecycle tests used for this integration live in the adjacent Aggie Bites repository (`tests/vue`). They cover stale-response handling, unmount cancellation, failures, zone labels, bilingual rendering and dark-mode binding; they do not replace a full Wiki release build or production deployment check.

## Project scope and upstream

This fork currently adds the homepage dining integration described above. Inherited permission-management fixes and upstream build-tool updates are not presented as new Agroverse features. User ratings, photo uploads, a new identity provider and a replacement for Wiki.js administration are not implemented by this component.

This is an unofficial UC Davis community project, not an official university product. Wiki.js remains licensed under [AGPL-3.0](LICENSE). See the [preserved upstream README](README.upstream.md) for upstream features, credits, support and contribution information.
