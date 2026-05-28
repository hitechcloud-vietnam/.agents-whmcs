# WHMCS Migration Tool Module

Data migration tools with pre-migration checks, data mapping, validation, and rollback support.

## Features

- Multiple data sources (WHMCS, CSV, JSON)
- Field mapping and transformation
- Batch processing
- Progress tracking
- Validation and verification
- Rollback support
- Detailed logging
- Error handling and retries

## Installation

1. Copy `migrationtool.php` to `/path/to/whmcs/modules/addons/migrationtool/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure batch settings

## Usage

```php
// Create migration project
$result = migrationtool_CreateProject(array(
    'project_name' => 'Client Data Migration',
    'project_key' => 'client_migration_2026',
    'source_type' => 'whmcs',
    'target_type' => 'whmcs',
    'created_by' => $adminId
));

// Add field mapping
migrationtool_AddMapping($projectId, array(
    'source_entity' => 'clients',
    'target_entity' => 'tblclients',
    'field_mappings' => array(
        'firstname' => 'firstname',
        'lastname' => 'lastname',
        'email' => 'email',
        'companyname' => 'companyname',
        'created_at' => 'datecreated'
    ),
    'transformations' => array(
        'email' => array(
            array('type' => 'lowercase'),
            array('type' => 'trim')
        ),
        'firstname' => array(
            array('type' => 'uppercase'),
            array('type' => 'trim')
        )
    ),
    'filters' => array('status' => 'Active')
));

// Add mapping for hosting
migrationtool_AddMapping($projectId, array(
    'source_entity' => 'hosting',
    'target_entity' => 'tblhosting',
    'field_mappings' => array(
        'domain' => 'domain',
        'username' => 'username',
        'package_id' => 'packageid',
        'reg_date' => 'regdate'
    )
));

// Run pre-migration checks
$result = migrationtool_RunPreMigrationChecks($projectId);
// Returns: success, passed, checks

// Prepare migration (creates batches)
$result = migrationtool_PrepareMigration($projectId);
// Returns: success, total_records, batch_count

// Execute migration
$result = migrationtool_ExecuteMigration($projectId);
// Returns: success, project_id

// Resume interrupted migration
$result = migrationtool_ExecuteMigration($projectId, true);

// Validate migration results
$validation = migrationtool_ValidateMigration($projectId);
// Returns: success, results (per-entity counts)

// Get progress
$progress = migrationtool_GetProgress($projectId);
// Returns: project, total_batches, completed_batches, progress_percent, running_batch

// Get logs
$logs = migrationtool_GetLogs($projectId, 100);

// Rollback migration
$result = migrationtool_Rollback($projectId);
// Returns: success, restored

// Get projects
$projects = migrationtool_GetProjects();
$activeProjects = migrationtool_GetProjects('running');

// Get specific project
$project = migrationtool_GetProject($projectId);

// Get mappings
$mappings = migrationtool_GetMappings($projectId);

// Delete project
migrationtool_DeleteProject($projectId);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| ChunkSize | text | 1000 | Records per chunk |
| BatchDelay | text | 1 | Delay between batches (seconds) |
| EnableValidation | yesno | yes | Validate after migration |
| EnableRollback | yesno | yes | Enable rollback |
| MaxRetries | text | 3 | Max retry attempts |
| Timeout | text | 300 | Timeout (seconds) |

## Field Transformations

```php
// Uppercase
'transformations' => array(
    'field_name' => array(
        array('type' => 'uppercase')
    )
)

// Lowercase
'transformations' => array(
    'email' => array(
        array('type' => 'lowercase')
    )
)

// Trim whitespace
'transformations' => array(
    'name' => array(
        array('type' => 'trim')
    )
)

// MD5 hash
'transformations' => array(
    'password' => array(
        array('type' => 'md5')
    )
)

// Custom hash
'transformations' => array(
    'data' => array(
        array('type' => 'hash', 'algorithm' => 'sha256')
    )
)

// Date format
'transformations' => array(
    'date' => array(
        array('type' => 'date_format', 'format' => 'Y-m-d')
    )
)

// Default value
'transformations' => array(
    'status' => array(
        array('type' => 'default', 'value' => 'Active')
    )
)

// Value lookup
'transformations' => array(
    'country_code' => array(
        array('type' => 'lookup', 'map' => array('US' => 'USA', 'UK' => 'United Kingdom'))
    )
)

// Concatenate
'transformations' => array(
    'fullname' => array(
        array('type' => 'concat', 'template' => '%s Jr.', 'placeholder' => '%s')
    )
)

// Multiple transformations
'transformations' => array(
    'email' => array(
        array('type' => 'trim'),
        array('type' => 'lowercase')
    )
)
```

## Supported Entities

| Entity | Table |
|--------|-------|
| clients | tblclients |
| hosting | tblhosting |
| domains | tbldomains |
| invoices | tblinvoices |
| tickets | tbltickets |
| products | tblproducts |
| addons | tbladdons |
| pricing | tblpricing |

## Filters

```php
'filters' => array(
    'status' => 'Active',
    'datecreated' => '>=2026-01-01'
)

// Multiple filters
'filters' => array(
    'status' => 'Active',
    'packageid' => 5
)
```

## Project Status

| Status | Description |
|--------|-------------|
| draft | Not started |
| prepared | Ready to run |
| running | In progress |
| completed | Finished |
| failed | Failed |
| rolled_back | Rolled back |

## Pre-Migration Checks

| Check | Description |
|-------|-------------|
| project_exists | Project valid |
| has_mappings | Mappings defined |
| target_tables_exist | Target tables exist |
| source_data_exists | Source data available |
| field_compatibility | Fields compatible |
| disk_space | Sufficient disk space |

## Migration Flow

```
1. Create Project
   ↓
2. Add Mappings
   ↓
3. Run Pre-Migration Checks
   ↓
4. Prepare Migration (create batches)
   ↓
5. Execute Migration
   ↓
6. Validate Results
   ↓
7. Complete or Rollback
```

## Database Tables

- `mod_migrationtool_projects` - Migration projects
- `mod_migrationtool_mappings` - Field mappings
- `mod_migrationtool_batches` - Processing batches
- `mod_migrationtool_records` - Individual records
- `mod_migrationtool_logs` - Migration logs
- `mod_migrationtool_validation` - Validation results
- `mod_migrationtool_preserves` - Rollback data
