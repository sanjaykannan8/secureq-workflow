# SecureQ Module Documentation

## Overview

SecureQ is a security-focused PostgreSQL database query handler designed to provide secure, role-based database access with advanced query validation and header management capabilities. It implements multiple layers of security to prevent SQL injection and unauthorized access.

## Key Features

### 🔐 Security Features
- **SQL Fingerprinting**: Advanced query structure validation using AST parsing
- **Role-Based Access Control**: Granular permission management with restricted views
- **Header Management**: Time-based session tokens with automatic cleanup
- **Query Validation**: Structure-based query comparison to prevent injection attacks

### 🏗️ Architecture Components
- **Database Connection Management**: Secure PostgreSQL connection handling
- **Header Lifecycle Management**: Automatic token generation and cleanup
- **Role Creation & Permission System**: Dynamic role creation with column-level restrictions
- **Query Fingerprinting Engine**: SQL structure analysis using sqlglot

## Class: SecureQ

### Constructor
```python
SecureQ(user, password, database, host=None, port=None)
```

**Parameters:**
- `user`: Database username for administrative operations
- `password`: Database password
- `database`: Target database name
- `host`: Database host (default: 'localhost')
- `port`: Database port (default: 5432)

**Initialization Process:**
1. Establishes PostgreSQL connection
2. Creates header_manager table if not exists
3. Starts background cleanup thread for expired headers

### Methods

#### Header Management

##### `create_header()` → str
**Purpose**: Generates unique, time-stamped session headers

**Process:**
1. Generates random 6-character alphanumeric string
2. Creates header format: `SQ-{random}`
3. Ensures uniqueness by checking against existing headers
4. Stores header with current timestamp in database
5. Returns the generated header

**Use Case**: Session management and request validation

##### `cleanup_headers()` → None
**Purpose**: Removes expired headers (older than 10 minutes)

**Process:**
1. Calculates current timestamp
2. Deletes headers older than 600 seconds (10 minutes)
3. Commits transaction to database

##### `start_cleanup_thread()` → None
**Purpose**: Initiates background thread for automatic header cleanup

**Process:**
1. Creates daemon thread running cleanup every 10 minutes
2. Ensures expired headers are automatically removed
3. Prevents memory bloat and maintains security

#### Role & Permission Management

##### `Crole(role, table, column, value)` → bool
**Purpose**: Creates restricted database roles with column-level permissions

**Parameters:**
- `role`: Name of the role to create
- `table`: Target table for access control
- `column`: List of columns to RESTRICT (exclude from access)
- `value`: Password for the new role

**Process:**
1. **Role Existence Check**: Verifies if role already exists
2. **Column Analysis**: 
   - Retrieves all columns from target table
   - Calculates allowed columns (excludes restricted ones)
3. **Role Creation**:
   - Creates PostgreSQL role with LOGIN capability
   - Sets provided password
4. **View Creation**:
   - Creates restricted view (`pview`) with only allowed columns
   - Excludes sensitive columns (e.g., passwords)
5. **Permission Management**:
   - Revokes all direct table access
   - Grants SELECT permission only on restricted view

**Security Benefits:**
- **Column-Level Security**: Prevents access to sensitive fields
- **Read-Only Access**: No INSERT/UPDATE/DELETE permissions
- **View-Based Isolation**: Users see only filtered data

#### Query Security & Validation

##### `create_fingerprint(query)` → str
**Purpose**: Generates structural fingerprint of SQL queries using AST parsing

**Fingerprint Character Mapping:**
- `k`: Keywords/Commands (SELECT, INSERT, WHERE, JOIN, etc.)
- `l`: Literals/Values (strings, numbers, NULL, *)
- `t`: Table references
- `c`: Column references  
- `i`: Identifiers (aliases, column names in INSERT)
- `o`: Operators (AND, OR, =, >, <, etc.)
- `f`: Functions (COUNT, SUM, AVG, etc.)
- `s`: Structural elements (parentheses)
- `u`: UPDATE-specific elements (SET clause)

**Process:**
1. **SQL Parsing**: Uses sqlglot to parse query into AST
2. **Node Traversal**: Recursively visits all AST nodes
3. **Type Classification**: Maps each node to fingerprint character
4. **Fingerprint Generation**: Concatenates characters in traversal order

**Example:**
```sql
SELECT name FROM users WHERE id = 123
```
Fingerprint: `kctcolkl` (k=SELECT, c=name, t=users, c=column, o=WHERE, l=id, k=clause, l=123)

