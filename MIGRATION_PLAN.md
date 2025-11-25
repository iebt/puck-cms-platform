# .NET 6 to .NET 9 Migration Plan

## Project Overview

**Current State:**
- Solution: `puckcore.sln`
- Target Framework: `net6.0`
- Projects:
  1. **core** (`core.csproj`) - Core CMS library
  2. **puckweb** (`puckweb.csproj`) - Web application (ASP.NET Core MVC)
  3. **AzureDirectory** (`AzureDirectory.csproj`) - Azure Lucene storage support
  4. **tests** (`tests.csproj`) - NUnit test project

**Migration Target:** .NET 9.0

---

## Step 1: Baseline Validation

**Objective:** Establish current working state before migration begins.

### 1.1 Validate Current Build

```bash
dotnet clean puckcore.sln
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Expected Outcome:** Clean build with zero errors on .NET 6.0

**Success Criteria:**
- ✅ All 4 projects build successfully
- ✅ No build warnings related to deprecated APIs
- ✅ No compilation errors

**Document Results:**
```bash
# Save build output for comparison
dotnet build puckcore.sln --configuration Release > baseline_build.log 2>&1
```

### 1.2 Run Unit Tests

```bash
dotnet test tests/tests.csproj --configuration Release --logger "console;verbosity=detailed"
```

**Expected Outcome:** All tests pass

**Success Criteria:**
- ✅ All existing unit tests pass
- ✅ No test failures or errors

**Document Results:**
```bash
# Save test results
dotnet test tests/tests.csproj --configuration Release --logger "console;verbosity=detailed" > baseline_tests.log 2>&1
```

### 1.3 Verify Runtime Behavior

```bash
cd puckweb
dotnet run --project puckweb.csproj
```

**Manual Verification Steps:**
1. Application starts without errors
2. Access backoffice at `http://localhost:5000/puck`
3. Verify login functionality works
4. Check that Lucene indexing initializes
5. Verify database connectivity (SQLite/SQL Server/PostgreSQL/MySQL)

**Expected Outcome:** Application runs successfully on .NET 6.0

**Document Results:**
- Screenshot or log of successful startup
- Note any warnings in console output

### 1.4 Baseline Summary Document

Create a `BASELINE_REPORT.md` documenting:
- Build status
- Test results
- Runtime behavior
- Known issues (if any)
- Environment details (OS, .NET SDK version)

**Commit Point:**
```
git add baseline_build.log baseline_tests.log BASELINE_REPORT.md
git commit -m "chore(migration): establish .NET 6.0 baseline before migration to .NET 9

- Document current build output
- Record test results
- Capture runtime behavior
- Establish success criteria for migration validation"
```

---

## Step 2: Upgrade .NET Framework Version

**Objective:** Update all project files from `net6.0` to `net9.0`

### 2.1 Update TargetFramework in All Projects

**Files to modify:**

1. `core/core.csproj`
   - Change: `<TargetFramework>net6.0</TargetFramework>`
   - To: `<TargetFramework>net9.0</TargetFramework>`

2. `puckweb/puckweb.csproj`
   - Change: `<TargetFramework>net6.0</TargetFramework>`
   - To: `<TargetFramework>net9.0</TargetFramework>`

3. `AzureDirectory/AzureDirectory.csproj`
   - Change: `<TargetFramework>net6.0</TargetFramework>`
   - To: `<TargetFramework>net9.0</TargetFramework>`

4. `tests/tests.csproj`
   - Change: `<TargetFramework>net6.0</TargetFramework>`
   - To: `<TargetFramework>net9.0</TargetFramework>`

### 2.2 Attempt Initial Build

