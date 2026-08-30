# PufferPanel templates

Docker-only PufferPanel v3 templates converted from the Pterodactyl eggs in this
repository. The directory layout mirrors the egg layout, so
`minecraft/java/paper/egg-paper.json` becomes
`pufferpanel/minecraft/java/paper/paper.json`.

Every template validates against the official
[PufferPanel template schema](https://github.com/pufferpanel/templates/blob/v3/spec.json).

## Design rules

These rules are applied to every template in this directory.

### Docker only, and no package installation

`environment` and `supportedEnvironments` only ever contain a `docker` entry.
The `host` environment is deliberately not supported.

PufferPanel runs install steps in the **same** container as the server, as the
panel's own unprivileged user. That means `apt`, `apk` and friends cannot work
even if a template asked for them, so no template does. Everything an egg's
install script did with `curl`, `wget`, `jq`, `unzip` or `tar` is instead done
with PufferPanel's own operations, which run on the daemon and need nothing
inside the image:

| Pterodactyl install script | PufferPanel operation |
| --- | --- |
| `curl -o file URL` / `wget URL` | `download` |
| `unzip` / `tar -xf` | `extract` (zip, tar, tar.gz, tar.bz2, tar.xz) |
| `mkdir -p` | `mkdir` |
| `mv` | `move` |
| `cat > file <<EOF` | `writefile` |
| `sed -i` on a config file | `alterfile` with `regex: true` |

`command` operations are only used where something genuinely has to run inside
the container: SteamCMD, `chmod`, `cp`, Wine, and language runtimes such as
`npm`, `dotnet` and `java`. Every binary they invoke is already present in the
image the template selects.

### Version resolution without `jq`

Many eggs shell out to `curl … | jq` against the GitHub API to resolve "latest".
That needs `jq` in the container, so it is replaced by one of:

* GitHub's stable `releases/latest/download/<asset>` redirect, when the asset
  name does not contain the version;
* an explicit, pinned version variable when it does — which is also the safer
  choice for a production server, because an upstream release cannot silently
  change what a reinstall produces;
* a full `DOWNLOAD_URL` variable when upstream publishes no predictable URL at
  all (for example DDNet and Zandronum, which are scraped from a download page
  by the original egg).

### Shell features

PufferPanel executes `run.command` and `command` operations directly, without a
shell, so `&&`, `|`, `$(…)`, backgrounding and globbing do not work by default.
Where an egg's startup line genuinely needs them the command is wrapped in
`bash -c '…'` and the server process is started with `exec`, so it becomes the
container's main process and still receives the stop signal correctly.

Most of the time the shell is not needed at all, because PufferPanel has native
equivalents for what the eggs were emulating:

* eggs that background the server and then attach `telnet` (7 Days to Die,
  Empyrion) use `stdin: {"type": "telnet"}`;
* eggs that background the server and poll `rcon` (both ARK templates) use
  `stdin: {"type": "rcon"}` and a real console command as `run.stop`;
* eggs that background the server and `tail -F` a log file run the server in the
  foreground with its log directed at the console instead.

### Quoting in `run.command`

PufferPanel substitutes variables into `run.command` with `ShellReplace`, which
runs every value through `shellwords.Quote` before splitting the line. A value
that contains spaces therefore arrives already quoted, and writing
`-name="${SERVER_NAME}"` in a template yields the literal argument
`-name="A Server"` — quotes included. Variables in `run.command` are written
bare (`-name=${SERVER_NAME}`); the quoting PufferPanel adds is what keeps the
value a single argument, and an empty value still survives as an empty argument.

Inside `bash -c '…'` the rule does not apply: the shell parses the line a second
time and collapses the added quotes, so quoting there is written normally. This
is also why the one egg that passes a whole config script as a single argument
(Hurtworld's `-exec "host …;servername …"`) is wrapped in `bash -c`, since only
a shell can re-join the quoted value into the surrounding word.

`install` steps and `run.pre` operations use plain substitution instead, so
values there are quoted in the template where they need to be.

### Stopping the server

`config.stop` maps to `run.stop` for console commands, and to `run.stopCode` for
signals — `^C` becomes `stopCode: 2` (SIGINT) and `^SIGKILL` becomes
`stopCode: 9`.

### Configuration files

Pterodactyl rewrites configuration files from panel variables on every boot.
PufferPanel has no equivalent, so the same effect is produced explicitly:

* the file is created during `install` with `writefile` (or downloaded, or
  copied from the template the game ships), guarded by
  `if: "!file_exists(…)"` so an existing configuration is never destroyed;
* the settings that must follow the panel variables — the listen port above all
  — are re-applied on every boot from `run.pre` with `alterfile`, guarded by
  `if: "file_exists(…)"` so a missing file cannot block a start.

Everything else in those files stays editable through the panel's file manager,
which is the normal PufferPanel workflow.

### Variables

Egg variable names are kept verbatim (`SRCDS_APPID`, `MAX_PLAYERS`, …) so the
mapping back to the original egg stays obvious. The one exception is the primary
bind port, which is named `port` because PufferPanel itself reads `port` (and
`ip`) from a server's data to display and query it.

Ports the game derives by arithmetic (`port + 1`, `port + 2`) become their own
variables, because port bindings cannot compute — their descriptions say what
they have to be set to.

`AUTO_UPDATE` from the Steam eggs becomes `UPDATE_ON_BOOT`, implemented as a
`run.pre` SteamCMD step, which is how the official PufferPanel Squad template
does it. It defaults to on except where re-running SteamCMD would overwrite
patched binaries (Counter-Strike 1.6 ReHLDS) or needs credentials that may have
expired (Carrier Command 2).

### Images

The image an egg uses at runtime is kept, unless SteamCMD has to run in it.
SteamCMD is a 32-bit binary, so a Steam game whose egg ran on `yolks:debian`
(64-bit only) is moved to `steamcmd:debian`, and a Steam game that also needs
.NET uses `steamcmd:dotnet`. Wine and Proton templates set `WINEPREFIX`,
`STEAM_COMPAT_*` and `USER` explicitly, because the yolks entrypoint that
normally sets them is replaced by PufferPanel's own command.

## Caveats

* **SuperTuxKart** is built from source by its egg, which needs a compiler and
  a dozen `-dev` packages. It is converted to the official upstream Linux
  release archive instead. That build is not `SERVER_ONLY`, so it may need
  graphics libraries the chosen image does not carry.
* **Arma 3** and **DayZ** get the server install, configuration and startup, but
  not the Steam Workshop mod manager from the `games:arma3` / `games:dayz`
  entrypoints — that is several hundred lines of shell around a per-mod loop.
  Mod folders can still be uploaded and listed in the mod variables.
* **Counter-Strike 1.6 ReHLDS** installs ReHLDS and ReGameDLL_CS straight from
  their own upstream releases rather than from the third-party bundle the egg
  uses, because the bundle's installer needs `unzip` and shell loops inside the
  container. AMX Mod X, Metamod-R, ReAPI and Reunion are not installed.
* **CS2D** and **Foundry VTT** need a download URL that upstream only publishes
  on a web page (CS2D) or per-licence (Foundry VTT); both are exposed as
  variables.
* **Factorio with Mod Updater** installs the `requests` Python module with
  `pip install --user` into the server's own directory. Nothing is installed
  into the image.
* **Valheim** (all three templates) drops the egg's `sed` console filter. The
  filter only trims Unity noise, but piping the output puts a shell between the
  daemon and the game, and Valheim only saves when it receives SIGINT itself.
  `stopCode: 2` now delivers that signal directly, which is worth more than the
  tidier log. The BepInEx template also leaves out the egg's Thunderstore
  modpack installer, which needs the Thunderstore API and `jq`; BepInEx itself
  is installed, and plugin DLLs can be uploaded to `BepInEx/plugins`.
* **StarRupture** hashes its admin and player passwords through
  `starrupture-utilities.com`, which is the only published way to produce the
  `Password.json` files the server reads. That request only happens when a
  password variable is filled in, and it sends the password to that third party.
  Leave both empty to skip it entirely.
* **Urban Terror** replaces the official updater (which needs `xmllint`
  installed into the OS) with the full release archive, and **ET Legacy**
  replaces the 2.60b self-extracting installer with the retail `pk3` files that
  ET: Legacy mirrors directly.
* **Xonotic** leaves out the `rsync`-based autobuild updater the egg ran, since
  it needs `rsync` in the image. The release archive is a complete server.
* **Space Station 14** still scrapes the build number out of the Wizard's Den
  fork page, because that is the only place it is published; a `Build Version`
  variable pins it when that page changes shape.
* **Swords 'n Magic and Stuff** applies its panel variables to the `Game.ini`
  the game ships by key name. The egg never wrote them anywhere, and the key
  names are not documented, so a key that does not match is simply left alone.
* **Subnautica (Nitrox)** downloads the game with `steamgamedl` rather than
  SteamCMD, because Nitrox needs the .NET 9 runtime and no image carries both
  that and the 32-bit libraries SteamCMD needs. The downloader authenticates
  interactively when Steam Guard is on, and an install has no console to type a
  code into, so the egg's `STEAM_GUARDCODE` variable is gone and the account
  used must not have Steam Guard enabled.

## Coverage

All 254 eggs in this repository are converted, in 253 templates. The one place where the
counts differ is `minecraft/proxy/bedrock/waterdog_pe`, whose two eggs are byte-identical
apart from the list of Java images they offer; the single template covers both because it
takes the Java version as a variable.

Every directory below is converted.

* `7_days_to_die` (1)
* `abiotic_factor` (1)
* `aloft` (1)
* `americas_army/proving_grounds` (1)
* `among_us/bettercrewlink_server` (1)
* `among_us/impostor_server` (1)
* `Archean` (1)
* `ark_survival_ascended` (1)
* `ark_survival_evolved` (1)
* `arma/arma3` (1)
* `arma/arma_reforger` (1)
* `Aska` (1)
* `assetto_corsa` (2)
* `astro_colony` (1)
* `astroneer` (1)
* `automobilista2` (1)
* `avorion` (1)
* `banana_shooter` (1)
* `barotrauma` (1)
* `battalion_legacy` (1)
* `beamng/beammp` (1)
* `beamng/kissmp` (1)
* `black_mesa` (1)
* `brickadia` (1)
* `carrier_command_2` (1)
* `citadel` (1)
* `classicube/mcgalaxy` (1)
* `clone_hero` (1)
* `cod/iw4x` (1)
* `colony_survival` (1)
* `conan_exiles` (2)
* `contagion` (1)
* `core_keeper` (1)
* `counter_strike/counter_strike_1.6` (1)
* `counter_strike/counter_strike_1.6_rehlds` (1)
* `counter_strike/counter_strike_2` (1)
* `counter_strike/counter_strike_source` (1)
* `craftopia` (1)
* `cryofall` (1)
* `cs2d` (1)
* `cubeengine/assaultcube` (1)
* `cubeengine/cube2` (1)
* `cubic_odyssey` (1)
* `dayz` (1)
* `ddnet` (1)
* `ddracenetwork` (1)
* `dead_matter` (1)
* `dont_starve` (1)
* `doom/zandronum` (1)
* `eco` (1)
* `eft` (1)
* `empyrion` (1)
* `enshrouded` (1)
* `factorio/clusterio` (1)
* `factorio/factorio` (2)
* `factorio/factorio-modupdate` (1)
* `fof` (1)
* `fortresscraft_evolved` (1)
* `foundry` (1)
* `foundry_vtt` (1)
* `frozen_flame` (1)
* `ftl_tachyon` (1)
* `gmod` (1)
* `ground_branch` (1)
* `gta/altv` (1)
* `gta/fivem` (1)
* `gta/gtac` (1)
* `gta/mtasa` (1)
* `gta/openmp` (1)
* `gta/ragecoop` (1)
* `gta/ragemp` (1)
* `gta/samp` (1)
* `half_life_2_deathmatch` (1)
* `hlds_server/rehlds` (1)
* `hlds_server/vanilla` (1)
* `hogwarp` (1)
* `holdfast` (1)
* `humanitz` (1)
* `hurtworld` (1)
* `hytale` (1)
* `icarus` (1)
* `insurgency_sandstorm` (1)
* `iosoccer` (1)
* `just_cause/just_cause_3` (1)
* `killing_floor_2` (1)
* `ksp/DMP` (1)
* `ksp/LMP` (1)
* `League Sandbox` (1)
* `left4dead` (1)
* `left4dead_2` (1)
* `longvinter` (1)
* `losangelescrimes` (1)
* `midnight_ghost_hunt` (1)
* `mindustry` (1)
* `minecraft/bedrock/bedrock` (2)
* `minecraft/bedrock/gomint` (1)
* `minecraft/bedrock/LeviLamina` (1)
* `minecraft/bedrock/LiteLoader-bedrock` (1)
* `minecraft/bedrock/nukkit` (1)
* `minecraft/bedrock/pocketmine_mp` (1)
* `minecraft/bedrock/PowerNukkitX` (1)
* `minecraft/crossplay/purpur-geysermc-floodgate` (1)
* `minecraft/java/arclight` (1)
* `minecraft/java/canvas-mc` (1)
* `minecraft/java/cuberite` (1)
* `minecraft/java/curseforge` (1)
* `minecraft/java/fabric` (1)
* `minecraft/java/folia` (1)
* `minecraft/java/forge` (1)
* `minecraft/java/ftb` (1)
* `minecraft/java/glowstone` (1)
* `minecraft/java/krypton` (1)
* `minecraft/java/limbo` (1)
* `minecraft/java/magma` (1)
* `minecraft/java/modrinth` (1)
* `minecraft/java/mohist` (1)
* `minecraft/java/nanolimbo` (1)
* `minecraft/java/neoforge` (1)
* `minecraft/java/paper` (1)
* `minecraft/java/purpur` (1)
* `minecraft/java/quilt` (1)
* `minecraft/java/spigot` (1)
* `minecraft/java/spongeforge` (1)
* `minecraft/java/spongevanilla` (1)
* `minecraft/java/technic/attack-of-the-bteam` (1)
* `minecraft/java/technic/blightfall` (1)
* `minecraft/java/technic/hexxit` (1)
* `minecraft/java/technic/Tekkit` (1)
* `minecraft/java/technic/Tekkit-2` (1)
* `minecraft/java/technic/tekkit-classic` (1)
* `minecraft/java/technic/tekkit-legends` (1)
* `minecraft/java/technic/tekkit-smp` (1)
* `minecraft/java/technic/the-1-12-2-pack` (1)
* `minecraft/java/technic/the-1-7-10-pack` (1)
* `minecraft/java/vanillacord` (1)
* `minecraft/proxy/bedrock/waterdog_pe` (2) — 1 template; the eggs there are duplicates of one another
* `minecraft/proxy/java/travertine` (1)
* `minecraft/proxy/java/velocity` (1)
* `minecraft/proxy/java/viaaas` (1)
* `minecraft/proxy/java/waterfall` (1)
* `minetest` (1)
* `modiverse` (1)
* `mohaa` (1)
* `mordhau` (2)
* `mount_blade_II_bannerlord` (1)
* `myth_of_empires` (1)
* `Nazi Zombies Portable` (1)
* `necesse` (1)
* `neosvr` (1)
* `neverwinter_nights_ee` (1)
* `night_of_the_dead` (1)
* `Nightingale` (1)
* `nmrih` (1)
* `no_love_lost` (1)
* `no_one_survived` (1)
* `novalife_amboise` (1)
* `nuclear_option` (1)
* `onset` (1)
* `open_fortress` (1)
* `openarena` (1)
* `openra/openra_dune2000` (1)
* `openra/openra_red_alert` (1)
* `openra/openra_tiberian_dawn` (1)
* `openrct2` (1)
* `openttd` (1)
* `operation_harsh_doorstop` (1)
* `palworld` (2)
* `path_of_titans` (1)
* `pavlov_vr` (1)
* `pixark` (1)
* `plains_of_pain` (1)
* `portal_knights` (1)
* `post_scriptum` (1)
* `project_zomboid` (1)
* `puck` (1)
* `quake_live` (1)
* `r5reloaded` (1)
* `rdr/redm` (1)
* `renown` (1)
* `resonite` (1)
* `return_to_moria` (1)
* `rimworld/open_world` (1)
* `rimworld/together` (1)
* `rising_world/legacy` (1)
* `rising_world/unity` (1)
* `risk_of_rain_2` (1)
* `romestead` (1)
* `rust/rust_autowipe` (1)
* `rust/rust_staging` (1)
* `satisfactory` (1)
* `scpsl/dedicated` (1)
* `scpsl/exiled` (1)
* `scum` (1)
* `smalland_survive_the_wilds` (1)
* `solace_crafting` (1)
* `soldat` (1)
* `soldat_2` (1)
* `sonic_robo_blast_2` (1)
* `sonsoftheforest` (1)
* `soulmask` (1)
* `sourcecoop` (1)
* `space_engineers/default` (1)
* `space_engineers/torch` (1)
* `spacestation_14` (1)
* `squad` (1)
* `starbound` (1)
* `starmade` (1)
* `starrupture` (1)
* `stationeers/stationeers_bepinex` (1)
* `stationeers/stationeers_vanilla` (1)
* `stormworks` (1)
* `subnautica_nitrox_mod` (1)
* `sunkenland` (1)
* `SuperTuxKart` (1)
* `svencoop` (1)
* `swords_'n_Magic_and_Stuff` (1)
* `team_fortress_2` (1)
* `team_fortress_2_classic` (1)
* `teeworlds` (1)
* `terraria/tmodloader` (1)
* `terraria/tshock` (2)
* `terraria/vanilla` (1)
* `terratech_worlds` (1)
* `the_forest` (1)
* `the_isle/evrima` (1)
* `thefront` (1)
* `tower_unite` (1)
* `trackmania` (1)
* `truck-simulator/american-truck-simulator` (1)
* `truck-simulator/euro-truck-simulator2` (1)
* `unturned` (1)
* `urbanterror` (1)
* `v_rising/v_rising_bepinex` (1)
* `v_rising/v_rising_vanilla` (1)
* `valheim/valheim_bepinex` (1)
* `valheim/valheim_plus` (1)
* `valheim/valheim_vanilla` (1)
* `vein` (1)
* `veloren` (1)
* `vintage_story` (1)
* `voyagers_of_nera` (1)
* `windrose` (1)
* `wine/generic` (1)
* `wolfenstein_enemy_territory/etlegacy` (1)
* `wurm_unlimited` (1)
* `xonotic` (1)

## What is checked

`pufferpanel/` is validated as a whole, not template by template. Every file has
to pass, and the checks are:

* the official `spec.json` schema;
* `docker` is the only environment, in both `environment` and
  `supportedEnvironments`;
* every `${TOKEN}` resolves to a declared variable, and every declared variable
  is referenced somewhere — including bare identifiers inside CEL `if`
  expressions, which is how the boolean variables are used;
* no `apt`/`apk`/`yum`/`dnf` install, and `pip` only with `--user`;
* no `${VAR}` wrapped in quotes inside `run.command`, since PufferPanel quotes
  the value itself (see *Quoting in `run.command`* above);
* every `bash -c` script parses under `bash -n`;
* SteamCMD only appears in images that carry i386 libraries, because it is a
  32-bit binary. Where the runtime rules those images out — Rising World Java
  Legacy needs a JRE, Subnautica needs the .NET 9 runtime — the template uses
  the `steamgamedl` operation instead, which runs the daemon's own 64-bit
  DepotDownloader;
* port bindings are well formed.

Beyond that, the finished set was audited for a handful of things a schema
cannot see: that every `extract` names an archive some earlier step actually
produces under that name, that CEL conditions match the type of the variable
they test (booleans bare, strings compared against quoted literals), that every
conditional `run.command` list has a branch that always matches, and that each
declared port is either bound or deliberately left unbound.
