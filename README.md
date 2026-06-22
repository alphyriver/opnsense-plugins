# opnsense-plugins

A single signed FreeBSD **pkg feed** that bundles alphyriver's OPNsense plugins,
so a firewall adds **one** repo config + **one** signing key and can install all
of them.

This repo contains **no plugin source**. Each plugin is built and released in its
own canonical repo; this repo only re-bundles their released `.pkg` artifacts into
one RSA-signed pkg repository published to GitHub Pages.

## Bundled plugins

| Package | Source repo |
|---------|-------------|
| `os-oidc` | [alphyriver/opnsense-oidc](https://github.com/alphyriver/opnsense-oidc) |
| `os-truenas-api-client` | [alphyriver/opnsense-truenas-api-client](https://github.com/alphyriver/opnsense-truenas-api-client) |

Exact pinned versions live in [`plugins.yaml`](plugins.yaml).

## Install on OPNsense (run as root)

```sh
fetch -o /usr/local/etc/pkg/repos/alphyriver.conf https://alphyriver.github.io/opnsense-plugins/plugins.conf
fetch -o /usr/local/etc/pkg/keys/alphyriver.pub   https://alphyriver.github.io/opnsense-plugins/pkg-repo.pub
pkg update
pkg install os-oidc os-truenas-api-client
```

## How it works / maintenance

The aggregate workflow ([`.github/workflows/aggregate.yml`](.github/workflows/aggregate.yml))
reads [`plugins.yaml`](plugins.yaml), downloads each plugin's released `.pkg`,
re-signs the combined catalogue, and publishes the feed.

- **Add a plugin / bump a version:** edit `plugins.yaml` and push (or run the
  workflow manually with `gh workflow run aggregate.yml`).
- **One-time setup & design rationale:** see [`deploy/repo/README.md`](deploy/repo/README.md).