```bash
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Expected Issues:**
- Dependency version conflicts
- Breaking API changes
- Obsolete API warnings

**Document Results:**
```bash
# Capture initial .NET 9 build output
dotnet build puckcore.sln --configuration Release > net9_initial_build.log 2>&1
```

### 2.3 Analyze Build Output

Review `net9_initial_build.log` for:
1. **Framework-level breaking changes** (compiler errors)
2. **Dependency mismatch errors** (package compatibility issues)
3. **Obsolete API warnings**

**Decision Tree:**
- ✅ **If build succeeds:** Proceed to Step 3 (Validation)
- ❌ **If framework-level errors exist:** Address breaking changes (see Section 2.4)
- ❌ **If dependency errors exist:** Proceed to Step 4 (Update NuGet Packages)

### 2.4 Address Framework-Level Breaking Changes (If Applicable)

**Known .NET 6 → .NET 9 Breaking Changes:**

1. **System.Text.Json changes:**
   - Default serialization options may differ
   - Check `Startup.cs:58` for `AddJsonOptions` configuration

2. **Entity Framework Core changes:**
   - EF Core 9 may have behavior changes in query translation
   - Review any raw SQL or complex LINQ queries

3. **ASP.NET Core Identity changes:**
   - Authentication/authorization middleware order matters
   - Verify `Startup.cs:204-206` authentication setup

4. **Nullable reference types enforcement:**
   - .NET 9 may be stricter with nullable annotations

**Action Items:**
- Fix compiler errors one by one
- Consult Microsoft's breaking changes documentation
- Test each fix in isolation

### 2.5 Framework Update Validation

```bash
# Try building again after framework fixes
dotnet clean puckcore.sln
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Commit Point (if build succeeds):**
```
git add core/core.csproj puckweb/puckweb.csproj AzureDirectory/AzureDirectory.csproj tests/tests.csproj
git commit -m "chore(migration): upgrade target framework from net6.0 to net9.0

- Update all project files to target .NET 9.0
- Address framework-level breaking changes (if any)
- Build compiles successfully on .NET 9.0 framework"
```

**Commit Point (if framework fixes were needed):**
```
git add .
git commit -m "fix(migration): resolve .NET 9 framework-level breaking changes

- Fix compiler errors from .NET 6 to .NET 9 upgrade
- Update code to comply with .NET 9 API changes
- Document breaking changes in commit message"
```

---

## Step 3: Post-Framework Validation

**Objective:** Verify framework upgrade without dependency updates

### 3.1 Build Validation

```bash
dotnet build puckcore.sln --configuration Release
```

**Success Criteria:**
- ✅ All projects build successfully
- ⚠️ Dependency warnings are acceptable at this stage

### 3.2 Test Execution

```bash
dotnet test tests/tests.csproj --configuration Release --logger "console;verbosity=detailed"
```

**Expected Outcome:**
- Tests may fail due to dependency mismatches
- Document failures for comparison after dependency updates

**Document Results:**
```bash
dotnet test tests/tests.csproj --configuration Release > net9_framework_tests.log 2>&1
```

### 3.3 Decision Point

**Analysis:**
- Compare `net9_framework_tests.log` with `baseline_tests.log`
- Identify test failures caused by framework vs. dependencies

**Next Steps:**
- ✅ **If tests pass:** Skip dependency updates (unlikely but possible)
- ❌ **If dependency errors exist:** Proceed to Step 4
- ❌ **If runtime errors exist:** Proceed to Step 4

---

## Step 4: Update NuGet Dependencies

**Objective:** Update packages to .NET 9 compatible versions

### Package Categories

#### 4.1 MUST Update (Critical for .NET 9)

**Microsoft Core Packages (core/core.csproj & puckweb/puckweb.csproj):**

