# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Puck CMS Platform is a pure .NET Core MVC CMS platform. It's code-first, unobtrusive, and extensible with complete querying and Lucene integration. The CMS is inspired by Umbraco and can be used as an integrated CMS, headless/decoupled, or as a searchable datastore.

**Key Characteristics:**
- Pages are based on ViewModels (POCOs with attributes)
- Edit screens use standard ASP.NET MVC Editor Templates
- Queries use Lucene instead of database for performance
- Strongly typed design throughout
- Multi-site and multilingual support
- Supports SQL Server, SQLite, MySQL, PostgreSQL

## Solution Structure

The solution (`puckcore.sln`) contains 4 projects:

1. **core** - Core CMS library (`puck.core`)
   - Contains Abstract interfaces, Concrete implementations
   - Lucene indexing (`PuckLucene/`)
   - Database migrations for all DB providers (`Migrations/`)
   - Identity management (`Identity/`)
   - Background tasks (`Tasks/`)
   - Analyzers, Filters, Globalisation support
   - Target: `net6.0`

2. **puckweb** - Web application / Demo site
   - ASP.NET Core MVC web application
   - Contains example ViewModels (`ViewModels/`)
   - Controllers, Views, Areas
   - Static content (`wwwroot/`)
   - Target: `net6.0`

3. **AzureDirectory** - Azure Lucene storage support
   - Handles Azure blob storage for Lucene indexes

4. **tests** - Unit tests
   - NUnit tests (`ContentServiceTests.cs`)
   - Uses Moq for mocking
   - Test configuration in `appSettings.json`

## First-Time Setup

### 1. Configure Database
Edit `puckweb/appSettings.json` to select your database provider:

**For SQLite (recommended for development/macOS):**
```json
{
  "ConnectionStrings": {
    "SQLite": "Data Source=puck.db"
  },
  "UseSQLServer": false,
  "UsePostgreSQL": false,
  "UseMySQL": false,
  "UseSqlite": true
}
```

**For SQL Server:**
```json
{
  "ConnectionStrings": {
    "SQLServer": "Server=(localdb)\\mssqllocaldb;Database=PuckCMS;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "UseSQLServer": true,
  "UsePostgreSQL": false,
  "UseMySQL": false,
  "UseSqlite": false
}
```

### 2. Configure Admin User
Set initial admin credentials in `puckweb/appSettings.json`:
```json
{
  "InitialUserEmail": "admin@localhost.com",
  "InitialUserPassword": "YourSecureP@ssw0rd1!"
}
```

**Important:** Clear these values after first login for security.

Password requirements:
- At least 6 characters
- At least one non-alphanumeric character
- At least one digit ('0'-'9')
- At least one uppercase ('A'-'Z')

### 3. Cross-Platform Build Support
The project includes PreBuild/PostBuild events that are now cross-platform compatible:
- **Windows**: Uses `echo /a >` and `del` commands
- **macOS/Linux**: Uses `echo >` and `rm -f` commands

## Common Commands

### Build
```bash
dotnet build puckcore.sln
```

### Run the web application
```bash
dotnet run --project puckweb/puckweb.csproj
```

Run on a specific port (if default is in use):
```bash
dotnet run --project puckweb/puckweb.csproj --urls "http://localhost:5001"
```

**Access the application:**
- Backoffice (admin): `http://localhost:5000/puck` (or your configured port)
- Login with the credentials configured in `InitialUserEmail` and `InitialUserPassword`
- **Important**: You must set up content and domain mapping in the backoffice before the frontend will work

### Initial Setup Steps (First Run)

After starting the application for the first time, you need to configure it through the backoffice:

1. **Access the backoffice**: Navigate to `http://localhost:5000/puck` (or your configured port)

2. **Login**: Use the credentials from `InitialUserEmail` and `InitialUserPassword` in appSettings.json

3. **Create root content**:
   - In the backoffice, create your first page/content node
   - This will serve as your site's homepage

4. **Set domain mapping**:
   - Right-click on your root content node in the content tree
   - Select "Set Domain Mapping"
   - Map your domain (e.g., `localhost:5000` or `localhost:5050`) to this root node
   - This tells Puck which content to serve for each domain

5. **Access frontend**: After setting up domain mapping, you can access the frontend at `http://localhost:5000`

**Common error**: If you see "domain root not set" when accessing the frontend, it means you haven't completed steps 3-4 above. Visit the backoffice to set up your site first.

### Run tests
```bash
dotnet test tests/tests.csproj
```

### Run specific test
```bash
dotnet test tests/tests.csproj --filter "FullyQualifiedName~TestMethodName"
```

### Database migrations (from core/ directory)
```bash
# Add migration for SQL Server
dotnet ef migrations add MigrationName --context DbContextSQLServer --output-dir Migrations/SQLServer

# Add migration for PostgreSQL
dotnet ef migrations add MigrationName --context DbContextPostgreSQL --output-dir Migrations/PostgreSQL

# Add migration for MySQL
dotnet ef migrations add MigrationName --context DbContextMySQL --output-dir Migrations/MySQL

# Add migration for SQLite
dotnet ef migrations add MigrationName --context DbContextSQLite --output-dir Migrations/SQLite
```

