# Tildagon Apps

A collection of apps for the [EMF Tildagon badge](https://tildagon.badge.emfcamp.org/)
by [Brian Corteil](https://github.com/Corteil).

## Apps

| App | Description |
|-----|-------------|
| [GPS Info](gps-info/) | GPS status, satellite count and a sky map of the satellites in view, for the GPS Hexpansion. Round-screen UI with an LED status ring. |

## Installing an app

Each app is a folder. The Tildagon launcher loads an app as `apps.<folder>.app`
and reads its menu name from `metadata.json`, so an app must be installed as a
**folder** on the badge (a bare `.mpy` file will not load).

Pre-built `.mpy` files are committed, so you can flash without `mpy-cross`. Find
the badge's serial port with `mpremote devs` (substitute it for `COM13` below):

```powershell
# from an app folder, e.g. gps-info
mpremote connect COM13 fs mkdir :/apps/gps_info ^
  + fs cp app.mpy       :/apps/gps_info/app.mpy ^
  + fs cp metadata.json :/apps/gps_info/metadata.json ^
  + fs cp tildagon.toml :/apps/gps_info/tildagon.toml
```

To rebuild from source:

```powershell
pip install mpy-cross
python -m mpy_cross app.py
```

See each app's own `README.md` for app-specific steps (e.g. GPS Info also needs
the GPS Hexpansion firmware flashing — see [gps-info/README.md](gps-info/README.md)).

## Requirements

* **Tildagon OS v2.0.0+** for apps that use hexpansion discovery.

## Credits

Built on the EMF Tildagon ecosystem:

* [emfcamp/badge-2024-software](https://github.com/emfcamp/badge-2024-software) — Tildagon OS.
* [mbooth101/emf-speedometer](https://github.com/mbooth101/emf-speedometer) — GPS Hexpansion firmware and app patterns.
* [TechCabin/EMFBadge-Hexpansions-GPS](https://github.com/TechCabin/EMFBadge-Hexpansions-GPS) — GPS Hexpansion hardware.

## License

MIT — see [LICENSE](LICENSE).
