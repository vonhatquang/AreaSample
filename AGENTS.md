# AGENTS.md

## Cursor Cloud specific instructions

### What this is
`AreaSample` is a legacy **ASP.NET MVC 5** web app targeting **.NET Framework 4.6.1** (`System.Web`, `packages.config`, IIS-oriented). There is only one project (`AreaSample/AreaSample.csproj`) and one solution (`AreaSample.sln`). There are no automated tests and no lint configuration.

Because it is full .NET Framework (not .NET Core/5+), it is built and hosted on Linux via **Mono**. `dotnet` and `msbuild` are intentionally NOT used here.

### Toolchain (already installed in the VM snapshot)
- `mono` 6.8, `xbuild` (Mono's MSBuild-compatible builder for this old-style project), `mono-xsp4`, and Apache + `mod_mono` (`mono-apache-server4`).
- `nuget.exe` lives at `$HOME/.tools/nuget.exe` and is run via `mono` (there is no native `nuget`/`msbuild` binary).

### Restore + build
- Restore NuGet packages (also done by the startup update script): `mono "$HOME/.tools/nuget.exe" restore AreaSample.sln`
- Build (Debug): `xbuild /p:Configuration=Debug AreaSample.sln` — output goes to `AreaSample/bin/`. A clean build reports 0 errors (a few harmless warnings like `System.Web.Entity not resolved` are expected).

### Run the app (Apache + mod_mono)
- Hosting is via Apache using the vhost at `/etc/apache2/sites-available/areasample.conf`, which maps `/` to `/workspace/AreaSample` and listens on **port 8080** (default site also answers on 80).
- Apache does NOT auto-start (no systemd/init in this VM). Start it manually: `sudo apache2ctl start` (restart: `sudo apache2ctl restart`).
- Verify: `curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8080/` should return `200`; the home page renders the standard MVC "ASP.NET" jumbotron. Routes `/Home/About` and `/Home/Contact` also work.

### Gotchas
- **`xsp4` does not work on Mono 6.8** — it crashes with `TypeLoadException` for `Mono.Security.Protocol.Tls.PrivateKeySelectionCallback`. Use Apache + `mod_mono` (above) instead; `mod-mono-server4` is unaffected.
- After changing C# code, `xbuild` again AND restart Apache (`sudo apache2ctl restart`) so the `mod-mono-server4` worker reloads the rebuilt assembly. `.cshtml` view edits are recompiled on the fly and don't require a rebuild.
- If Apache start fails with a `Failed to create shared memory segment ... /tmp/mod_mono_dashboard_*` crit, remove the stale files: `sudo rm -f /tmp/mod_mono_dashboard_*` then start again.
- `packages/`, `bin/`, and `obj/` are gitignored and are not committed.
