# PetaPoco DB Entity Generator - Claude Documentation

## Project Overview

**PetaPoco.DBEntityGenerator** is a command-line tool that generates C# entity classes from database schemas for use with PetaPoco ORM. This tool was created as a replacement for the deprecated T4 template system that was removed in PetaPoco Version 6.

**Current Version:** 0.1.4.1
**Target Framework:** .NET 9.0
**License:** See LICENSE file

## Purpose

Automatically generate strongly-typed C# classes that map to database tables, eliminating manual entity creation and reducing errors. The tool connects to your database, reads the schema, and generates POCO (Plain Old CLR Objects) classes with proper PetaPoco attributes.

## Supported Databases

- **PostgreSQL** (Npgsql)
- **SQL Server** (Microsoft.Data.SqlClient)
- **MySQL** (MySql.Data)
- **Oracle** (Oracle.ManagedDataAccess)

## Key Features

1. **Multi-database Support** - Works with PostgreSQL, SQL Server, MySQL, and Oracle
2. **Flexible Configuration** - JSON-based or command-line configuration
3. **Column Tracking** - Optional modified column tracking for change detection
4. **Table Customization** - Rename tables, ignore columns, add custom attributes
5. **View Support** - Optional inclusion of database views
6. **Custom Templates** - SQL insert/update template customization per column
7. **Multiple Output Options** - Console or file output

## Architecture & Code Structure

### Core Components

```
PetaPoco.DBEntityGenerator/
├── Program.cs                    # Entry point, command-line parsing
├── Generator.cs                  # Main generation logic
├── GenerateCommand.cs           # Configuration model
├── GenerateContext.cs           # Generation context wrapper
├── ProgramOptions.cs            # CLI options model
├── Helpers.cs                   # Utility functions
├── Inflector.cs                 # String pluralization/singularization
├── Descriptions.cs              # Database schema models (Table, Column, etc.)
├── SchemaReaders/               # Database-specific schema readers
│   ├── SchemaReader.cs          # Base abstract reader
│   ├── PostgreSqlSchemaReader.cs
│   ├── SqlServerSchemaReader.cs
│   ├── MySqlSchemaReader.cs
│   └── OracleSchemaReader.cs
└── Outputs/                     # Output handlers
    ├── IOutput.cs               # Output interface
    ├── ConsoleOutput.cs         # Console output
    └── FileOutput.cs            # File output
```

### Key Classes

#### Generator (`Generator.cs`)
- **Location:** `PetaPoco.DBEntityGenerator/Generator.cs`
- **Purpose:** Core generation engine
- **Key Methods:**
  - `Generate(GenerateCommand cmd)` - Main entry point (line 20)
  - `LoadTables(GenerateContext context)` - Reads database schema (line 46)
  - `ApplyTableConfigurations()` - Applies JSON config customizations (line 180)
  - `WriteTable()` - Generates class code for a table (line 335)

#### SchemaReaders
- **Location:** `PetaPoco.DBEntityGenerator/SchemaReaders/`
- **Purpose:** Database-specific schema extraction
- **Pattern:** Factory pattern - Reader selected based on DbProviderFactory type (lines 97-121 in Generator.cs)

#### Program (`Program.cs`)
- **Entry Point:** `Main(string[] args)` at line 16
- **Key Features:**
  - Registers DbProviderFactories for .NET Core (lines 18-23)
  - Parses CLI arguments using CommandLineParser library
  - Supports both JSON config file and CLI parameters

## Usage

### Installation

```bash
# Via NuGet
dotnet tool install PetaPoco.DBEntityGenerator

# Or install as local tool
dotnet add package PetaPoco.DBEntityGenerator
```

### Running the Tool

#### Using .NET 6.0+
```bash
dotnet ~/.nuget/packages/petapoco.dbentitygenerator/0.1.4.1/tools/net9.0/PetaPoco.DBEntityGenerator.dll [options]
```

#### Command-Line Parameters

```
--config <path>              # Path to JSON config file
-p, --providerName <name>    # Database provider: Npgsql, SqlServer, MySql, Oracle
-c, --connectionString <cs>  # Database connection string
--namespace <ns>             # Namespace for generated classes (default: Entities)
--explicitColumns <bool>     # Use [ExplicitColumns] attribute (default: true)
--trackModifiedColumns <bool># Track modified columns (default: true)
-o, --output <type>          # Output type: console or file (default: console)
--outputFile <path>          # Output file path (default: Database.cs)
```

