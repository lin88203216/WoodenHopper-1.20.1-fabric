# Wooden Hopper — Unofficial Fabric 1.20.1 Port

An **unofficial** backport of [cech12/WoodenHopper](https://github.com/cech12/WoodenHopper) from Minecraft 1.20.4 to **Minecraft 1.20.1 (Fabric)**.

> Not affiliated with, endorsed by, or supported by the original author. If you are looking for the official release, use the upstream project.

## About

The Wooden Hopper is a vanilla-like hopper that moves items at half the speed of a normal hopper (16 ticks instead of 8, configurable) and holds a single slot.

This repository is a version port only. The code was taken from upstream `1.20.4-2.2.2.0` and adapted to the 1.20.1 API surface; the Forge and NeoForge modules were dropped, leaving `common` + `fabric`. Gameplay, balancing, recipes, models, textures, sounds and translations are unchanged. AI assistance was used during the porting work.

- Minecraft: **1.20.1** only (`>=1.20.1 <1.20.2`)
- Loader: **Fabric**
- Mod id: `woodenhopper`

## Requirements

| Mod | Version |
|---|---|
| [Fabric Loader](https://fabricmc.net/use/) | 0.16.10 or newer |
| [Fabric API](https://modrinth.com/mod/fabric-api) | 0.92.x for 1.20.1 (`0.92.12+1.20.1` used during development) |
| [Cloth Config API](https://modrinth.com/mod/cloth-config) | 11.1.x for 1.20.1 (`11.1.136` used during development) |
| [ModMenu](https://modrinth.com/mod/modmenu) | 7.2.x — optional, only needed for the in-game config screen |

Use the **Fabric** builds of Fabric API and Cloth Config, not the Forge ones. Versions from the 1.20.4 line (Fabric API 0.96+, Cloth Config 13+) will not work.

## Installation

1. Install Minecraft 1.20.1 with Fabric Loader 0.16.10 or newer.
2. Drop the following into `.minecraft/mods/`:
   - `woodenhopper-fabric-1.20.1-2.2.2.0.jar` (this port)
   - Fabric API
   - Cloth Config API
   - ModMenu (optional)
3. Launch the game. The mod refuses to load on any Minecraft version other than 1.20.1.

Configuration is available at `config/woodenhopper.toml`, or in game through ModMenu:

- hopper cooldown in ticks (1–1000, default 16)
- whether the hopper pulls item entities from the world
- whether the hopper pulls items from inventories above it

## Building from source

```bash
./gradlew :fabric:build
# output: fabric/build/libs/woodenhopper-fabric-1.20.1-2.2.2.0.jar
```

Requires JDK 17. See [PORTING-NOTES.md](PORTING-NOTES.md) for the exact list of changes made for 1.20.1.

## Issues

Please report problems with **this port** here, in this repository. Do not report port-specific issues to the upstream project; the original author is not responsible for this build.

## Credits

- **cech12** — original mod, code, art and translations: https://github.com/cech12/WoodenHopper
- All translators listed in the upstream project

## License

MIT License. The original copyright notice (`Copyright (c) 2021 cech12`) is preserved. Changes made for this port are released under the same license — see [LICENSE](LICENSE) for the full text and the port statement.