| Package | Current Version | Target Version | Notes |
|---------|----------------|----------------|-------|
| `Microsoft.EntityFrameworkCore` | 6.0.2 | 9.0.0 | Core EF package |
| `Microsoft.EntityFrameworkCore.Sqlite` | 6.0.2 | 9.0.0 | SQLite provider |
| `Microsoft.EntityFrameworkCore.SqlServer` | 6.0.2 | 9.0.0 | SQL Server provider |
| `Microsoft.EntityFrameworkCore.Tools` | 6.0.2 | 9.0.0 | EF migrations |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | 6.0.3 | 9.0.1 | PostgreSQL provider |
| `Pomelo.EntityFrameworkCore.MySql` | 6.0.1 | 9.0.0 | MySQL provider |
| `Microsoft.AspNetCore.Identity.EntityFrameworkCore` | 6.0.2 | 9.0.0 | Identity with EF |
| `Microsoft.AspNetCore.Identity.UI` | 6.0.2 | 9.0.0 | Identity UI |
| `Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore` | 6.0.2 | 9.0.0 | EF diagnostics |
| `Microsoft.Extensions.DependencyModel` | 6.0.0 | 9.0.0 | DI model |
| `Microsoft.Extensions.Hosting` | 6.0.1 | 9.0.0 | Hosting abstractions |
| `Microsoft.Extensions.Configuration` | 6.0.0 | 9.0.0 | Configuration (tests) |
| `Microsoft.Extensions.Configuration.Json` | 6.0.0 | 9.0.0 | JSON config |
| `Microsoft.Extensions.Logging.Console` | 6.0.0 | 9.0.0 | Console logging |

**Incompatible Legacy ASP.NET Core Packages (core/core.csproj):**
- ⚠️ These are .NET Core 2.2 packages and must be removed or replaced:

| Package | Current Version | Action | Notes |
|---------|----------------|--------|-------|
| `Microsoft.AspNetCore.Identity` | 2.2.0 | **REMOVE** | Functionality is in .NET 9 framework |
| `Microsoft.AspNetCore.Localization` | 2.2.0 | **REMOVE** | Functionality is in .NET 9 framework |
| `Microsoft.AspNetCore.Mvc.Core` | 2.2.5 | **REMOVE** | Functionality is in .NET 9 framework |
| `Microsoft.AspNetCore.Mvc.DataAnnotations` | 2.2.0 | **REMOVE** | Functionality is in .NET 9 framework |
| `Microsoft.AspNetCore.ResponseCaching` | 2.2.0 | **REMOVE** | Functionality is in .NET 9 framework |
| `Microsoft.AspNetCore.Mvc.ViewFeatures` | 2.2.0 | **REMOVE** (puckweb) | Functionality is in .NET 9 framework |

**Test Packages (tests/tests.csproj):**

| Package | Current Version | Target Version | Notes |
|---------|----------------|----------------|-------|
| `Microsoft.EntityFrameworkCore.InMemory` | 6.0.2 | 9.0.0 | In-memory testing |
| `Microsoft.NET.Test.Sdk` | 17.1.0 | 17.12.0 | Test SDK |

#### 4.2 RECOMMENDED Update (Best Practice)

| Package | Current Version | Recommended Version | Notes |
|---------|----------------|---------------------|-------|
| `SixLabors.ImageSharp` | 1.0.4 | 3.1.6 | Major version upgrade - **review breaking changes** |
| `SixLabors.ImageSharp.Drawing` | 1.0.0-beta0008 | 2.1.5 | Stable release available |
| `SixLabors.ImageSharp.Web` | 1.0.4 | 3.1.4 | Compatible with ImageSharp 3.x |
| `SixLabors.ImageSharp.Web.Providers.Azure` | 1.0.4 | 3.1.4 | Azure provider update |
| `Azure.Storage.Blobs` | 12.10.0 | 12.23.0 | Bug fixes & performance |
| `MailKit` | 3.1.1 | 4.9.0 | Security & stability fixes |
| `MimeKit` | 3.1.1 | 4.9.0 | Must match MailKit version |
| `Newtonsoft.Json` | 13.0.1 | 13.0.3 | Patch updates |
| `Moq` | 4.16.1 | 4.20.72 | Latest stable |
| `nunit` | 3.13.2 | 4.2.2 | Major update - **review changes** |
| `NUnit3TestAdapter` | 4.2.1 | 4.6.0 | Adapter for NUnit 4 |
| `MiniProfiler.AspNetCore.Mvc` | 4.2.22 | 4.3.8 | Performance profiling |
| `Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation` | 6.0.2 | 9.0.0 | Runtime Razor compilation |
| `Microsoft.VisualStudio.Web.CodeGeneration.Design` | 6.0.2 | 9.0.0 | Code generation tools |