### Configuration File

The tool supports a JSON configuration file for advanced customization:

```json
{
  "ProviderName": "Npgsql",
  "ConnectionString": "Host=localhost;Database=mydb;Username=user;Password=pass",
  "SchemaName": "public",
  "ClassPrefix": "",
  "ClassSuffix": "",
  "IncludeViews": false,
  "ExcludePrefix": ["temp_", "tmp_"],
  "Namespace": "MyApp.Entities",
  "UsingNamespaces": ["System.ComponentModel.DataAnnotations"],
  "ExplicitColumns": true,
  "TrackModifiedColumns": true,
  "Tables": {
    "public.users": {
      "Ignore": false,
      "ClassName": "User",
      "Columns": {
        "user_data": {
          "Ignore": false,
          "PropertyName": "UserData",
          "PropertyType": "string",
          "CustomAttributes": ["[JsonProperty]"],
          "InsertTemplate": "{0}{1}::JSONB",
          "UpdateTemplate": "{0} = {1}{2}::JSONB"
        }
      }
    }
  }
}
```

**Configuration Options:**
- `ProviderName` - Database provider name
- `ConnectionString` - Database connection string
- `SchemaName` - Filter by schema (optional)
- `ClassPrefix`/`ClassSuffix` - Add prefix/suffix to class names
- `IncludeViews` - Include database views (default: false)
- `ExcludePrefix` - Array of table name prefixes to exclude
- `Namespace` - Target namespace for generated code
- `UsingNamespaces` - Additional using statements
- `Tables` - Per-table customizations (see sample_config.json)

### Per-Table Customization

```json
"Tables": {
  "schema.table_name": {
    "Ignore": false,              // Skip this table
    "ClassName": "CustomName",     // Override class name
    "Columns": {
      "column_name": {
        "Ignore": false,           // Skip this column
        "PropertyName": "Name",    // Override property name
        "PropertyType": "string",  // Override property type
        "ForceToUtc": true,        // Force DateTime to UTC
        "CustomAttributes": [...], // Add custom attributes
        "InsertTemplate": "...",   // SQL insert template
        "UpdateTemplate": "..."    // SQL update template
      }
    }
  }
}
```

## Generated Code Structure

### With TrackModifiedColumns = true

```csharp
namespace MyApp.Entities
{
    using System;
    using System.Collections.Generic;
    using PetaPoco;

    public partial class Record<T> where T : new()
    {
        private Dictionary<string, bool> modifiedColumns = null;

        [Ignore]
        public Dictionary<string, bool> ModifiedColumns { get { return modifiedColumns; } }

        protected void MarkColumnModified(string column_name)
        {
            // Tracks which columns have been modified
        }
    }

    [TableName("users")]
    [PrimaryKey("id")]
    [ExplicitColumns]
    public partial class User : Record<User>
    {
        [Column]
        public int Id
        {
            get { return _Id; }
            set
            {
                _Id = value;
                MarkColumnModified("Id");
            }
        }
        private int _Id;

        [Column]
        public string Name
        {
            get { return _Name; }
            set
            {
                _Name = value;
                MarkColumnModified("Name");
            }
        }
        private string _Name;
    }
}
```

### With TrackModifiedColumns = false

```csharp
[TableName("users")]
[PrimaryKey("id")]
[ExplicitColumns]
public partial class User
{
    [Column]
    public int Id { get; set; }

    [Column]
    public string Name { get; set; }
}
```

## Common Development Patterns

### Adding Support for a New Database

1. Create a new SchemaReader in `SchemaReaders/` directory
2. Inherit from `SchemaReader` base class
3. Override `ReadSchema(DbConnection connection, DbProviderFactory factory)` method
4. Add factory detection logic in `Generator.cs:LoadTables()` (around line 97-121)
5. Register the DbProviderFactory in `Program.cs:Main()` if needed

### Modifying Code Generation

The code generation logic is in `Generator.cs`:
- `WriteFileStarting()` - Generates namespace and using statements (line 256)
- `WriteRecoredClass()` - Generates base Record<T> class (line 288)
- `WriteTable()` - Generates entity class for each table (line 335)
- `WriteFileEnding()` - Closes namespace (line 282)

