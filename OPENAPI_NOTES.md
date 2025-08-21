# OpenAPI Specification Notes

This document contains implementation details, assumptions, and verification notes for the ClickHouse Backup API OpenAPI specification.

## Analysis Sources

### Primary Code Sources
- **Entry Point**: `pkg/server/server.go` (lines 207-250) - Route registration
- **HTTP Stack**: gorilla/mux router + standard Go net/http
- **Authentication**: `pkg/server/server.go` (lines 275-301) - Basic Auth middleware
- **Response Utilities**: `pkg/server/utils.go` - Error handling and JSON streaming
- **Configuration**: `pkg/config/config.go` - API configuration structure

### Route Discovery
All routes discovered from `registerHTTPHandlers()` function in `pkg/server/server.go`:

```go
// Lines 217-238: Core route registrations
r.HandleFunc("/", api.httpRootHandler).Methods("GET", "HEAD")
r.HandleFunc("/", api.httpRestartHandler).Methods("POST")
r.HandleFunc("/restart", api.httpRestartHandler).Methods("POST", "GET")
r.HandleFunc("/backup/version", api.httpVersionHandler).Methods("GET", "HEAD")
r.HandleFunc("/backup/kill", api.httpKillHandler).Methods("POST", "GET")
r.HandleFunc("/backup/watch", api.httpWatchHandler).Methods("POST", "GET")
r.HandleFunc("/backup/tables", api.httpTablesHandler).Methods("GET")
r.HandleFunc("/backup/tables/all", api.httpTablesHandler).Methods("GET")
r.HandleFunc("/backup/list", api.httpListHandler).Methods("GET", "HEAD")
r.HandleFunc("/backup/list/{where}", api.httpListHandler).Methods("GET")
r.HandleFunc("/backup/create", api.httpCreateHandler).Methods("POST")
r.HandleFunc("/backup/clean", api.httpCleanHandler).Methods("POST")
r.HandleFunc("/backup/clean/remote_broken", api.httpCleanRemoteBrokenHandler).Methods("POST")
r.HandleFunc("/backup/clean/local_broken", api.httpCleanLocalBrokenHandler).Methods("POST")
r.HandleFunc("/backup/upload/{name}", api.httpUploadHandler).Methods("POST")
r.HandleFunc("/backup/download/{name}", api.httpDownloadHandler).Methods("POST")
r.HandleFunc("/backup/restore/{name}", api.httpRestoreHandler).Methods("POST")
r.HandleFunc("/backup/delete/{where}/{name}", api.httpDeleteHandler).Methods("POST")
r.HandleFunc("/backup/status", api.httpStatusHandler).Methods("GET")
r.HandleFunc("/backup/actions", api.actionsLog).Methods("GET", "HEAD")
r.HandleFunc("/backup/actions", api.actions).Methods("POST")
```

### Authentication Mechanism
**File**: `pkg/server/server.go` (lines 275-301)
- **Type**: HTTP Basic Authentication
- **Alternative**: Query parameters `user` and `pass`
- **Middleware**: Applied to all routes except `/metrics`
- **Configuration**: Username/password from `APIConfig` structure

```go
// Basic auth extraction logic
user, pass, _ := r.BasicAuth()
query := r.URL.Query()
if u, exist := query["user"]; exist {
    user = u[0]
}
if p, exist := query["pass"]; exist {
    pass = p[0]
}
```

## Request/Response Patterns

### Error Response Format
**File**: `pkg/server/utils.go` (writeError function)
```json
{
  "status": "error",
  "operation": "operation_name",
  "error": "error_message"
}
```

### Success Response Format
**File**: `pkg/server/server.go` (httpCreateHandler lines 1024-1034)
```json
{
  "status": "acknowledged",
  "operation": "operation_type",
  "backup_name": "backup_name",
  "operation_id": "uuid"
}
```

### Status Response Format
**File**: `pkg/status/status.go` (ActionRowStatus struct lines 30-36)
```json
{
  "command": "command_string",
  "status": "in progress|success|error|cancel",
  "start": "timestamp",
  "finish": "timestamp",
  "error": "error_message"
}
```

## Parameter Analysis

### httpCreateHandler Parameters
**File**: `pkg/server/server.go` (lines 918-1035)

Discovered query parameters:
- `table` - Table pattern filter (line 944)
- `diff-from-remote` - Incremental backup base (line 948)
- `partitions` - Specific partitions (line 951)
- `schema` - Schema only backup (line 955)
- `rbac` - Include RBAC (line 959)
- `rbac-only` - RBAC only backup (line 963)
- `configs` - Include configs (line 967)
- `configs-only` - Configs only backup (line 971)
- `skip-check-parts-columns` - Skip column checks (line 977)
- `skip-projections` - Skip projections (line 982)
- `resume` - Resume operation (line 988)
- `name` - Custom backup name (line 991)
- `callback` - Callback URL (parseCallback function)

### httpUploadHandler Parameters
**File**: `pkg/server/server.go` (lines 1219-1340)

