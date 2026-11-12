# OpenPigeonBot (OPB)

OpenPigeonBot is StephenFeng's downstream build of
[OpenMausBot](https://github.com/milind-soni/OpenMausBot) (Apache-2.0). It is
not affiliated with the OpenMausBot maintainers, MausBot Cloud, or xAI.

## Branches and remotes

| Name | Meaning |
| --- | --- |
| `origin` | `https://github.com/5tephenFeng/OpenPigeonBot` (this fork, public) |
| `upstream` | `https://github.com/milind-soni/OpenMausBot` |
| `main` | Mirrors `upstream/main`; never commit OPB changes here |
| `opb/branding` | OPB branding and fork-only changes, rebased or merged onto `main` |

Sync: `git fetch upstream && git checkout main && git merge --ff-only upstream/main && git push origin main`,
then bring `opb/branding` up to date with `main` and re-run the checks below.

## Compatibility contract

OPB must keep working with the official iOS, Android and desktop clients.
Do not change: HTTP/WebSocket routes and payloads, the companion pairing
protocol and pairing link / QR format, the `openmausbot://` URL scheme and
app-link domains, the `_openmausbot._tcp` Bonjour type, default ports
(8799/8800/8810/8811…), `OMB_*` environment variables, `config.json` keys,
MCP/ACP protocols, npm package names other scripts depend on, the database
schema, or the data directory `~/.openmausbot`.

Branding lives in: `src/lib/brand.ts` (`PRODUCT_NAME`, `DEFAULT_BRAND`),
`src/lib/app-links.ts`, `index.html`, `rail.html`, `public/manifest.webmanifest`,
`electron-builder.yml` (appId `com.stephenfeng.openpigeonbot`, productName,
artifact names, macOS usage strings, publish target = this fork),
`package.json` (`productName`, links), and the desktop window/tray titles.

## Build on macOS (Apple silicon), unsigned

```sh
pnpm install            # Node >= 24, pnpm 10.33 via packageManager
pnpm typecheck
pnpm test               # or a subset: pnpm exec vitest run src/lib
pnpm build && pnpm build:server && pnpm build:companion
CSC_IDENTITY_AUTO_DISCOVERY=false pnpm exec electron-builder --mac dir --arm64 \
  -c.mac.identity=null --publish never
```

Never sign, notarize or publish OPB artifacts with the upstream identity or
to the upstream release repository. Build output (`dist*`, `release/`,
`node_modules/`) must stay out of Syncthing; see the `.stignore` of the
synced AI folder.
