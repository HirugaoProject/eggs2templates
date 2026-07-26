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