Additional parameters:
- `diff-from` - Differential upload base
- `delete-source` - Delete source files
- `resumable` - Enable resumable upload

### httpRestoreHandler Parameters
**File**: `pkg/server/server.go` (lines 1341-1543)

Additional parameters:
- `data` - Restore data only
- `restore-database-mapping` - Database mapping
- `restore-table-mapping` - Table mapping
- `drop` - Drop existing tables
- `ignore-dependencies` - Ignore dependencies

## Data Structures

### Backup List Structure
**File**: `pkg/server/server.go` (lines 809-822)
```go
type backupJSON struct {
    Name           string `json:"name"`
    Created        string `json:"created"`
    Size           uint64 `json:"size,omitempty"`
    DataSize       uint64 `json:"data_size,omitempty"`
    ObjectDiskSize uint64 `json:"object_disk_size,omitempty"`
    MetadataSize   uint64 `json:"metadata_size"`
    RBACSize       uint64 `json:"rbac_size,omitempty"`
    ConfigSize     uint64 `json:"config_size,omitempty"`
    CompressedSize uint64 `json:"compressed_size,omitempty"`
    Location       string `json:"location"`
    RequiredBackup string `json:"required"`
    Desc           string `json:"desc"`
}
```

### Callback Response Structure
**File**: `pkg/server/utils.go` (lines 71-75)
```go
type CallbackResponse struct {
    Status      string `json:"status"`
    Error       string `json:"error"`
    OperationId string `json:"operation_id"`
}
```

## Test Evidence

### Integration Tests
**File**: `test/integration/integration_test.go`

Key test functions providing API usage examples:
- `testAPIBackupCreate` (lines 1520-1530) - POST /backup/create with parameters
- `testAPIBackupList` (lines 1531-1550) - GET /backup/list response format
- `testAPIBackupStatus` (lines 1490-1505) - GET /backup/status response
- `testAPIRestart` (lines 1194-1200) - POST /restart

Example curl commands from tests:
```bash
curl -sfL -XPOST "http://localhost:7171/backup/create?table=long_schema.*&name=z_backup_$i"
curl -sfL 'http://localhost:7171/backup/list'
curl -sL "http://localhost:7171/backup/status"
```

## Assumptions and Implementation Notes

### 1. Path Parameters
- **gorilla/mux format**: `{name}` and `{where}` converted to OpenAPI path parameters
- **Validation**: `where` parameter constrained to enum `[local, remote]` based on handler logic

### 2. Query Parameter Alternatives
Many parameters support both hyphenated and underscore variants:
- `diff-from-remote` / `diff_from_remote`
- `rbac-only` / `rbac_only`
- `configs-only` / `configs_only`
- `skip-check-parts-columns` / `skip_check_parts_columns`

### 3. Response Formats
- **JSON Streaming**: Uses chunked transfer encoding for large responses
- **Content-Type**: `application/json; charset=UTF-8`
- **Cache Headers**: `no-store, no-cache, must-revalidate`

### 4. Async Operations
Most backup operations are asynchronous:
- Immediate response with `operation_id`
- Status trackable via `/backup/status`
- Optional callback notifications

### 5. Security
- **Basic Auth**: Primary authentication method
- **Query Auth**: Alternative via `user`/`pass` parameters
- **HTTPS Support**: Configurable via API settings
- **Metrics Endpoint**: No authentication required

## Areas Requiring Verification

### 1. Exact Response Schemas
Some response structures inferred from handler logic may need validation:
- **TODO**: Verify actual JSON structure from live API responses
- **TODO**: Confirm all optional fields and their conditions

### 2. Parameter Validation Rules
While basic types are documented, specific validation rules need verification:
- **TODO**: Confirm pattern validation for table names
- **TODO**: Verify partition format requirements
- **TODO**: Check database/table mapping syntax

### 3. Status Code Coverage
Standard HTTP status codes documented, but edge cases may exist:
- **TODO**: Verify all possible error conditions and their status codes
- **TODO**: Confirm 423 Locked usage for parallel operation prevention

### 4. Callback Payload Format
Callback mechanism documented but payload structure needs confirmation:
- **TODO**: Verify exact JSON payload sent to callback URLs
- **TODO**: Confirm callback timing (on completion, on error, etc.)

## File References

- `pkg/server/server.go`: Main HTTP handler implementations
- `pkg/server/utils.go`: Response utilities and error handling
- `pkg/server/callback.go`: Callback mechanism implementation
- `pkg/config/config.go`: Configuration structures
- `pkg/status/status.go`: Status tracking and response structures
- `test/integration/integration_test.go`: API usage examples and test cases
- `ReadMe.md`: API documentation with usage examples

## OpenAPI Specification Validation

The generated OpenAPI 3.0.3 specification follows best practices:
- ✅ Consistent error response schemas across all endpoints
- ✅ Reusable components for common structures
- ✅ Proper parameter validation and documentation
- ✅ Security schemes defined for Basic Auth
- ✅ Examples provided for key operations
- ✅ Tags used for logical grouping of endpoints
- ✅ All discovered endpoints documented with accurate HTTP methods