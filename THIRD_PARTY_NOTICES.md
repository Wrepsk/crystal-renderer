# Third-party notices

CrystalRenderer is licensed under AGPL-3.0-or-later; see [LICENSE.md](LICENSE.md). Dependencies retain their respective licenses.

## Derivation from Pictomancy

CrystalRenderer includes a focused derivative of the screen-rendering code from [sourpuh/ffxiv_pictomancy](https://github.com/sourpuh/ffxiv_pictomancy) and the CrystalCast-specific [Wrepsk fork](https://github.com/Wrepsk/ffxiv_pictomancy), extracted at commit `ed205c725fe1d17124480823eac7affaccd669ee`. Original authorship and inline credits are retained. Ported files are marked with attribution; the upstream AGPL-3.0-or-later license text is included at [CrystalRenderer/LICENSE.Pictomancy](CrystalRenderer/LICENSE.Pictomancy).

The relevant CrystalRenderer and CrystalRenderer.Tests history was extracted from [Wrepsk/crystal-cast](https://github.com/Wrepsk/crystal-cast) at commit `a04e327`. Filtered commit IDs differ from the source repository; original authors, dates, and messages are retained.

## Dependencies

The library uses SharpDX and KamiToolKit. Exact versions and integrity hashes are recorded in [CrystalRenderer/packages.lock.json](CrystalRenderer/packages.lock.json). KamiToolKit brings ImageSharp transitively; the existing audit findings and the original CrystalCast investigation are retained in [docs/DEPENDENCY_AUDIT.md](docs/DEPENDENCY_AUDIT.md). CrystalRenderer has no direct ImageSharp API calls or package reference.