#### 4.3 PROBABLY CAN KEEP (But Monitor)

| Package | Current Version | Keep? | Notes |
|---------|----------------|-------|-------|
| `LinqKit.Microsoft.EntityFrameworkCore` | 6.1.1 | ✅ Yes | Should work with EF 9 |
| `Lucene.Net` (all packages) | 4.8.0-beta00008 | ✅ Yes | Beta but stable - no .NET 9 specific version |
| `Lucene.Net.Analysis.Common` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.Highlighter` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.Memory` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.Queries` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.QueryParser` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.Sandbox` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Lucene.Net.Spatial` | 4.8.0-beta00008 | ✅ Yes | Part of Lucene suite |
| `Microsoft.Azure.Storage.Blob` | 11.2.3 | ⚠️ Monitor | Legacy SDK - `Azure.Storage.Blobs` is preferred |
| `System.Collections.Concurrent` | 4.3.0 | ✅ Yes | Part of framework |
| `System.Drawing.Common` | 6.0.0 | ⚠️ Monitor | Not cross-platform - consider alternatives |
| `System.Text.Encoding` | 4.3.0 | ✅ Yes | Part of framework |
| `System.Text.Encoding.CodePages` | 6.0.0 | ⚠️ Update to 9.0.0 | For consistency |
| `System.Text.Encoding.Extensions` | 4.3.0 | ✅ Yes | Part of framework |

### 4.4 Update Strategy

**Phase 1: MUST Update (Critical Microsoft Packages)**

Update all Microsoft.* packages to version 9.0.0:

```bash
# Core project
cd core
dotnet add package Microsoft.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 9.0.1
dotnet add package Pomelo.EntityFrameworkCore.MySql --version 9.0.0
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.Extensions.DependencyModel --version 9.0.0
dotnet add package Microsoft.Extensions.Hosting --version 9.0.0

# Remove legacy ASP.NET Core 2.2 packages
dotnet remove package Microsoft.AspNetCore.Identity
dotnet remove package Microsoft.AspNetCore.Localization
dotnet remove package Microsoft.AspNetCore.Mvc.Core
dotnet remove package Microsoft.AspNetCore.Mvc.DataAnnotations
dotnet remove package Microsoft.AspNetCore.ResponseCaching

cd ..

# Web project
cd puckweb
dotnet add package Microsoft.AspNetCore.Diagnostics.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore --version 9.0.0
dotnet add package Microsoft.AspNetCore.Identity.UI --version 9.0.0
dotnet add package Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.SqlServer --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Tools --version 9.0.0
dotnet add package Microsoft.Extensions.Logging.Console --version 9.0.0
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design --version 9.0.0
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL --version 9.0.1
dotnet add package Pomelo.EntityFrameworkCore.MySql --version 9.0.0

# Remove legacy package
dotnet remove package Microsoft.AspNetCore.Mvc.ViewFeatures

cd ..

# Test project
cd tests
dotnet add package Microsoft.EntityFrameworkCore.InMemory --version 9.0.0
dotnet add package Microsoft.EntityFrameworkCore.Sqlite --version 9.0.0
dotnet add package Microsoft.Extensions.Configuration --version 9.0.0
dotnet add package Microsoft.Extensions.Configuration.Json --version 9.0.0
dotnet add package Microsoft.Extensions.Hosting --version 9.0.0
dotnet add package Microsoft.NET.Test.Sdk --version 17.12.0

cd ..
```

**Build and Test:**
```bash
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Document Results:**
```bash
dotnet build puckcore.sln --configuration Release > net9_must_update_build.log 2>&1
```

**Commit Point:**
```
git add core/core.csproj puckweb/puckweb.csproj tests/tests.csproj
git commit -m "chore(migration): update critical Microsoft packages to .NET 9

