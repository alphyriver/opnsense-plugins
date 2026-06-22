# Consolidated plugin feed

This directory holds the metadata for the **aggregate** alphyriver OPNsense pkg
feed. The feed bundles plugins that are built and released in their own canonical
repos into a single signed pkg repository, so a firewall adds **one** repo conf +
**one** key and can install every plugin.

The feed is (re)built by [`.github/workflows/aggregate.yml`](../../.github/workflows/aggregate.yml):
it reads [`plugins.yaml`](../../plugins.yaml), downloads each plugin's released
`.pkg` from its GitHub Release, re-generates an **RSA-signed pkg repository** over
all of them, and publishes it to **GitHub Pages** (`gh-pages` branch).

## Files here

| File | Purpose |
|------|---------|
| `pkg-repo.pub` | Public half of the **aggregate** repo signing key. **Committed.** Pinned on nodes as `/usr/local/etc/pkg/keys/alphyriver.pub`. |
| `plugins.conf.in` | OPNsense repo conf template. CI substitutes `@PAGES_URL@` and publishes the result at `<pages-url>/plugins.conf`. |
| `index.html.in` | Landing page template (CI substitutes the package list, ABI, key fingerprint). |

The private signing key is **never** committed — it lives only in the
`PKG_SIGNING_KEY` repository secret. This aggregate key is distinct from the
per-plugin signing keys in the source repos; the combined catalogue is re-signed
under it, so nodes pin only this one key.

> **Why re-signing is safe.** OPNsense's `signature_type: pubkey` signs the repo
> **catalogue** (which records each package's checksum), not each `.pkg`
> individually. Running `pkg repo <dir> rsa:<key>` over downloaded packages
> produces a fresh catalogue signed by the aggregate key — identical in kind to
> how each source repo signs its own feed.

## One-time setup

1. **Aggregate signing key.** Generate an RSA keypair and keep the private key
   safe (offline):
   ```sh
   openssl genrsa -out alphyriver-pkg-signing.key 4096
   openssl rsa -in alphyriver-pkg-signing.key -pubout -out deploy/repo/pkg-repo.pub
   ```
   Commit `deploy/repo/pkg-repo.pub`; add the **private** key as the
   `PKG_SIGNING_KEY` Actions secret (Settings → Secrets and variables → Actions).
   To rotate, replace both and re-publish the public key to every node.

2. **GitHub Pages.** Settings → Pages → Source: **Deploy from a branch** →
   `gh-pages` / `root`. (The branch is created by the first feed build.)

3. **Release read access.** The workflow uses `gh release download` against each
   repo in `plugins.yaml`. The default `GITHUB_TOKEN` can read releases of repos
   under the same owner with matching visibility. If a source repo is private or
   under a different owner, add a fine-grained PAT (contents: read on those
   repos) as a secret and pass it to the download step.

## Updating the feed

- **Add a plugin:** append an entry to `plugins.yaml` (`repo`, `tag`, `asset`).
- **Bump a plugin:** change its `tag` in `plugins.yaml`.
- **Rebuild:** push the change (the workflow triggers on `plugins.yaml` /
  `deploy/repo/**`), or run it by hand: `gh workflow run aggregate.yml`.

On each run, `aggregate.yml`:

1. reads `plugins.yaml` (ABI + list of `repo`/`tag`/`asset`),
2. downloads each plugin's `.pkg` from its GitHub Release into `site/<ABI>/`,
3. re-generates + RSA-signs the catalogue over all packages
   (`pkg repo site/<ABI> rsa:<key>` on a FreeBSD VM),
4. renders `plugins.conf` + `pkg-repo.pub` + a landing page at the Pages root,
5. publishes `site/` to `gh-pages`.

## Installing on a node

```sh
fetch -o /usr/local/etc/pkg/repos/alphyriver.conf <pages-url>/plugins.conf
fetch -o /usr/local/etc/pkg/keys/alphyriver.pub   <pages-url>/pkg-repo.pub
pkg update
pkg install os-oidc os-truenas-api-client
```

## Notes

- The feed directory is keyed by pkg ABI (e.g. `FreeBSD:14:amd64`), set by `abi`
  in `plugins.yaml`. All bundled plugins must publish a `.pkg` for that ABI.
- The repo conf uses `signature_type: pubkey`; a node only trusts packages whose
  catalogue verifies against the pinned `alphyriver.pub`.
- The feed carries exactly the manifest-pinned version of each plugin. That is
  enough for `pkg upgrade` (the catalogue advertises a newer version than what is
  installed). To keep older versions installable for rollback, list multiple
  tags for a plugin.
