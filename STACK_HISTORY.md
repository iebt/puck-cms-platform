# .NET Version History

This document tracks the .NET version evolution of the Puck CMS Platform project throughout its git history.

## Version Timeline

### 1. .NET Core 3.0 - Initial version
- **From:** September 8, 2019 (initial commit: 16ef0d9)
- **To:** December 14, 2019
- **Details:**
  - `puckweb` project: `netcoreapp3.0`
  - `core` project: `netstandard2.1`
  - Initial release of Puck CMS as a .NET Core platform

### 2. .NET Core 3.1 - First major upgrade
- **Upgraded on:** December 14, 2019 (commit: 3294dc3)
- **Message:** "updated to .net core 3.1 updated packages"
- **Follow-up:** January 7, 2020 (commit: 80e728e) - "updated tests.csproj to .net core 3.1 and updated package references"
- **Details:**
  - `puckweb` project: `netcoreapp3.1`
  - `core` project: remained `netstandard2.1`
  - `tests` project: upgraded to `netcoreapp3.1`

### 3. .NET 6.0 - Current version
- **Initial attempt:** February 22, 2022 (commit: 88d2f6e)
- **Message:** "update to .net6"
- **Successful upgrade:** February 23, 2022 (commit: 1309974)
- **Message:** "second attempt at update, seems to work well now and all tests pass"
- **Merged:** February 24, 2022 (commit: d6ab3b6)
- **Details:**
  - `puckweb` project: `net6.0`
  - `core` project: `net6.0` (changed from `netstandard2.1`)
  - `tests` project: `net6.0`
  - `AzureDirectory` project: `net6.0`
- **Notes:**
  - The upgrade required two attempts, with the second being successful
  - Initial issues with MiniProfiler compatibility (commit: 77c17e6) - switched to in-process hosting
  - All tests passing after the successful upgrade

## Summary

The project has undergone 2 major .NET version upgrades:

1. **.NET Core 3.0 → 3.1** (December 2019) - Minor upgrade completed quickly
2. **.NET Core 3.1 → .NET 6.0** (February 2022) - Required two attempts but successful

**Current Status:** All projects are on **.NET 6.0** as of February 2022 and remain on this version as of the latest commit.

## Notes for Future Upgrades

- The .NET 6 upgrade highlighted compatibility issues with MiniProfiler
- When upgrading, consider testing with all database providers (SQL Server, PostgreSQL, MySQL, SQLite)
- The migration from `netstandard2.1` to `net6.0` for the core library was significant and may require similar consideration for future upgrades
- All unit tests should pass before considering an upgrade complete
