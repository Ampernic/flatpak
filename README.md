# Ampernic Flatpak repository

A personal, multi-app Flatpak repository served as a static OSTree repo over GitHub
Pages at **https://flatpak.ampernic.space/** (behind Cloudflare).

This repo is **app-agnostic**: it knows nothing about individual apps. Each app
publishes *itself* here from its own CI via `repository_dispatch`. This repo only
builds the manifest it is handed, signs it, and serves the result.

## Use it

```sh
flatpak remote-add --if-not-exists --user \
  ampernic https://flatpak.ampernic.space/ampernic.flatpakrepo
flatpak remote-ls --user ampernic
flatpak install --user ampernic <app-id>          # stable (default branch)
flatpak install --user ampernic <app-id>//devel   # development build
```

## How it works

- **State** lives on the `gh-pages` branch: the whole OSTree repository, one
  squashed (force-pushed) commit so the branch never accumulates history while its
  content accumulates apps and builds. GitHub Pages serves that branch.
- **Publishing** is driven by [`.github/workflows/publish.yml`](.github/workflows/publish.yml),
  triggered by `repository_dispatch` (`type: publish`) or run manually. Payload:
  - `app_id` — for logging / build dir naming
  - `channel` — `stable` or `devel` (becomes the OSTree branch via `--default-branch`)
  - `manifest_b64` — base64 of a **self-contained** Flatpak manifest (git sources
    with explicit commits); manual runs use `manifest_url` instead
  - `runtime_version` — GNOME runtime the manifest needs
  The workflow clones the existing store, builds into it, regenerates metadata +
  static deltas, GPG-signs, overlays the static site + `.flatpakrepo`, and force-pushes.
- **Signing**: a dedicated ed25519 key. Private key in the `FLATPAK_GPG_PRIVATE_KEY`
  Actions secret; public key embedded in [`remote/ampernic.flatpakrepo`](remote/ampernic.flatpakrepo).
  Rotating it forces clients to re-add the remote — it is meant to be long-lived.
- **`.nojekyll`** is required so Pages keeps the OSTree `objects/` and `deltas/` trees.

## Onboarding a new app

The app stays in full control; this repo gains nothing app-specific. In the app's
own repository:

1. Add a self-contained Flatpak manifest (git `sources` with `commit:` pins, not
   local `path:` sources). Use `__*_COMMIT__` placeholders if CI resolves refs.
2. Create a `FLATPAK_DISPATCH_TOKEN` secret — a token with `contents:write` on
   `Ampernic/flatpak` (fine-grained, scoped to this repo only).
3. Add a workflow that, on push to `main` (→ `devel`) and on tags (→ `stable`),
   resolves commits, renders the manifest, base64-encodes it, and dispatches:

   ```sh
   curl -fsS -X POST \
     -H "Authorization: Bearer $FLATPAK_DISPATCH_TOKEN" \
     -H "Accept: application/vnd.github+json" \
     https://api.github.com/repos/Ampernic/flatpak/dispatches \
     -d "$(jq -n --arg id "$APP_ID" --arg ch "$CHANNEL" \
              --arg m "$(base64 -w0 manifest.yml)" --arg rt '49' \
              '{event_type:"publish",
                client_payload:{app_id:$id,channel:$ch,manifest_b64:$m,runtime_version:$rt}}')"
   ```

See `another-tgproxy` for a working reference.

## Limits / maintenance

- GitHub Pages: repo soft-limit ~1 GB, ~100 GB/month bandwidth — Cloudflare in front
  caches objects so origin pulls stay low. `--prune --prune-depth=20` bounds growth.
- All publishes serialize (`concurrency: flatpak-publish`) since they share one store.
