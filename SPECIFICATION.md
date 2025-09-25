# Super Data MCP - Implementation Guide for Claude Code

## Project Overview

You are building Super Data MCP, a Model Context Protocol server that provides LLMs with sophisticated data querying capabilities. This server is designed for team collaboration with centralized SQLite storage, S3 synchronization, and support for complex jq functions. It will primarily be used by teams accessing a shared server via http-stream.

## Critical Architecture Decisions

### Primary Storage: SQLite
- ALL tools, functions, and metadata stored in SQLite database
- YAML files used only as seed data on initial setup
- Enables instant team-wide updates without file deployments
- Complete audit trail and version history

### Team Collaboration Focus
- Multi-user access via http-stream transport
- Centralized state management with S3 sync
- Approval workflows for learned tools
- Shared jq function library with dependency management

## Implementation Priority Order

### Phase 1: Core Foundation with SQLite (Build First)
1. **Project Setup**
   - Initialize Python 3.12 project with uv
   - Create directory structure as specified
   - Set up pyproject.toml with dependencies
   - Create basic logging utilities

2. **Database Layer** (`src/database/`)
   - `models.py`: SQLAlchemy models for tools, functions, users
   - `connection.py`: Connection pooling and management
   - `migrations.py`: Database schema migrations
   - Initialize SQLite with WAL mode for concurrency

3. **Core MCP Server** (`src/core/server.py`)
   - Implement MCP server using official Python SDK
   - Start with HTTP transport (primary use case)
   - Add stdio support for local development
   - User identification from auth headers

### Phase 2: Function & Tool Management
1. **jq Function Library** (`src/functions/`)
   - `library.py`: Manage shared jq functions
   - `resolver.py`: Dependency resolution for functions
   - `compiler.py`: Compile functions with tools
   - Import functions from YAML seed files

2. **Tool System** (`src/tools/`)
   - `manager.py`: CRUD operations for tools
   - `executor.py`: Execute tools with function resolution
   - `importer.py`: Import tools from YAML to SQLite
   - Track tool source (yaml/learned/manual)

3. **Query Engine** (`src/query/engines/jq.py`)
   - Execute compiled jq programs (functions + query)
   - Handle complex jq syntax including def statements
   - Timeout and resource management

### Phase 3: Team Collaboration Features
1. **Sync System** (`src/sync/`)
   - `s3_sync.py`: Bi-directional S3 synchronization
   - `lock.py`: Distributed locking mechanism
   - `merge.py`: Conflict resolution strategies
   - Implement optimistic locking with retry

2. **Approval Workflow** (`src/learning/approval.py`)
   - Proposal submission with justification
   - Team voting system
   - Auto-approval thresholds
   - Notification system for team updates

3. **User Management** (`src/auth/`)
   - User identification from headers
   - Permission levels (user/admin/owner)
   - Activity tracking per user

### Phase 4: Data Layer
1. **Filesystem Provider** (`src/data/providers/filesystem.py`)
   - Load JSON datasets from local/S3
   - Handle large files with streaming
   - Change detection for auto-reload

2. **Schema Management** (`src/data/schemas/`)
   - Auto-generate and store schemas in SQLite
   - Version tracking with migration paths
   - Compatibility matrix in database

### Phase 5: Learning System
1. **Tool Learning** (`src/learning/`)
   - `proposer.py`: LLM proposes new tools
   - `validator.py`: Test proposed tools
   - `optimizer.py`: Suggest improvements
   - Store all learned tools in SQLite

2. **Performance Tracking** (`src/learning/metrics.py`)
   - Track usage statistics
   - Measure execution times
   - Identify optimization opportunities

### Phase 6: Advanced Features
1. **WebSocket Support** for real-time updates
2. **Query Optimization** with execution plans
3. **Caching Layer** with Redis support
4. **Export/Import** tools for sharing

## Key Implementation Details