- Update Entity Framework Core to 9.0.0
- Update ASP.NET Core Identity to 9.0.0
- Update all Microsoft.Extensions.* to 9.0.0
- Remove legacy ASP.NET Core 2.2 packages (now in framework)
- Update database providers (Npgsql, Pomelo) to 9.0.x compatible versions"
```

**Phase 2: RECOMMENDED Update (Test Independently)**

**⚠️ IMPORTANT: ImageSharp 3.x is a major breaking change. Test carefully.**

```bash
# Azure Directory
cd AzureDirectory
dotnet add package Azure.Storage.Blobs --version 12.23.0

cd ..

# Core project
cd core
dotnet add package MailKit --version 4.9.0
dotnet add package MimeKit --version 4.9.0
dotnet add package MiniProfiler.AspNetCore.Mvc --version 4.3.8
dotnet add package Newtonsoft.Json --version 13.0.3
dotnet add package System.Text.Encoding.CodePages --version 9.0.0

cd ..

# Web project - ImageSharp updates
cd puckweb
dotnet add package SixLabors.ImageSharp --version 3.1.6
dotnet add package SixLabors.ImageSharp.Drawing --version 2.1.5
dotnet add package SixLabors.ImageSharp.Web --version 3.1.4
dotnet add package SixLabors.ImageSharp.Web.Providers.Azure --version 3.1.4
dotnet add package MiniProfiler.AspNetCore.Mvc --version 4.3.8

cd ..

# Core project - ImageSharp (core also uses it)
cd core
dotnet add package SixLabors.ImageSharp --version 3.1.6

cd ..

# Test project
cd tests
dotnet add package Moq --version 4.20.72
dotnet add package nunit --version 4.2.2
dotnet add package NUnit3TestAdapter --version 4.6.0

cd ..
```

**⚠️ Breaking Change Alert - ImageSharp 3.x:**

Changes required in code:
1. Namespace changes: `SixLabors.ImageSharp.Processing` namespace structure changed
2. API changes in image processing methods
3. Configuration changes in `Startup.cs` for ImageSharp.Web

**Review files that use ImageSharp:**
- `core/ImageSharp/WebProcessors/` - Custom processors
- `puckweb/Startup.cs:106-151` - ImageSharp configuration
- Any ViewModels or code using `IFormFile` with image transformers

**Build and Test:**
```bash
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Document Results:**
```bash
dotnet build puckcore.sln --configuration Release > net9_recommended_update_build.log 2>&1
```

**Commit Point:**
```
git add .
git commit -m "chore(migration): update recommended packages for .NET 9

- Update ImageSharp to 3.1.6 (breaking changes - review usage)
- Update MailKit/MimeKit to 4.9.0
- Update Azure.Storage.Blobs to 12.23.0
- Update MiniProfiler to 4.3.8
- Update test packages (Moq, NUnit)
- Update miscellaneous packages for stability"
```

**Phase 3: Monitor Packages (Keep Current Versions)**

No changes needed for:
- All Lucene.Net packages (beta but stable)
- LinqKit.Microsoft.EntityFrameworkCore
- System.* packages included in framework

**Optional Updates:**
```bash
# Only if needed
cd tests
dotnet add package System.Text.Encoding.CodePages --version 9.0.0
```

---

## Step 5: Post-Dependency Update Validation

**Objective:** Verify all updates work correctly

### 5.1 Clean Build

```bash
dotnet clean puckcore.sln
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
```

**Success Criteria:**
- ✅ Zero build errors
- ✅ Minimal or zero warnings
- ✅ All 4 projects compile

**Document Results:**
```bash
dotnet build puckcore.sln --configuration Release > net9_final_build.log 2>&1
```

### 5.2 Run All Tests

```bash
dotnet test tests/tests.csproj --configuration Release --logger "console;verbosity=detailed"
```

**Success Criteria:**
- ✅ All tests pass (same as baseline)
- ✅ No new test failures compared to baseline

**Document Results:**
```bash
dotnet test tests/tests.csproj --configuration Release --logger "console;verbosity=detailed" > net9_final_tests.log 2>&1
```

### 5.3 Compare with Baseline

**Analysis:**
- Compare `net9_final_build.log` with `baseline_build.log`
- Compare `net9_final_tests.log` with `baseline_tests.log`
- Verify build is clean
- Verify test results match or improve

