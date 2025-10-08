## Quick orientation for coding agents

This repository builds the cross-platform PowerShell runtime (native + managed) and many cmdlet modules. The notes below focus on immediately actionable knowledge an AI coding agent needs to be productive here.

### Big picture
- Solution root: `PowerShell.sln`. Major source trees live under `src/`.
- Managed engine & cmdlets: `src/System.Management.Automation`, `src/Microsoft.PowerShell.*` projects.
- Native/host bits: `src/powershell-native`, `src/powershell` (host), and related native helpers.
- SDK / packaging: `src/Microsoft.PowerShell.SDK` and `src/Modules/` for shipped modules.
- Tests: Pester (PowerShell language) under `test/powershell`, xUnit (C#) under `test/xUnit`.

Read these to understand boundaries: `src/System.Management.Automation` (engine), `src/powershell-native` (native runtime), `src/Modules` (module packaging and tests).

### Build / bootstrap / common commands (PowerShell host)
- Bootstrap developer machine and install .NET tooling:

```powershell
Import-Module ./build.psm1; Start-PSBootstrap
```

- Full build (examples):

```powershell
Import-Module ./build.psm1; Start-PSBuild -Configuration Debug -Runtime linux-x64
Import-Module ./build.psm1; Start-PSBuild -Clean
```

- Fast engine-only rebuild: `Start-PSBuild -SMAOnly` (rebuilds only `System.Management.Automation.dll`).
- Use `-UseNuGetOrg` to switch to public nuget.org feeds; otherwise repository uses Microsoft feeds and the repository-specific feed handling in `build.psm1`.
- Dotnet SDK version is pinned in `global.json` and `DotnetRuntimeMetadata.json` — do not change without checking `build.psm1` expectations.
- `Start-PSBootstrap` installs CLI tools to `~/.dotnet` (Linux/macOS) or `$env:LocalAppData\Microsoft\dotnet` (Windows) but does not add them to PATH — add them to PATH in your session if necessary.

### CI / tags / releases
- `build.psm1` exposes `Sync-PSTags`, `Get-PSLatestTag`, and `Get-PSCommitId`. Many scripts expect repository tags to be fetched; call `Sync-PSTags` in scripts that need tags.

### Testing and test structure
- Pester tests: `test/powershell` (PowerShell script tests). xUnit: `test/xUnit` (C# tests).
- Test helper modules are autoloaded from `test/tools/Modules` — the build adds this path to `PSModulePath` during setup (see `build.psm1`).

### Debugging
- Preferred editor: VS Code with C# (OmniSharp) and .NET Core debugger extensions. The repo contains a committed `.vscode/launch.json` and `tasks.json` configured to build and debug (they use `Start-PSBuild`).
- To attach/use debugger: ensure .NET CLI tools are on PATH and open a C# file so OmniSharp installs the debugger. The default debug output is `PowerShell/debug/powershell`.
- For low-level native debugging: `./tools/debug.sh` (LLDB + SOS) and `COREHOST_TRACE=1` to enable host tracing for the native host.

### Project-specific conventions & patterns
- The repository uses a PowerShell module `build.psm1` as its canonical build front-end — prefer invoking `Start-PSBuild` rather than calling `dotnet` directly unless you know the internal expectations.
- Cross-compilation and runtime targets are validated in `Start-PSBuild` (ValidateSet). When adding new runtimes or changing packaging, update `Start-PSBuild` accordingly.
- Many scripts use `Start-NativeExecution` wrappers to run native processes; search `Start-NativeExecution` for examples of hermetic command execution.
- Avoid committing changes to the committed `.vscode/` debug configs; they are intended to be a "just work" default for contributors.

### Integration points & external dependencies
- NuGet feeds: default uses Microsoft feeds; `-UseNuGetOrg` switches to public nuget.org. Secrets for internal feeds are read from `__DOTNET_RUNTIME_FEED` / `__DOTNET_RUNTIME_FEED_KEY` environment variables.
- dotnet SDK tooling: controlled by `global.json` and `DotnetRuntimeMetadata.json` — CI assumes specific SDK/Runtime channels.

### Files to read first (examples)
- `build.psm1` — canonical build and CI helpers (tags, bootstrap, Start-PSBuild).
- `PowerShell.sln` and `src/System.Management.Automation` — engine entry points and most core behavior.
- `src/powershell-native` and `src/powershell` — native host and platform glue.
- `test/powershell` and `test/xUnit` — tests and examples of how features are exercised.
- `docs/building/*` — platform-specific build steps (Linux, macOS, Windows).

### Short examples for agents (do this first)
- Bootstrap: `Import-Module ./build.psm1; Start-PSBootstrap`
- Quick build: `Import-Module ./build.psm1; Start-PSBuild -SMAOnly -Configuration Debug`
- Full build: `Import-Module ./build.psm1; Start-PSBuild -Configuration Debug -Runtime win7-x64`
- Run Pester tests (from repo root): `pwsh -NoProfile -Command 'Invoke-Pester -Path test/powershell -OutputFormat NUnitXml -OutputFile results.xml'`

### When making changes
- If touching native + managed boundaries, run the full build for the target platform and run the related tests. Use `Start-PSBuild -Clean` when outputs are stale.
- Respect the `git clean` exclusions in `Start-PSBuild -Clean` (sqlite3 and certain nuget.config files) — the build intentionally keeps them.

### If something is missing
- The repository has extensive human-oriented docs under `docs/` (building, debugging, maintainers). Use those as authoritative references for platform quirks.

---
If you want, I can iterate this file to add more code examples (e.g., common diagnostic one-liners, important function call sites) — tell me which areas you want expanded.