##### `s_query(d_query, o_query)` → bool
**Purpose**: Secure query execution with structure validation

**Parameters:**
- `d_query`: Template/reference query (defines allowed structure)
- `o_query`: Actual query to execute (must match structure)

**Security Process:**
1. **Fingerprint Generation**:
   - Creates fingerprint for template query (d_query)
   - Creates fingerprint for actual query (o_query)
2. **Structure Comparison**:
   - Compares both fingerprints for exact match
   - Rejects if structures differ (potential injection)
3. **Secure Execution**:
   - If fingerprints match: executes query using restricted role
   - If fingerprints differ: rejects query and logs security violation

**Security Benefits:**
- **Injection Prevention**: Blocks queries with different structure
- **Role Isolation**: Executes with minimal privileges
- **Structure Validation**: Ensures query follows expected pattern

## Security Architecture

### Multi-Layer Security Model

1. **Connection Layer**: Secure PostgreSQL connection with authentication
2. **Session Layer**: Time-based header management with automatic expiration
3. **Authorization Layer**: Role-based access with column restrictions
4. **Query Layer**: Structure validation using SQL fingerprinting
5. **Execution Layer**: Isolated execution with minimal privileges

### Security Mechanisms

#### SQL Injection Prevention
- **Structural Analysis**: Fingerprinting detects query structure changes
- **Parameterized Queries**: Uses proper parameter binding
- **Template Matching**: Only allows queries matching predefined structures

#### Access Control
- **Role Isolation**: Each operation uses purpose-specific roles
- **Column Restriction**: Sensitive data excluded from accessible views
- **Read-Only Access**: Prevents data modification through restricted roles

#### Session Security
- **Time-Based Expiration**: Headers automatically expire after 10 minutes
- **Unique Tokens**: Each session gets cryptographically random identifier
- **Background Cleanup**: Automatic removal of expired sessions

## Usage Examples

### Basic Setup
```python
# Initialize SecureQ instance
sq = SecureQ(
    user='admin_user',
    password='admin_pass',
    database='myapp_db',
    host='localhost',
    port=5432
)
```

### Role Creation
```python
# Create role that can access all columns except 'password' and 'secret_key'
success = sq.Crole(
    role='app_reader',
    table='users',
    column=['password', 'secret_key'],  # Columns to RESTRICT
    value='reader_password'
)
```

### Secure Query Execution
```python
# Define template query structure
template = "SELECT name, email FROM users WHERE id = 1"

# Execute actual query (must match template structure)
actual = "SELECT name, email FROM users WHERE id = 42"

# Execute securely
result = sq.s_query(template, actual)  # Returns True if successful
```

### Header Management
```python
# Generate session header
session_header = sq.create_header()  # Returns: "SQ-Xx7nM2"

# Headers automatically expire and cleanup after 10 minutes
```

## Best Practices

### Security
1. **Template Validation**: Always validate query templates before use
2. **Role Minimization**: Create roles with minimum necessary permissions
3. **Regular Cleanup**: Monitor header cleanup logs for security events
4. **Error Handling**: Implement proper error handling for security failures

### Performance
1. **Fingerprint Caching**: Cache fingerprints for frequently used queries
2. **Background Cleanup**: Monitor cleanup thread performance

### Monitoring
1. **Query Logs**: Log all fingerprint mismatches for security analysis
2. **Role Usage**: Monitor role access patterns
3. **Header Lifecycle**: Track header generation and cleanup metrics

## Integration Patterns

### Web Application Integration
```python
# In web framework (Flask/Django/FastAPI)
class DatabaseMiddleware:
    def __init__(self):
        self.sq = SecureQ(...)
    
    def execute_user_query(self, template, user_input):
        return self.sq.s_query(template, user_input)
```

### API Security
```python
# API endpoint with header validation
@app.route('/api/data')
def get_data():
    header = sq.create_header()
    # Use header for request validation
    return {"data": result, "session": header}
```

## Error Handling

### Common Error Scenarios
- **Role Creation Failure**: Role already exists or insufficient permissions
- **Fingerprint Mismatch**: Query structure doesn't match template
- **Connection Failure**: Database connection issues
- **Permission Denied**: Role lacks necessary permissions

### Error Response Patterns
```python
try:
    result = sq.s_query(template, query)
    if not result:
        # Handle security violation
        log_security_event("Query structure mismatch")
except Exception as e:
    # Handle database errors
    log_error(f"Database error: {e}")
```
