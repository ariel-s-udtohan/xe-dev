# @ignite/web integration

The minimum setup for using Xcel Energy's **@ignite/web** design-system
components (Lit web components) in Edge Delivery Services blocks. No block uses
them yet.

## What's in the repo

| Path | Purpose |
|---|---|
| `package.json` | `@ignite/web` dependency, and the `build:ignite` scripts that generate the bundle. |
| `.npmrc` | Registries for the private packages: `@ignite` (Xcel JFrog) and `@fortawesome` (Font Awesome Pro, a dependency of `@ignite/web`). No tokens. |
| `scripts/ignite/bundle/` | The generated Ignite components: `primitives/`, `compositions/`, `themes/tokens.css`. Edge Delivery has no build step, so the bundle is committed. Don't edit it by hand. |
| `scripts/components/ignite.js` | Loads an Ignite bundle on demand (see below). |
| `scripts/components/xe-icon.js` + `icons.js` | The local `<xe-icon>` and the site's icon set, used by every Ignite component. |
| `scripts/components/fontawesome-shim.js` | Stand-in for the Font Awesome Pro icons the bundles import, mapped by the import map in `head.html`. |
| `tools/ignite/extract-tokens.mjs` | Derives `themes/tokens.css` from the Ignite theme during `build:ignite`. |

## Installing

Both private registries need a token in your user npmrc (never in the repo):

```sh
npm config set "//xcel.jfrog.io/artifactory/api/npm/npm/:_authToken" "<jfrog-token>"
npm config set "//npm.fontawesome.com/:_authToken" "<fontawesome-token>"
npm install
```

CI does the same from the repository secrets `JFROG_NPM_TOKEN` and
`FONTAWESOME_NPM_TOKEN` (Settings → Secrets and variables → Actions); the
Build workflow fails with a clear error if either is missing.

## Updating the bundle

**From GitHub:** run the **Build Ignite bundle** workflow (Actions tab → Run
workflow), pick the branch and enter the version (for example `0.40.0`). It
installs that version with the repository secrets, runs `build:ignite` and
lint, and opens a pull request with the new bundle, `package.json` and
`package-lock.json`. It needs "Allow GitHub Actions to create and approve pull
requests" enabled under Settings → Actions → General; otherwise it pushes the
`ignite/<version>` branch for you to open the pull request.

**Locally**, with registry access, after changing the `@ignite/web` version:

```sh
npm install
npm run build:ignite
```

`build:ignite` copies the package's `dist`, bundles every component with
esbuild (keeping the Font Awesome imports external), and derives
`themes/tokens.css`. Commit the regenerated `scripts/ignite/bundle`. The
committed bundle is built from `@ignite/web` 0.34.0.

## Using a component in a block

```js
import loadIgnite from '../../scripts/components/ignite.js';

export default async function decorate(block) {
  const button = document.createElement('xe-button');
  button.setAttribute('variant', 'primary');
  button.setAttribute('href', '/programs');
  button.textContent = 'Explore Programs';
  block.replaceChildren(button);
  await loadIgnite(() => import('../../scripts/ignite/bundle/primitives/action/button/xe-button.js'));
}
```

Each bundle is self-contained: it inlines Lit and the primitives it uses, and
registers them with `customElements.define()`. Loading two bundles would
normally throw on the second registration of a shared element. `ignite.js`
handles that before any bundle loads:

- it registers the local `<xe-icon>` first, so every icon (including those
  Ignite renders inside its shadow roots) comes from the site's icon set;
- it makes `customElements.define()` skip names already registered (first
  registration wins) instead of throwing;
- it loads the design tokens (`themes/tokens.css`: the `--xe-*` custom
  properties, without the theme's global element resets).

`loadIgnite()` resolves `null` if the bundle fails to load, so a block can keep
its fallback content. For blocks above the fold, don't await it, so the page
renders without waiting for the component.

The bundles keep bare imports such as `@fortawesome/pro-regular-svg-icons`; the
import map in `head.html` resolves them to `fontawesome-shim.js`. It carries the
`aem` nonce the Content Security Policy requires and must stay before the
module scripts. To use more Font Awesome icons, add them to `icons.js` and
export them from the shim.