**Action Items if Issues Found:**
- If tests fail: Debug and fix (may require code changes)
- If build warnings exist: Review and address if critical
- If runtime errors: Proceed to Step 6

---

## Step 6: Runtime Validation & Integration Testing

**Objective:** Verify application behavior in .NET 9 runtime

### 6.1 Start Application

```bash
cd puckweb
dotnet run --project puckweb.csproj
```

**Monitoring:**
- Watch console output for errors/warnings
- Check for startup exceptions
- Verify all services initialize

### 6.2 Functional Testing Checklist

**Database Connectivity:**
- ✅ SQLite connection works
- ✅ SQL Server connection works (if configured)
- ✅ PostgreSQL connection works (if configured)
- ✅ MySQL connection works (if configured)

**Lucene Indexing:**
- ✅ Lucene index initializes
- ✅ Index path is accessible
- ✅ No indexing errors in logs

**Authentication/Authorization:**
- ✅ Navigate to `/puck` (backoffice)
- ✅ Login with admin credentials
- ✅ Authentication middleware works correctly
- ✅ Authorization policies are enforced

**Content Management:**
- ✅ Create new content node
- ✅ Edit existing content
- ✅ Delete content
- ✅ Content is indexed in Lucene

**Image Processing (ImageSharp):**
- ✅ Upload image via backoffice
- ✅ Image transformers work
- ✅ Image resizing/cropping works (query parameters)
- ✅ Azure blob storage works (if configured)

**Frontend Rendering:**
- ✅ Set domain mapping in backoffice
- ✅ Access frontend at `/`
- ✅ Content renders correctly
- ✅ Views and templates work

**MiniProfiler:**
- ✅ Access `/profiler`
- ✅ Profiling data displays correctly

### 6.3 Performance Comparison

**Baseline Metrics (NET 6):**
- Application startup time: _________
- First request time: _________
- Average response time: _________

**NET 9 Metrics:**
- Application startup time: _________
- First request time: _________
- Average response time: _________

**Expected:** Similar or improved performance on .NET 9

### 6.4 Error Testing

**Test Error Handling:**
- ✅ Access non-existent route (verify 404 handling)
- ✅ Trigger application error (verify 500 handling)
- ✅ Check error logs

### 6.5 Multi-Database Testing (If Applicable)

Test each database provider configured:

**SQLite:**
```bash
# Configure UseSQLite: true in appSettings.json
dotnet run --project puckweb.csproj
# Run functional tests
```

**SQL Server:**
```bash
# Configure UseSQLServer: true in appSettings.json
dotnet run --project puckweb.csproj
# Run functional tests
```

**PostgreSQL:**
```bash
# Configure UsePostgreSQL: true in appSettings.json
dotnet run --project puckweb.csproj
# Run functional tests
```

**MySQL:**
```bash
# Configure UseMySQL: true in appSettings.json
dotnet run --project puckweb.csproj
# Run functional tests
```

### 6.6 Migration Validation Document

Create `NET9_VALIDATION_REPORT.md`:
- Functional test results
- Performance metrics
- Known issues (if any)
- Database compatibility matrix
- ImageSharp compatibility notes

**Commit Point:**
```
git add NET9_VALIDATION_REPORT.md
git commit -m "docs(migration): .NET 9 runtime validation complete

- Document functional testing results
- Record performance metrics
- Verify database provider compatibility
- Confirm ImageSharp 3.x integration
- Validate authentication and content management flows"
```

---

## Step 7: Address Issues (If Any)

**Objective:** Fix any runtime issues discovered during validation

### 7.1 Common Issues & Solutions

**Issue: ImageSharp 3.x Breaking Changes**

**Symptoms:**
- Compilation errors in custom processors
- Runtime errors during image processing
- Configuration errors in Startup.cs

**Solution:**
1. Review ImageSharp 3.x migration guide
2. Update custom processors in `core/ImageSharp/WebProcessors/`
3. Update ImageSharp.Web configuration in `Startup.cs`
4. Review breaking changes: https://docs.sixlabors.com/articles/imagesharp/releasenotes.html

