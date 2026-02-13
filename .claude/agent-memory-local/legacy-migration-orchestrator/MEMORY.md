# Legacy Migration Orchestrator - Memory

## Successful Migration Patterns

### Java/Swing to React 18 Component Mappings
- **csFleetCombo** → React Select with API data loading + React Query
- **csFleetChapterCombo** → React Select with dynamic filtering
- **csJButton** → Material-UI Button or standard React button
- **ChapterAlertRates (EditableTable)** → AG-Grid or Material-UI DataGrid with custom cell editors
- **csButtons1** → Button group with Save/Cancel/Close actions
- **csInfoPanel** → Material-UI Alert/Snackbar for messages
- **Multi-line cells (XML-tagged)** → JSON array expansion with custom React component

### Oracle PL/SQL to PostgreSQL Function Patterns
- **SYS_REFCURSOR returns** → PostgreSQL RETURNS TABLE
- **VARCHAR2_ARRAY parameters** → PostgreSQL VARCHAR[] array type
- **Nested arrays (VARCHAR2_ARRAY[])** → PostgreSQL VARCHAR[][] double array
- **SAVEPOINT rollback patterns** → PostgreSQL SAVEPOINT with ROLLBACK TO
- **XML-tagged multi-line data (editable_table_helper_pkg)** → JSONB with jsonb_agg and jsonb_build_object
- **OUT parameters** → PostgreSQL RETURNS TABLE with multiple columns
- **Oracle MERGE/UPSERT** → PostgreSQL INSERT ... ON CONFLICT DO UPDATE

### Business Logic Preservation Techniques
1. **State flags** (afterAddChapter, weMadeAnEdit) → useState hooks in React
2. **Field enable/disable coordination** → Controlled component disabled prop
3. **Dirty state tracking** → useState + useEffect dependency arrays
4. **Validation on edit** → React Hook Form + Zod/Yup schemas + backend FluentValidation
5. **Multi-step workflows** (add → reload → save defaults) → State machine pattern or sequential async operations
6. **Default values (999.9)** → Database defaults + application-level defaults in DTOs

### Data Migration Strategies
- **CHAR(N) to VARCHAR** type conversions preserve data, increase flexibility
- **NUMBER(precision, scale)** → DECIMAL(precision, scale) direct mapping
- **Composite keys** preserved in EF Core with [Key, Order] attributes
- **Audit fields** (LAST_UPDATED, UPDATED_BY) implemented with PostgreSQL triggers + EF Core interceptors
- **Foreign key cascades** explicitly defined for DELETE and UPDATE

## Common Challenges and Solutions

### Challenge: Multi-line Editable Cells
**Legacy:** XML-tagged strings parsed by editable_table_helper_pkg
**Solution:** JSONB arrays in PostgreSQL, custom React component for expand/collapse, AG-Grid cell renderers

### Challenge: COBOL Program Dependencies
**Legacy:** g1lc.fleetChapterCodes (external COBOL)
**Solution:** Replace with direct PostgreSQL function get_available_chapters, expose via REST API

### Challenge: Transactional Integrity Across Multiple Tables
**Legacy:** Oracle PL/SQL with SAVEPOINT and ROLLBACK
**Solution:** PostgreSQL function with SAVEPOINT, try/catch exception handling, explicit transaction management in repository layer

### Challenge: Optimistic Concurrency Control
**Legacy:** Implicit via Oracle locking
**Solution:** LAST_UPDATED timestamp with [ConcurrencyCheck] attribute in EF Core, HTTP 409 Conflict responses

### Challenge: Security and Audit
**Legacy:** SecurityPermissionEnum checks, csBase.getTopsyLogName()
**Solution:** ASP.NET Core Authorization attributes, ClaimsPrincipal for user context, middleware for audit logging

## Project-Specific Conventions

### Metadata File Structure
```
metadata/
  FORMS/
    {FORM_ID}/
      FORM_DOCS/           # Requirements, descriptions
      FORM_FILE_DEPENDENCIES/  # Dependency lists
  LEGACY_CODEBASE/
    oases-master/src/main/
      java/                # Java source
      database/            # Oracle DDL and PL/SQL
  DB_PRD/                  # Production database schemas
  EXPORT_CODEBASE_PRD/     # Exported artifacts
```

### Field Descriptor (FD) Naming
- Legacy FDs like "LEEKFL" (fleet) map to camelCase properties (fleet)
- "Z_" prefix indicates composite/calculated fields (Z_REPORT_CLASSIFICATION)
- Descriptions use "DES" suffix (LEBDES = chapter description)