### Directory Structure Creation
```
super-data/
├── src/
│   ├── core/
│   ├── database/         # New: Central database layer
│   │   ├── models.py
│   │   ├── connection.py
│   │   └── migrations.py
│   ├── data/
│   │   ├── providers/
│   │   └── schemas/
│   ├── query/
│   │   └── engines/
│   ├── functions/        # New: jq function management
│   │   ├── library.py
│   │   ├── resolver.py
│   │   └── compiler.py
│   ├── sync/            # New: S3 synchronization
│   │   ├── s3_sync.py
│   │   ├── lock.py
│   │   └── merge.py
│   ├── learning/
│   │   ├── proposer.py
│   │   ├── validator.py
│   │   └── approval.py
│   ├── auth/            # New: User management
│   │   └── identity.py
│   ├── tools/
│   └── utils/
├── database/
│   ├── tools.db         # Primary SQLite database
│   └── .version
├── datasets/
│   └── json/
│       ├── functions.yaml  # Shared function definitions
│       └── tools.yaml      # Seed tool definitions
├── tests/
└── config/
```

### Database Schema (SQLite)
```sql
-- Core tables for team collaboration
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    username TEXT UNIQUE NOT NULL,
    email TEXT,
    role TEXT DEFAULT 'user', -- user/admin/owner
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_active TIMESTAMP
);

CREATE TABLE jq_functions (
    id TEXT PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    definition TEXT NOT NULL,
    description TEXT,
    parameters_json TEXT,  -- Expected parameters
    created_by TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_by TEXT,
    updated_at TIMESTAMP,
    version INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    datasets_json TEXT,
    FOREIGN KEY (created_by) REFERENCES users(id)
);

CREATE TABLE tools (
    id TEXT PRIMARY KEY,
    name TEXT UNIQUE NOT NULL,
    description TEXT,
    query_template TEXT NOT NULL,
    required_functions_json TEXT,  -- List of jq_function names
    parameters_json TEXT NOT NULL,
    datasets_json TEXT NOT NULL,
    created_by TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    approval_status TEXT DEFAULT 'approved',
    approved_by TEXT,
    approved_at TIMESTAMP,
    usage_count INTEGER DEFAULT 0,
    avg_execution_time_ms INTEGER,
    source TEXT DEFAULT 'yaml',  -- yaml/learned/manual
    original_yaml_path TEXT,
    version INTEGER DEFAULT 1,
    FOREIGN KEY (created_by) REFERENCES users(id),
    FOREIGN KEY (approved_by) REFERENCES users(id)
);

CREATE TABLE function_dependencies (
    function_id TEXT NOT NULL,
    depends_on_function_id TEXT NOT NULL,
    PRIMARY KEY (function_id, depends_on_function_id),
    FOREIGN KEY (function_id) REFERENCES jq_functions(id),
    FOREIGN KEY (depends_on_function_id) REFERENCES jq_functions(id)
);

CREATE TABLE tool_execution_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    tool_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    executed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    parameters_json TEXT,
    execution_time_ms INTEGER,
    success BOOLEAN,
    error_message TEXT,
    result_size_bytes INTEGER,
    FOREIGN KEY (tool_id) REFERENCES tools(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE approval_votes (
    tool_id TEXT NOT NULL,
    user_id TEXT NOT NULL,
    vote TEXT NOT NULL, -- approve/reject
    comment TEXT,
    voted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (tool_id, user_id),
    FOREIGN KEY (tool_id) REFERENCES tools(id),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Indexes for performance
CREATE INDEX idx_tools_status ON tools(approval_status);
CREATE INDEX idx_tools_source ON tools(source);
CREATE INDEX idx_execution_log_tool ON tool_execution_log(tool_id);
CREATE INDEX idx_execution_log_user ON tool_execution_log(user_id);
```

