# KeeAnywhere

KeePass 2.x plugin (C#, .NET Framework 4.8, WinForms) that adds cloud storage providers
(OneDrive, Dropbox, Google Drive, Box, Amazon S3, Azure, Google Cloud Storage, HiDrive) to KeePass'
I/O layer, plus offline cache and backup support.

## Structure

- `KeeAnywhere.sln` / `KeeAnywhere/KeeAnywhere.csproj` – the only project (old-style csproj,
  NuGet via `packages.config`, packages in `packages/`).
- `KeeAnywhere/KeeAnywhereExt.cs` – plugin entry point (`KeeAnywhereExt : Plugin`).
- `KeeAnywhere/StorageProviders/<Provider>/` – one folder per cloud provider;
  `StorageRegistry.cs` registers them, `IStorageProvider*.cs` are the interfaces.
- `KeeAnywhere/Configuration`, `Forms`, `OAuth2`, `Offline`, `Backup`, `WebRequest` – settings,
  UI, auth flows, offline cache, backups, KeePass `IOConnection` web request integration.
- `lib/KeePass.exe` – KeePass reference assembly (currently 2.57; the code also compiles against 2.62).
- `chocolatey/`, `docs/` (GitHub pages), `doc/`.

## Build

- `build.cmd` (needs VS 2022 Community, nuget, 7-Zip package, choco) → `build/dist`.
- Quick local build without packaging:
  `nuget.exe restore KeeAnywhere.sln` then
  `MSBuild.exe KeeAnywhere.sln /p:Configuration=Release /t:Build`
  → `KeeAnywhere/bin/Release/KeeAnywhere.plgx` (built by the `PlgxTool` NuGet package in the
  `AfterBuild` target; config in the `PlgxConfiguration` property group of the csproj).

## PLGX conventions

- KeePass >= 2.62 validates paths of files extracted from a PLGX: every packed file must have a
  path inside the plugin folder. Items referencing files outside the project (e.g.
  `<Analyzer Include="..\packages\...">`) must carry `<ExcludeFromPlgx />`, otherwise KeePass
  refuses to load the plugin ("Attempted to perform an unauthorized operation").
- NuGet references are packed under `References\` by PlgxTool automatically.
- KeePass compiles the PLGX itself with the .NET Framework CodeDOM C# compiler (C# 5, no MSBuild
  version unification). A reference version conflict that MSBuild silently resolves is a hard
  error there (e.g. CS1705). That's why `System.Net.Http` is a plain framework reference and
  not the NuGet `System.Net.Http` 4.3.4 DLL (Microsoft.Graph.Core needs 4.2.0.0).
- KeePass ignores `app.config` binding redirects for plugins. `KeeAnywhereExt.FixDependencyLoading()`
  (run from the static constructor, i.e. before KeePass' compatibility check) installs an
  `AssemblyResolve` handler: a fixed preload list plus a generic fallback that loads any
  requested assembly by simple name from the plugin (cache) folder, ignoring the version.
  Code compiled into the PLGX must stay C# 5 compatible (no `out var`, `?.`, `$""`, etc.).
- A failed PLGX compile only shows "Operation is not valid due to the current state of the
  object" (`InvalidOperationException` in `PlgxPlugin.Compile`). Start KeePass with `-debug` to
  get the compiler output saved.
- Config files (`app.config`, `KeePass.exe.config`) are excluded from the PLGX.

## OAuth client credentials

The `*ClientId` / `*ClientSecret` constants in `StorageProviders/<Provider>/<Provider>Helper.cs`
(Google Drive, OneDrive, Dropbox, Box, HiDrive) are `"dummy"` placeholders in git. A build with
them loads fine but every login fails (e.g. Google: `invalid_client`). For a working local build,
insert the real values temporarily before building and restore the placeholders afterwards
(`git checkout -- KeeAnywhere/StorageProviders/*/*Helper.cs`). Never commit real credentials.

## Versioning

See `BUILD.md`: version lives in `KeeAnywhere/Properties/AssemblyInfo.cs`, `build.cmd`,
`CHANGELOG.md`, `chocolatey/keepass-plugin-keeanywhere.nuspec`, `version_manifest.txt`.