**Issue: EF Core 9 Query Translation Changes**

**Symptoms:**
- Queries that worked in EF Core 6 fail
- Different results from LINQ queries
- InvalidOperationException during query execution

**Solution:**
1. Review EF Core 9 breaking changes
2. Update problematic queries
3. Consider using raw SQL for complex queries
4. Enable query logging for debugging

**Issue: ASP.NET Core Identity Changes**

**Symptoms:**
- Authentication fails
- Authorization policies not working
- Cookie authentication issues

**Solution:**
1. Review Identity API changes in .NET 9
2. Update authentication configuration in `Startup.cs`
3. Check cookie settings and policies
4. Verify password hashing compatibility

**Issue: Legacy Package Removal Issues**

**Symptoms:**
- Missing types after removing ASP.NET Core 2.2 packages
- Compilation errors for removed packages

**Solution:**
1. These types are now part of the .NET 9 framework
2. Update `using` statements to correct namespaces
3. Remove explicit package references (already in framework)

### 7.2 Fix and Validate Loop

For each issue:
1. Identify root cause
2. Implement fix
3. Build and test: `dotnet build && dotnet test`
4. Runtime test: `dotnet run`
5. Verify fix resolves issue
6. Commit fix

**Commit Pattern:**
```
git add .
git commit -m "fix(migration): resolve [specific issue] in .NET 9

- Describe the problem
- Explain the solution
- Reference any breaking changes or documentation"
```

---

## Step 8: Final Validation & Cleanup

**Objective:** Complete final checks before marking migration complete

### 8.1 Full Solution Build

```bash
dotnet clean puckcore.sln
dotnet restore puckcore.sln
dotnet build puckcore.sln --configuration Release
dotnet build puckcore.sln --configuration Debug
```

**Success Criteria:**
- ✅ Both Release and Debug configurations build cleanly
- ✅ Zero errors
- ✅ Zero critical warnings

### 8.2 Complete Test Suite

```bash
dotnet test tests/tests.csproj --configuration Release
dotnet test tests/tests.csproj --configuration Debug
```

**Success Criteria:**
- ✅ All tests pass in both configurations
- ✅ Test results match or exceed baseline

### 8.3 Code Quality Check

Review for:
- Deprecated API usage warnings
- Nullable reference type warnings
- Obsolete attribute warnings
- Compiler suggestions for modern C# features

**Action:**
- Address warnings where feasible
- Document intentionally ignored warnings

### 8.4 Documentation Updates

Update project documentation:

**README.md Updates:**
- Update .NET version requirement to 9.0
- Update SDK installation instructions
- Update any version-specific notes

**CLAUDE.md Updates:**
- Update target framework references
- Update package version examples
- Add .NET 9 specific notes

**appSettings.json:**
- Review configuration for .NET 9 compatibility
- Update comments if needed

### 8.5 Clean Up Migration Artifacts

```bash
# Remove log files (keep them in git history)
rm -f baseline_build.log baseline_tests.log
rm -f net9_initial_build.log net9_framework_tests.log
rm -f net9_must_update_build.log net9_recommended_update_build.log
rm -f net9_final_build.log net9_final_tests.log
```

### 8.6 Update Version Numbers (Optional)

If this is a release milestone:

**core/core.csproj:**
```xml
<Version>1.0.0-beta0002</Version>
```

**puckweb/puckweb.csproj:**
```xml
<Version>1.0.0-beta0002</Version>
```

**Commit Point:**
```
git add README.md CLAUDE.md core/core.csproj puckweb/puckweb.csproj
git commit -m "chore(migration): finalize .NET 9 migration

- Update documentation to reflect .NET 9 requirements
- Clean up migration artifacts
- Bump version to 1.0.0-beta0002
- Migration complete and validated"
```

---

## Step 9: Create Migration Summary

**Objective:** Document the completed migration for reference

### 9.1 Create MIGRATION_SUMMARY.md