### Dependencies (pyproject.toml)
```toml
[project]
name = "super-data"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "mcp>=0.1.0",           # Official MCP SDK
    "sqlalchemy>=2.0",      # Database ORM
    "alembic>=1.13",        # Database migrations
    "pyyaml>=6.0",
    "jsonschema>=4.0",
    "aiofiles>=24.0",
    "aiocache>=0.12",
    "boto3>=1.34",
    "aioboto3>=13.0",
    "fastapi>=0.110.0",     # For HTTP transport
    "uvicorn>=0.29.0",      # ASGI server
    "websockets>=12.0",     # WebSocket support
    "aiosqlite>=0.20.0",    # Async SQLite
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "black>=24.0",
    "ruff>=0.4",
    "mypy>=1.10",
    "faker>=24.0",  # Test data generation
]
```

### Environment Variables (.env.example)
```bash
# Core Configuration
SUPER_DATA_DATASETS_PATH=./datasets
SUPER_DATA_DATABASE_PATH=./database/tools.db
SUPER_DATA_LOG_LEVEL=INFO

# Transport Configuration
SUPER_DATA_TRANSPORT=http  # http/stdio
SUPER_DATA_HOST=0.0.0.0
SUPER_DATA_PORT=8080

# S3 Configuration (required for team sync)
SUPER_DATA_S3_BUCKET=team-data-bucket
SUPER_DATA_S3_DB_KEY=databases/tools.db
SUPER_DATA_SYNC_INTERVAL=30  # seconds
AWS_REGION=us-east-1

# Team Configuration
SUPER_DATA_APPROVAL_THRESHOLD=2  # votes needed
SUPER_DATA_AUTO_APPROVE_TIMEOUT=24  # hours
SUPER_DATA_ENABLE_LEARNING=true

# Performance
SUPER_DATA_QUERY_TIMEOUT_SECONDS=30
SUPER_DATA_MAX_RESULT_SIZE_MB=100
SUPER_DATA_CACHE_SIZE_MB=500
```

## Code Patterns to Follow

### Database Access Pattern
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from contextlib import asynccontextmanager

class DatabaseManager:
    def __init__(self, db_path: str):
        self.engine = create_async_engine(
            f"sqlite+aiosqlite:///{db_path}",
            echo=False,
            connect_args={
                "check_same_thread": False,
                "isolation_level": None,  # Use WAL mode
            }
        )

    @asynccontextmanager
    async def session(self):
        async with AsyncSession(self.engine) as session:
            yield session
```

### User Identification Pattern
```python
from fastapi import Header, HTTPException

async def get_current_user(
    x_user_id: str = Header(None),
    authorization: str = Header(None)
) -> str:
    # Extract user from headers for HTTP transport
    if x_user_id:
        return x_user_id
    # Fallback to token-based identification
    if authorization:
        return extract_user_from_token(authorization)
    raise HTTPException(401, "User identification required")
```

### Function Compilation Pattern
```python
class FunctionCompiler:
    async def compile_tool_query(self, tool_id: str) -> str:
        """Compile tool with all required jq functions."""
        tool = await self.db.get_tool(tool_id)
        functions = await self.db.get_tool_functions(tool_id)

        # Build complete jq program
        program_parts = []

        # Add function definitions in dependency order
        sorted_functions = self.topological_sort(functions)
        for func in sorted_functions:
            program_parts.append(func.definition)

        # Add the main query
        program_parts.append(tool.query_template)

        return "\n".join(program_parts)
```

### S3 Sync Pattern
```python
class S3Synchronizer:
    async def sync_with_lock(self):
        """Sync database with S3 using distributed locking."""
        lock_acquired = False
        try:
            # Acquire distributed lock
            lock_acquired = await self.acquire_lock()
            if not lock_acquired:
                # Another instance is syncing
                await asyncio.sleep(1)
                return await self.download_latest()

            # Pull latest from S3
            await self.download_latest()

            # Apply local changes
            await self.apply_local_changes()

            # Push merged database
            await self.upload_database()

        finally:
            if lock_acquired:
                await self.release_lock()
