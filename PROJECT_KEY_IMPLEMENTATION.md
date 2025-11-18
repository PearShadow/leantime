# Project Key Feature Implementation

## Overview
This implementation adds a Jira-style project key feature to Leantime. Each project now has a unique key (e.g., "FL" for "Fiesta Lama") that can be auto-generated or manually customized by users.

## Features Implemented

### 1. Database Changes
- **New Column**: Added `project_key` (VARCHAR(10)) to the `zp_projects` table
- **Unique Index**: Created unique index on `project_key` to ensure no duplicates
- **Migration**: Created migration `update_sql_30410()` that:
  - Adds the column and index
  - Auto-generates keys for existing projects
  - Handles collision detection with numeric suffixes

### 2. Backend Implementation

#### Models
- **Project.php**: Added `projectKey` property to the Project model with support for both camelCase and snake_case formats

#### Services
- **Projects.php**: Added three new public methods:
  - `generateProjectKey($projectName, $projectId)` - Generates a key from project name using first letters of each word
  - `isProjectKeyTaken($key, $excludeProjectId)` - Checks if a key is already in use
  - `validateProjectKey($key, $projectId)` - Validates format and uniqueness of a key

#### Repositories
- **Projects.php**: 
  - Updated `addProject()` to handle `project_key` column
  - Updated `editProject()` to handle `project_key` column
  - Added `isProjectKeyTaken()` method for database lookups

#### Controllers
- **NewProject.php**: 
  - Auto-generates key from project name if not provided
  - Validates key before creating project
  - Returns errors if validation fails
  
- **ShowProject.php**: 
  - Auto-generates key from project name if not provided during edit
  - Validates key before updating project
  - Returns errors if validation fails

### 3. Frontend Implementation

#### Templates
- **newProject.tpl.php**: 
  - Added project key input field with auto-generation
  - JavaScript to auto-generate key from project name on blur
  - JavaScript to enforce uppercase and alphanumeric characters only
  - Shows validation hints inline

- **projectDetails.sub.php** (used in showProject.tpl.php):
  - Added project key input field for editing
  - JavaScript to enforce uppercase and alphanumeric characters only
  - Shows current key value

### 4. Language Strings
- **en-US.ini**: Added translations for:
  - `label.project_key` - "Project Key"
  - `label.project_key_description` - Description of the feature
  - `input.placeholders.auto_generated` - Placeholder text
  - `input.placeholders.enter_project_key` - Example text
  - `validation.project_key_required` - Required field error
  - `validation.project_key_length` - Length validation error
  - `validation.project_key_format` - Format validation error
  - `validation.project_key_taken` - Uniqueness validation error

## Key Generation Algorithm

The auto-generation logic works as follows:

1. Takes the project name (e.g., "Fiesta Lama")
2. Removes special characters and normalizes spaces
3. Takes the first letter of each word (F + L = "FL")
4. If the result is less than 2 characters, uses first 2-3 characters of the name
5. Limits the key to 10 characters maximum
6. Converts to uppercase
7. Checks for uniqueness; if taken, adds a numeric suffix (FL1, FL2, etc.)

## Validation Rules

- **Length**: 2-10 characters
- **Format**: Uppercase letters and numbers only (A-Z, 0-9)
- **Uniqueness**: Must be unique across all projects
- **Optional**: If left empty, will be auto-generated

## User Experience

### Creating a New Project
1. User enters project name
2. System auto-generates a key when they blur from the name field
3. User can edit the generated key if desired
4. System validates the key on form submission
5. Shows error if key is invalid or already taken

### Editing an Existing Project
1. Existing projects without keys will have keys auto-generated during migration
2. Users can view and edit the key
3. System validates on save
4. Shows error if new key is invalid or already taken

## Migration Process

To apply this update to an existing installation:

```bash
php bin/leantime db:migrate
```

This will:
1. Add the `project_key` column to the `zp_projects` table
2. Create a unique index on the column
3. Auto-generate keys for all existing projects
4. Handle any conflicts by adding numeric suffixes

## Files Modified

### Backend Files
- `app/Domain/Install/Repositories/Install.php` - Database migration and schema
- `app/Domain/Projects/Models/Project.php` - Added projectKey property
- `app/Domain/Projects/Services/Projects.php` - Added key generation and validation
- `app/Domain/Projects/Repositories/Projects.php` - Database operations
- `app/Domain/Projects/Controllers/NewProject.php` - Create project logic
- `app/Domain/Projects/Controllers/ShowProject.php` - Edit project logic

### Frontend Files
- `app/Domain/Projects/Templates/newProject.tpl.php` - New project form
- `app/Domain/Projects/Templates/submodules/projectDetails.sub.php` - Edit project form

### Language Files
- `app/Language/en-US.ini` - English translations

## Testing Checklist

- [ ] Run database migration successfully
- [ ] Create a new project - verify key is auto-generated
- [ ] Create a new project with custom key - verify it's saved
- [ ] Try to create project with duplicate key - verify error message
- [ ] Try to create project with invalid characters - verify error message
- [ ] Edit existing project and change key - verify it's updated
- [ ] Edit existing project with duplicate key - verify error message
- [ ] Verify existing projects have keys after migration
- [ ] Verify projects with same name get different keys (FL, FL1, FL2, etc.)

## Future Enhancements

Potential improvements for future versions:

1. Display project key in project listings and selectors
2. Use project key in URLs (e.g., /projects/FL/tickets instead of /projects/123/tickets)
3. Add project key to ticket IDs (e.g., FL-123, FL-124)
4. Add API endpoint to check key availability in real-time
5. Allow searching projects by key
6. Add project key to exports and reports

## Bug Fixes Applied

During implementation, several bugs were identified and fixed:

### 1. Date Handling in Controller (NewProject.php)
**Issue**: When creating projects without start/end dates, empty strings were being passed to Carbon date formatter, causing "Unsupported operand types" error.

**Fix**: Added proper checks before formatting dates:
```php
'start' => (isset($_POST['start']) && ! empty($_POST['start'])) 
    ? format(value: $_POST['start'], fromFormat: FromFormat::UserDateStartOfDay)->isoDateTime() 
    : '',
```

### 2. Date Handling in Service Layer (Projects.php)
**Issue**: Service layer was checking `!= null` but empty strings `''` pass that check, causing Carbon errors.

**Fix**: Added check for both null AND empty strings:
```php
if ($values['start'] != null && $values['start'] !== '') {
    $values['start'] = format(value: $values['start'], fromFormat: FromFormat::UserDateStartOfDay)->isoDateTime();
}
```

### 3. Validation Flow Logic (NewProject.php)
**Issue**: Project creation code was in an `else` block that only executed when `$projectKey` was empty, meaning projects with custom keys were never created!

**Fix**: Restructured validation flow:
- Validate name and client first
- Then validate project key if provided (nested)
- Create project when ALL validations pass

## Notes

- The `project_key` column accepts NULL values for backward compatibility
- The unique index only enforces uniqueness on non-NULL values
- Keys are case-insensitive (always stored and displayed in uppercase)
- The 10-character limit balances readability with flexibility
- The migration is idempotent and can be run multiple times safely
- Empty date fields are properly handled throughout the application flow