Document:
- Migration start date and completion date
- .NET versions (6.0 → 9.0)
- Package updates performed
- Breaking changes encountered and resolved
- Test results (before/after comparison)
- Performance impact (if any)
- Known issues or limitations
- Rollback instructions (if needed)

**Commit Point:**
```
git add MIGRATION_SUMMARY.md
git commit -m "docs(migration): add comprehensive migration summary

- Document complete migration journey from .NET 6 to .NET 9
- List all package updates and version changes
- Record breaking changes and solutions
- Provide rollback instructions if needed
- Include validation results and performance metrics"
```

---

## Rollback Plan

**If migration must be rolled back:**

### Quick Rollback (Git)

```bash
# If changes are not pushed
git reset --hard <commit-before-migration>

# If changes are pushed
git revert <migration-commits> --no-commit
git commit -m "revert: rollback .NET 9 migration to .NET 6"
```

### Manual Rollback

1. Revert all `TargetFramework` changes back to `net6.0`
2. Revert all package versions to original versions
3. Restore removed packages:
   ```bash
   dotnet add package Microsoft.AspNetCore.Identity --version 2.2.0
   dotnet add package Microsoft.AspNetCore.Localization --version 2.2.0
   # etc.
   ```
4. Build and test: `dotnet build && dotnet test`

---

## Risk Assessment

**Low Risk:**
- ✅ Microsoft package updates (well-documented, stable)
- ✅ Database provider updates (tested and stable)
- ✅ Framework upgrade (Microsoft provides migration guides)

**Medium Risk:**
- ⚠️ ImageSharp 3.x upgrade (major version, breaking changes)
- ⚠️ NUnit 4.x upgrade (major version)
- ⚠️ Custom code that relies on obsolete APIs

**High Risk:**
- ❌ None identified (all changes are standard framework upgrades)

**Mitigation:**
- Follow step-by-step approach
- Validate after each major change
- Maintain baseline for comparison
- Keep git history for rollback
- Test on non-production environment first

---

## Success Criteria

**Migration is complete when:**
- ✅ All projects target .NET 9.0
- ✅ All critical packages updated to .NET 9 compatible versions
- ✅ Solution builds without errors
- ✅ All tests pass
- ✅ Application runs successfully
- ✅ Functional testing passes
- ✅ Performance is equal or better than baseline
- ✅ All database providers work
- ✅ Documentation is updated

---

## Timeline Estimate

**Estimated Duration:** 6-10 hours

- Step 1 (Baseline): 1-2 hours
- Step 2 (Framework Upgrade): 30 minutes - 2 hours (depends on breaking changes)
- Step 3 (Validation): 30 minutes
- Step 4 (Dependencies): 2-3 hours
- Step 5 (Validation): 1 hour
- Step 6 (Runtime Testing): 2-3 hours
- Step 7 (Issues): 0-2 hours (depends on findings)
- Step 8 (Cleanup): 30 minutes
- Step 9 (Documentation): 30 minutes

**Note:** Timeline assumes no major blocking issues. ImageSharp 3.x migration may add 2-4 hours if significant code changes are needed.

---

## References

- [.NET 9 Release Notes](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-9)
- [ASP.NET Core 9.0 Breaking Changes](https://learn.microsoft.com/en-us/aspnet/core/release-notes/aspnetcore-9.0)
- [EF Core 9.0 Breaking Changes](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/breaking-changes)
- [ImageSharp 3.x Migration Guide](https://docs.sixlabors.com/articles/imagesharp/releasenotes.html)
- [NUnit 4.0 Release Notes](https://docs.nunit.org/articles/nunit/release-notes/framework.html)

---

## Notes

- **No Docker configuration found:** Build validation will be command-line only
- **Legacy packages removed:** ASP.NET Core 2.2 packages are obsolete and removed
- **ImageSharp upgrade:** Major version upgrade requires code review
- **Lucene.Net:** No updates available (beta version is stable)
- **Multi-database support:** Test all 4 database providers (SQLite, SQL Server, PostgreSQL, MySQL)
- **Cross-platform:** PreBuild/PostBuild events already cross-platform compatible

---

**END OF MIGRATION PLAN**