### Database Naming Conventions
- Tables: lowercase with underscores (fleet_chapter, chapter_alert_rates)
- Columns: lowercase with underscores (sys_alert_rate, tech_log_type)
- Constraints: prefixed (pk_, fk_, chk_, unq_)
- Indexes: prefixed idx_
- Functions: snake_case (get_chapter_alert_rates)

### .NET Naming Conventions
- Entities: PascalCase, singular (FleetChapter, not FleetChapters)
- DTOs: Suffix with "Dto" (ChapterAlertRateDto)
- Requests: Suffix with "Request" (SaveChapterAlertRatesRequest)
- Interfaces: Prefix with "I" (IChapterAlertRatesRepository)
- Private fields: _camelCase with underscore prefix

### React Naming Conventions
- Components: PascalCase (FleetSelector.tsx)
- Hooks: prefix with "use" (useChapterAlertRates.ts)
- Types: PascalCase, suffix with "Data" or "Request" (ChapterAlertRateData)
- API services: camelCase object with methods (chapterAlertRatesApi)

## Key File Paths
- Migration output: `/migrations/{FORM_ID}/`
- Database scripts: `/migrations/{FORM_ID}/Database/`
- .NET code: `/migrations/{FORM_ID}/DotNet/`
- React code: `/migrations/{FORM_ID}/React/`
- Tests: `/migrations/{FORM_ID}/Tests/`

## Performance Optimizations Discovered
- Load fleet/chapter lists once and cache (5-minute stale time in React Query)
- Batch updates in single database function call (save_chapter_alert_rates processes arrays)
- Indexes on foreign keys and frequently queried columns
- JSONB for complex nested data vs. multiple queries

## Testing Patterns
- Backend: xUnit + FluentAssertions + Moq for repositories
- Frontend: Jest + React Testing Library for components
- Integration: TestContainers for PostgreSQL + WebApplicationFactory
- E2E: Playwright or Cypress for full workflows

## Migration Time Estimates (for similar forms)
- Simple CRUD form (5-10 fields): 5-7 days
- Complex form with multi-line edits (like LE11): 11-15 days
- Form with extensive business rules: 15-20 days
- Add 30% for forms with unmigrated dependencies

## LE11 Migration - Completed Patterns

### Complete Code Generation Approach
- Generate ALL working code files, not just documentation
- Database: Full DDL with tables, functions, triggers, sample data
- Backend: Complete .NET solution with all layers (Controllers, Services, Repositories, Models, DTOs)
- Frontend: Complete React app with components, hooks, services, types
- Configuration: Program.cs, appsettings.json, package.json, tsconfig.json
- Documentation: Deployment guide, migration summary, README

### Expandable Table Pattern for Multi-line Cells
**Legacy:** EditableTable with XML-tagged multi-item columns
**Modern:** Material-UI collapsible rows with nested tables
```typescript
// Main row shows summary, click to expand for details
<IconButton onClick={() => setOpen(!open)}>
  {open ? <KeyboardArrowUp /> : <KeyboardArrowDown />}
</IconButton>
// Collapse component shows detailed editable grid
<Collapse in={open}>
  <Table size="small">
    {/* Detailed rows for each classification/rate */}
  </Table>
</Collapse>
```

### React Query Integration Pattern
- Use TanStack React Query for all API calls
- Custom hooks encapsulate query logic (useChapterAlertRates, useFleets, useAvailableChapters)
- 5-minute stale time for reference data (fleets, chapters)
- Automatic cache invalidation after mutations
- Loading/error states handled by query hooks

### State Management Pattern
- Custom useFormState hook for dirty tracking and updates
- Separate state for UI (selectedFleet, selectedChapter) vs. data (formData)
- Confirmation dialogs for unsaved changes
- Sequential async operations for multi-step workflows (add chapter → reload → auto-save defaults)

### Npgsql JSONB Array Parameters
```csharp
// Convert JSON strings to JSONB array for PostgreSQL
var jsonArray = data.Select(d => JsonSerializer.Serialize(d)).ToArray();
command.Parameters.Add(new NpgsqlParameter("p_json_array", NpgsqlDbType.Jsonb | NpgsqlDbType.Array) {
    Value = jsonArray.Select(j => JsonDocument.Parse(j).RootElement).ToArray()
});
```

### PostgreSQL JSONB for Multi-line Data
- Use jsonb_agg() with jsonb_build_object() to create structured arrays
- Parse JSON in .NET with System.Text.Json
- Deserialize to strongly-typed DTOs in repository layer
- Benefits: Single query vs. multiple queries, easy expansion of attributes