## Architecture

### Database Provider Selection
Database provider is configured in `puckweb/appSettings.json` via boolean flags:
- `UseSQLServer`, `UsePostgreSQL`, `UseMySQL`, `UseSqlite`
- Connection strings defined in `ConnectionStrings` section
- Startup.cs conditionally registers the appropriate DbContext based on these flags

### Content Storage & Querying
- Content is stored in database but **queried via Lucene** for performance
- Lucene index path configured via `LuceneIndexPath` in appSettings
- `PuckLucene/` in core project handles all indexing operations
- ViewModels can specify field analyzers and settings per property

### ViewModels & Content Types
- ViewModels in `puckweb/ViewModels/` define content types (e.g., Page, Homepage, Folder, ImageVM)
- ViewModels are POCOs decorated with attributes
- Properties can use transformer attributes (e.g., for image/file handling)
- Editor templates (standard ASP.NET MVC) define edit UI

### Request Pipeline
- `puck.core.Bootstrap.Ini()` initializes Puck in Startup.cs Configure()
- Catch-all route `{**path}` in Startup.cs routes all requests through HomeController
- HomeController handles content resolution and template rendering
- Display modes can be configured (e.g., iPhone detection)

### Media & Image Processing
- Uses SixLabors.ImageSharp for image processing
- Supports local file system or Azure Blob Storage for media
- Image transformers handle file upload/processing before indexing
- ImageSharp middleware handles dynamic image resizing/cropping via query parameters

### Multi-tenancy & Multilingual
- Multi-site: Multiple site roots mapped to different domains
- Multilingual: Languages associated with nodes recursively
- Each content node may have translations

### Background Tasks
- Task API in `core/Tasks/` supports one-off and recurring tasks
- Configured via `PuckUpdateTaskLastRun`, `PuckUpdateRecurringTaskLastRun`, `PuckTaskCatchUp` in appSettings

### Load Balancing
- Syncs between servers in load balanced environments
- `IsEditServer` flag in appSettings determines edit vs delivery servers

## Configuration

Key settings in `puckweb/appSettings.json`:
- **Database selection**: `UseSQLServer`, `UsePostgreSQL`, etc.
- **Initial admin**: `InitialUserEmail`, `InitialUserPassword` (clear after first login)
- **Backoffice path**: `/puck`
- **Revisions**: `MaxRevisions` controls version history
- **Azure**: Settings for Azure blob storage and Azure Lucene directory
- **SMTP**: Email notification configuration
- **Error templates**: Custom 404/500 page paths

## Development Notes

### When working with content types (ViewModels):
- Add new ViewModels to `puckweb/ViewModels/`
- Use attributes to control indexing, storage, analyzers
- Create corresponding Razor views in `Views/`
- Create editor templates if custom edit UI needed

### When working with the core library:
- Abstract interfaces in `core/Abstract/`
- Implementations in `core/Concrete/`
- Lucene-related code in `core/PuckLucene/`
- Be aware of multi-database support - test changes across providers

### When modifying startup/configuration:
- Bootstrap logic in `puck.core.Bootstrap.Ini()`
- Startup.cs handles DI registration via `AddPuckServices<>()` extension
- Different DbContext per database provider

### Image/File handling:
- Expose `IFormFile` properties in ViewModels
- Use transformer attributes (e.g., `PuckImageTransformer`, `PuckAzureBlobImageTransformer`)
- Transformers execute before indexing

## Testing

Tests use:
- NUnit as test framework
- Moq for mocking
- In-memory database for testing
- Test configuration in `tests/appSettings.json`

## Troubleshooting

### Build Issues

**Error: `del` or `echo /a` command not found (macOS/Linux)**
- The PreBuild/PostBuild events have been updated to be cross-platform compatible
- If you encounter this error, ensure `puckweb/puckweb.csproj` has conditional build targets based on OS

**Error: Connection string is empty**
- Ensure you've configured a connection string in `puckweb/appSettings.json`
- Make sure the corresponding `Use[DatabaseType]` flag is set to `true`
- For quick development, use SQLite with: `"SQLite": "Data Source=puck.db"` and `"UseSqlite": true`

### Runtime Issues

**Error: "domain root not set"**
- This is expected on first run - there's no content yet
- **Solution**:
  1. Go to the backoffice at `/puck` (e.g., `http://localhost:5000/puck`)
  2. Login with your configured credentials
  3. Create a root content node
  4. Right-click on the node and select "Set Domain Mapping"
  5. Map your domain (e.g., `localhost:5000`) to this root node
- The frontend will only work after domain mapping is configured

**Error: Port already in use**
- Run on a different port: `dotnet run --project puckweb/puckweb.csproj --urls "http://localhost:5001"`
- Or find and stop the process using port 5000

**Database migrations not applied**
- Migrations run automatically on first startup
- Check console output for migration status
- Database file (for SQLite) will be created in the puckweb directory as `puck.db`

### macOS-Specific Notes

- SQLite is recommended for development on macOS (no additional database server required)
- The project builds and runs successfully on macOS with .NET 6.0
- Lucene index will be stored in `puckweb/App_Data/Lucene[MachineName]/`
