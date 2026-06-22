# FM2K Rollback Launcher binaries

Public release artifacts for [FM2K Rollback](https://www.patreon.com/Armonte).
This repo only hosts compiled binaries; source lives in private development repos.

The launcher's auto-updater reads `LatestVersion` from this repo's `main` branch
and downloads the matching `fm2k_v<version>.zip` from GitHub Releases.

- **LatestVersion** — single line, current published version code (e.g. `0.1.0`).
- **Releases** — each tagged `v<version>` with `fm2k_v<version>.zip` attached.

---

The launcher UI is built with [Slint](https://slint.dev) under the Slint
Royalty-free Desktop, Mobile, and Web Applications License.

<a href="https://slint.dev">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://github.com/slint-ui/slint/raw/master/logo/MadeWithSlint-logo-light.svg">
    <img alt="Made with Slint" src="https://github.com/slint-ui/slint/raw/master/logo/MadeWithSlint-logo-dark.svg" width="200">
  </picture>
</a>

