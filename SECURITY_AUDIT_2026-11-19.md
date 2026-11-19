# Filee Security Audit — 2026-11-19

## Scope

- **Audited version:** Filee v2.5.0, commit `aee96c3d` (Merge pull request #1564 from KnifeLemon/feat/whats-new-2.5.0)
- **Upstream repository:** https://github.com/KnifeLemon/Filee
- **Audit branch:** `security/audit-and-regression-fixes-2026-11-19` on fork `keerthan-ms/Filee`
- **Installed copy:** `C:\Program Files\Filee`, version 2.5.0, same commit `aee96c3d`
- **.NET SDK:** Not available in this environment — build and test execution blocked

## Previously disclosed vulnerabilities (GHSA advisories)

All three advisories state **Affected versions: < 2.5.0** and **Patched versions: 2.5.0**.
The v2.5.0 tag (`d3c5c641`) is an ancestor of the installed commit (`aee96c3d`), and no Setup or archive-security files changed between v2.5.0 and `aee96c3d`. All patches are present.

| Advisory | Severity | CVSS | CWE | Title | Status |
|----------|----------|------|-----|-------|--------|
| GHSA-x2fp-2ch7-5fvx | High | 7.9 | CWE-59 | Windows installer follows directory junctions when deleting %LOCALAPPDATA%\Filee with admin rights | **PATCHED** in v2.5.0 |
| GHSA-vq2m-qq38-mxx3 | Moderate | 6.7 | CWE-367 | Windows uninstaller executes unverified copy from user-writable %TEMP% (TOCTOU) | **PATCHED** in v2.5.0 |
| GHSA-vcp3-99h4-v3wx | Moderate | — | CWE-400 / CWE-770 | Small crafted documents and archives can exhaust memory, disk or CPU during conversion | **PATCHED** in v2.5.0 |

### Patch locations (verified in source)

**GHSA-x2fp-2ch7-5fvx** — `src/Filee.Setup/Installation/LinkSafeDelete.cs`
- `RealFolder()` validates a folder isn't reached through a junction/link before anything inside is deleted
- `Entry()` opens with `OpenReparsePoint` (deletes the link itself, never follows it) and validates the real path is inside the root
- `DeleteTree()` handles junctions and links without following them
- All Setup deletion paths (`VelopackInstall.Remove`, `RemoveLeftoverWorkFolders`, uninstaller settings cleanup) route through `LinkSafeDelete`

**GHSA-vq2m-qq38-mxx3** — `src/Filee.Setup/Program.cs`, `CopyUninstaller()`
- When elevated, the uninstaller copy goes to `%SystemRoot%\Temp` (SYSTEM/administrators only), not user-writable `%TEMP%`
- When unelevated, the copy goes to user `%TEMP%` but runs unelevated
- No Authenticode/hash re-verification of the copy is performed (the copy is byte-identical to the signed original, and the elevated copy is placed in a protected directory)

**GHSA-vcp3-99h4-v3wx** — Multiple files:
- `src/Filee.Engines/Archives/ArchiveSafety.cs` — MaxEntries (1,000,000), MaxPathDepth (512), zip-slip protection, disk space reserve, stream limits, symlink/junction removal without following, reserved device name sanitization
- `src/Filee.Engines/Infrastructure/BoundedXml.cs` — MaxPartBytes (256 MB), MaxPackageBytes (1 GB), MaxElementDepth (1000), DTD prohibited, XmlResolver = null, CountingStream with size checks during decompression
- `src/Filee.Engines/Infrastructure/EngineInstaller.cs` — SHA-256 verification of every downloaded engine component against `engines.json`, download discarded on mismatch

## Open GitHub bugs (functional, not security)

All three issues are **open** and awaiting reproduction files from reporters. No fix can be implemented without sample inputs.

| Issue | Title | Status | Notes |
|-------|-------|--------|-------|
| #1647 | Word-to-PDF text mismatch | Open | Reporter hasn't attached sample .docx/.pdf files. Maintainer asked for clarification on what "mismatched" means. |
| #1685 | File conversion produces very small file, loss of detail | Open | No files attached. No engine status or logs provided. |
| #1687 | Word table layout changes during PDF conversion (1 page → 2 pages) | Open | No files attached. Reporter on Windows 26H2, Filee 2.5.0. |

## Security-sensitive code reviewed (no issues found)

| File | What it does | Security assessment |
|------|-------------|---------------------|
| `src/Filee.App/Services/UpdateService.cs` | Update download and verification | SHA-256 against release SHA256SUMS, ECDSA P-256 signature of SHA256SUMS, Authenticode publisher check, mirror with same verification, path traversal and symlink protection in tar extraction |
| `src/Filee.Engines/Infrastructure/EngineDownloads.cs` | Engine download metadata | SHA-256 and size pinned in `engines.json`, content-addressable mirror URLs |
| `src/Filee.Engines/Infrastructure/EngineInstaller.cs` | Engine download + install | SHA-256 verification on download, size check, marker file records installed hash |
| `src/Filee.Engines/Infrastructure/BoundedXml.cs` | XML reading with limits | Size and depth limits, DTD prohibited, no external entities |
| `src/Filee.Core/Platform/MarkOfTheWeb.cs` | Mark-of-the-web propagation | Reparse points skipped during enumeration |
| `src/Filee.Core/Settings/PortableImport.cs` | Portable settings import | Explicitly rejects symbolic links and junctions |
| `src/Filee.Core/Gallery/SharedPresetFolderCheck.cs` | Preset folder validation | Reparse points rejected |
| `src/Filee.Core/Conversion/JobQueue.cs` | Conversion job queue | Reparse points skipped during enumeration |
| `src/Filee.Core/Presets/FolderCopy.cs` | Preset folder copy | Reparse points rejected |

## Build and test status

| Check | Status | Reason |
|-------|--------|--------|
| `dotnet build` | BLOCKED | .NET 10 SDK not installed (global.json pins 10.0.100) |
| `dotnet test` | BLOCKED | .NET 10 SDK not installed |
| Installer check (`build-installer.ps1 -CheckOnly`) | BLOCKED | .NET 10 SDK not installed, Windows-only |
| Dependency vulnerability scan | NOT PERFORMED | Requires `dotnet list package --vulnerable` or `pip-audit` equivalent |

## Recommendations

1. **No code changes required** — All three disclosed GHSA advisories are patched in v2.5.0, which is an ancestor of the installed commit.
2. **Install .NET 10 SDK** and run `dotnet test` to confirm no regressions were introduced after v2.5.0.
3. **For issues #1647, #1685, #1687:** Request reproduction files from reporters before any fix can be developed.
4. **Monitor for new GHSA advisories** — the security policy (SECURITY.md) asks reporters to use private advisories, not public issues.
5. **Consider adding regression tests** for LinkSafeDelete (junction resistance), ArchiveSafety (entry bombs, zip slip), and EngineInstaller (checksum mismatch) once the .NET SDK is available.

## Git state

- Branch: `security/audit-and-regression-fixes-2026-11-19` (pushed to fork `keerthan-ms/Filee`)
- No code changes committed — all security patches already present in v2.5.0
- This document is the only addition to the branch
- No push to upstream `KnifeLemon/Filee:main`; no merge performed