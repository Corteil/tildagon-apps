# Tildagon Apps

An index of [EMF Tildagon badge](https://tildagon.badge.emfcamp.org/) apps by
[Brian Corteil](https://github.com/Corteil).

The Tildagon app store discovers one app per repository (it reads `tildagon.toml`
from the repo root), so each published app lives in its own repo. This repo is a
landing page that links to them.

## Apps

| App | Repo | App store | Description |
|-----|------|-----------|-------------|
| **GPS Info** | [tildagon-gps-info](https://github.com/Corteil/tildagon-gps-info) | [directory](https://apps.badge.emfcamp.org/) | GPS status, satellite count and a sky map of the satellites in view, for the GPS Hexpansion. Round-screen UI with an LED status ring. |

## Publishing notes

To get an app into the [app store](https://apps.badge.emfcamp.org/):

1. App in its own public repo with `app.py` + `tildagon.toml` **at the root**.
2. Add the **`tildagon-app`** topic to the repo.
3. Create a **GitHub release** (e.g. `v0.0.1`).
4. The store auto-discovers it within ~15 minutes.

See the [publish docs](https://tildagon.badge.emfcamp.org/tildagon-apps/publish/).

## Credits

Built on the EMF Tildagon ecosystem:

* [emfcamp/badge-2024-software](https://github.com/emfcamp/badge-2024-software) — Tildagon OS.
* [mbooth101/emf-speedometer](https://github.com/mbooth101/emf-speedometer) — GPS Hexpansion firmware and app patterns.
* [TechCabin/EMFBadge-Hexpansions-GPS](https://github.com/TechCabin/EMFBadge-Hexpansions-GPS) — GPS Hexpansion hardware.

## License

MIT — see [LICENSE](LICENSE).