### Customizing Output

Implement the `IOutput` interface:
```csharp
public interface IOutput : IDisposable
{
    void WriteLine(string text);
}
```

Examples: `ConsoleOutput.cs`, `FileOutput.cs`

## Dependencies

### NuGet Packages
- **CommandLineParser** (2.9.1) - CLI argument parsing
- **Newtonsoft.Json** (13.0.4) - JSON configuration
- **Npgsql** (9.0.4) - PostgreSQL support
- **Microsoft.Data.SqlClient** (6.1.2) - SQL Server support
- **MySql.Data** (9.4.0) - MySQL support
- **Oracle.ManagedDataAccess.Core** (23.9.1) - Oracle support (.NET Core)
- **Oracle.ManagedDataAccess** (19.18.0) - Oracle support (.NET Framework)

## Key Implementation Details

### SQL Identifier Escaping
- **PostgreSQL:** Double quotes `"identifier"`
- **MySQL:** Backticks `` `identifier` ``
- **Oracle:** Double quotes uppercase `"IDENTIFIER"`
- **SQL Server:** No escaping by default

Set in `Generator.cs:LoadTables()` via `context.EscapeSqlIdentifier`

### Property Name Collision Prevention
The generator prevents naming conflicts (Generator.cs:148-159):
- Renames properties that match PetaPoco method names (Equals, GetHashCode, Save, etc.)
- Prevents property names matching class names
- Pattern: Prefix with underscore `_PropertyName`

### Modified Column Tracking
When `TrackModifiedColumns = true`:
- Each property has a private backing field
- Setter calls `MarkColumnModified(propertyName)`
- `ModifiedColumns` dictionary tracks changes
- Useful for partial updates and change tracking

### Connection String Security
Helper method `Helpers.zap_password()` removes passwords from connection strings in generated file headers (line 27 in Generator.cs)

## Examples

### Example 1: Quick Console Output
```bash
dotnet PetaPoco.DBEntityGenerator.dll \
  -p Npgsql \
  -c "Host=localhost;Database=mydb;Username=user;Password=pass" \
  --namespace MyApp.Data \
  -o console
```

### Example 2: Generate to File
```bash
dotnet PetaPoco.DBEntityGenerator.dll \
  -p SqlServer \
  -c "Server=localhost;Database=mydb;Trusted_Connection=True" \
  --namespace MyApp.Entities \
  -o file \
  --outputFile Entities.cs
```

### Example 3: Using Config File
```bash
dotnet PetaPoco.DBEntityGenerator.dll \
  --config config.json \
  -o file \
  --outputFile Database.cs
```

## Troubleshooting

### Common Issues

1. **Provider not found**: Ensure DbProviderFactory is registered (see Program.cs:18-23)
2. **Connection failed**: Verify connection string format for your database
3. **Missing tables**: Check schema name and ExcludePrefix settings
4. **Property name conflicts**: Use config file to rename properties
5. **Type mapping issues**: Override PropertyType in column configuration

### Debug Information

Generated files include connection info at the top:
```csharp
// Connection String: `Host=localhost;Database=***`
// Provider:          `Npgsql`
```

## Version History

- **0.1.4.1** - Current version, .NET 9.0 target, updated dependencies
- **0.1.3** - Previous stable release
- See git history for detailed changes

## Related Resources

- [PetaPoco Documentation](https://github.com/CollaboratingPlatypus/PetaPoco)
- [Original T4 Template](https://github.com/CollaboratingPlatypus/PetaPoco/tree/v5)
- [NuGet Package](https://www.nuget.org/packages/PetaPoco.DBEntityGenerator/)

## Contributing

When modifying the generator:
1. Test with all supported databases
2. Ensure backwards compatibility with existing configs
3. Update sample_config.json with new features
4. Follow existing code patterns and naming conventions

## File Generation Header

All generated files include this header (Generator.cs:22-28):
```csharp
// <auto-generated />
// This file was automatically generated by the PetaPocoGenerator
//
// The following connection settings were used to generate this file
//
// Connection String: `[sanitized]`
// Provider:          `[provider name]`
```