```

### Tool Proposal Pattern
```python
class ToolProposer:
    async def propose_tool(
        self,
        user_id: str,
        name: str,
        query: str,
        parameters: List[Dict],
        justification: str
    ) -> str:
        """Propose a new tool for team approval."""
        # Validate query syntax
        await self.validate_jq_syntax(query)

        # Extract required functions
        required_functions = self.extract_function_calls(query)

        # Create proposal
        tool_id = await self.db.create_tool(
            name=name,
            query_template=query,
            parameters_json=json.dumps(parameters),
            required_functions_json=json.dumps(required_functions),
            created_by=user_id,
            approval_status='pending',
            source='learned'
        )

        # Notify team for approval
        await self.notify_team(tool_id, justification)

        return tool_id
```

## Testing Strategy

### Test Structure
```
tests/
├── unit/
│   ├── test_function_compiler.py
│   ├── test_query_engine.py
│   └── test_approval_workflow.py
├── integration/
│   ├── test_s3_sync.py
│   ├── test_team_collaboration.py
│   └── test_database_operations.py
├── fixtures/
│   ├── sample_functions.yaml
│   ├── sample_tools.yaml
│   └── test_data.json
└── conftest.py
```

### Critical Test Cases
1. **Function dependency resolution**
2. **Concurrent database access**
3. **S3 sync conflict resolution**
4. **Tool approval workflow**
5. **Complex jq function compilation**
6. **User permission enforcement**
7. **Query timeout handling**
8. **Large result streaming**
9. **Database migration rollback**
10. **Lock timeout scenarios**

## Dataset Examples

### datasets/json/functions.yaml
```yaml
# Shared jq function library
functions:
  - name: calculate_efficiency
    description: "Calculate efficiency metrics"
    definition: |
      def calculate_efficiency($revenue; $cost):
        if $cost > 0 then
          (($revenue - $cost) / $cost * 100) | floor
        else 0 end;
    parameters:
      - name: revenue
        type: number
      - name: cost
        type: number

  - name: analyze_trends
    description: "Analyze data trends over time"
    definition: |
      def analyze_trends($data; $window):
        $data | length as $len |
        if $len >= $window then
          [range(0; $len - $window + 1) as $i |
           $data[$i:$i+$window] | add / $window]
        else [] end;
    parameters:
      - name: data
        type: array
      - name: window
        type: integer

  - name: group_and_aggregate
    description: "Group by field and aggregate"
    definition: |
      def group_and_aggregate($group_field; $agg_field; $agg_type):
        group_by(.[$group_field]) |
        map({
          ($group_field): .[0][$group_field],
          value: (
            if $agg_type == "sum" then [.[][$agg_field]] | add
            elif $agg_type == "avg" then ([.[][$agg_field]] | add / length)
            elif $agg_type == "max" then [.[][$agg_field]] | max
            elif $agg_type == "min" then [.[][$agg_field]] | min
            else [.[][$agg_field]] | length end
          )
        });
    depends_on: []  # No dependencies
```

### datasets/json/tools.yaml
```yaml
# Seed tools that use shared functions
tools:
  - name: analyze_performance
    description: "Analyze dataset performance with efficiency calculation"
    query: |
      # Use shared function
      calculate_efficiency(.revenue; .cost) as $efficiency |
      {
        dataset: "${dataset_name}",
        efficiency: $efficiency,
        profitable: ($efficiency > 0)
      }
    required_functions:
      - calculate_efficiency
    parameters:
      - name: dataset_name
        type: string
        required: true
    datasets: ["sales", "products"]

  - name: trend_analysis
    description: "Perform trend analysis on time series data"
    query: |
      .data |
      analyze_trends(.; ${window_size}) as $trends |
      {
        original_data: .,
        trends: $trends,
        direction: (
          if $trends[-1] > $trends[0] then "increasing"
          elif $trends[-1] < $trends[0] then "decreasing"
          else "stable" end
        )
      }
    required_functions:
      - analyze_trends
    parameters:
      - name: window_size
        type: integer
        default: 7
    datasets: ["metrics", "timeseries"]
```

### Initial Database Seed Script
```python
# src/database/seed.py
async def seed_database():
    """Import YAML seeds into SQLite on first run."""

    # Check if already seeded
    if await db.get_tool_count() > 0:
        return

    # Import functions
    functions_yaml = load_yaml("datasets/json/functions.yaml")
    for func in functions_yaml["functions"]:
        await db.create_function(
            name=func["name"],
            definition=func["definition"],
            description=func.get("description"),
            created_by="system",
            source="yaml"
        )

    # Import tools
    tools_yaml = load_yaml("datasets/json/tools.yaml")
    for tool in tools_yaml["tools"]:
        await db.create_tool(
            name=tool["name"],
            query_template=tool["query"],
            required_functions_json=json.dumps(
                tool.get("required_functions", [])
            ),
            parameters_json=json.dumps(tool["parameters"]),
            datasets_json=json.dumps(tool["datasets"]),
            created_by="system",
            source="yaml",
            approval_status="approved"
        )
```

## Important Design Decisions

1. **SQLite as Single Source of Truth**: All tools, functions, and metadata live in SQLite, YAML files are only seed data

2. **Team-First Design**: Every feature considers multi-user concurrent access and team collaboration

3. **Function Compilation**: Always compile jq functions with tools into complete programs before execution

4. **Optimistic Locking**: Use version numbers and retry logic for concurrent modifications

5. **Approval by Default**: YAML-imported tools are pre-approved, learned tools require approval

6. **Streaming First**: Process large datasets as streams, never load entire datasets unless explicitly cached

7. **User Attribution**: Track who created, modified, and approved every tool and function

8. **Fail Fast**: Validate queries, functions, and parameters before execution

## Common Pitfalls to Avoid

1. **Don't use file-based tools in production** - SQLite is the source of truth
2. **Don't ignore locking** - Always use distributed locks for S3 sync
3. **Don't trust user input** - Validate and sanitize all jq queries
4. **Don't skip function resolution** - Always resolve dependencies before execution
5. **Don't forget WAL mode** - Enable for concurrent SQLite access
6. **Don't hardcode user IDs** - Extract from request context
7. **Don't block on approvals** - Use async notifications

## Success Criteria

Your implementation is successful when:
1. **Multiple users can propose and use tools simultaneously**
2. **Functions are properly compiled with tools**
3. **S3 sync maintains consistency across instances**
4. **Complex jq functions (with def statements) execute correctly**
5. **Approval workflow notifies team and tracks votes**
6. **Database handles 100+ concurrent connections**
7. **Tool execution is tracked per user**
8. **HTTP transport supports team authentication**

## Critical Implementation Order

1. **Start with SQLite setup** - This is the foundation
2. **Implement function compiler** - Core to query execution
3. **Add HTTP transport early** - Primary use case
4. **Test S3 sync thoroughly** - Critical for team consistency
5. **Add approval workflow** - Essential for team trust

## S3 Sync Architecture

```
S3 Bucket Structure:
s3://bucket/
├── databases/
│   ├── tools.db           # Current database
│   ├── tools.db.lock      # Lock file (contains owner ID and timestamp)
│   ├── tools.db.version   # Version for quick checks
│   └── backups/
│       └── tools.db.YYYYMMDD-HHMMSS  # Automatic backups
├── exports/
│   ├── approved_tools.json   # For sharing across teams
│   └── functions.json         # Shareable function library
```

### Lock Mechanism
```python
class DistributedLock:
    async def acquire(self, timeout=30):
        lock_content = {
            "owner": self.instance_id,
            "acquired_at": datetime.now().isoformat(),
            "expires_at": (datetime.now() + timedelta(seconds=timeout)).isoformat()
        }
        # Use S3 conditional puts to ensure atomicity
```

## Additional Notes

- **Start with HTTP transport** since that's the primary use case
- **Test with multiple concurrent users** from day one
- **Use SQLAlchemy for database operations** to ensure proper migrations
- **Implement health checks** for monitoring database and S3 sync status
- **Add metrics collection** for tool usage and performance
- **Document the approval workflow** clearly for team adoption
- **Consider rate limiting** per user to prevent abuse
- **Implement soft deletes** for tools (mark inactive vs hard delete)
