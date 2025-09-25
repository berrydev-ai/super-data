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
   - Verify jq binary installation and add PATH detection

2. **Database Layer** (`src/database/`)
   - `models.py`: SQLAlchemy models for tools, functions, users
   - `connection.py`: Connection pooling and management
   - `migrations.py`: Database schema migrations with version checks
   - `version.py`: Database version management for team sync
   - Initialize SQLite with WAL mode for concurrency

3. **Basic Authentication** (`src/auth/identity.py`)
   - User identification from HTTP headers (X-User-ID, Authorization)
   - Simple header-based auth for team identification
   - User creation/lookup in database

4. **Query Engine** (`src/super_data/query/engines/jq.py`)
   - Execute compiled jq programs via subprocess calls to jq binary
   - Input sanitization and timeout management
   - Stream handling for large results

5. **Core MCP Server** (`src/core/server.py`)
   - Implement MCP server using official Python SDK
   - Start with HTTP transport (primary use case)
   - Add stdio support for local development
   - Integration with auth and query engine

### Phase 2: Function & Tool Management
1. **jq Function Library** (`src/super_data/functions/`)
   - `library.py`: Manage shared jq functions
   - `resolver.py`: Dependency resolution for functions
   - `compiler.py`: Compile functions with tools
   - Import functions from YAML seed files

2. **Tool System** (`src/super_data/tools/`)
   - `manager.py`: CRUD operations for tools
   - `executor.py`: Execute tools with function resolution
   - `importer.py`: Import tools from YAML to SQLite
   - Track tool source (yaml/learned/manual)

### Phase 3: Team Collaboration Features
1. **Sync System** (`src/super_data/sync/`)
   - `s3_sync.py`: Bi-directional S3 synchronization
   - `lock.py`: Distributed locking mechanism
   - `merge.py`: Conflict resolution strategies
   - Implement optimistic locking with retry

2. **Approval Workflow** (`src/super_data/learning/approval.py`)
   - Proposal submission with justification
   - Team voting system
   - Auto-approval thresholds
   - Notification system for team updates

3. **Advanced User Management** (`src/super_data/auth/`)
   - Permission levels (user/admin/owner)
   - Activity tracking per user
   - JWT token validation (if needed beyond headers)

### Phase 4: Data Layer
1. **Filesystem Provider** (`src/super_data/data/providers/filesystem.py`)
   - Load JSON datasets from local/S3
   - Handle large files with streaming
   - Change detection for auto-reload

2. **Schema Management** (`src/super_data/data/schemas/`)
   - Auto-generate and store schemas in SQLite
   - Version tracking with migration paths
   - Compatibility matrix in database

### Phase 5: Learning System
1. **Tool Learning** (`src/super_data/learning/`)
   - `proposer.py`: LLM proposes new tools
   - `validator.py`: Test proposed tools
   - `optimizer.py`: Suggest improvements
   - Store all learned tools in SQLite

2. **Performance Tracking** (`src/super_data/learning/metrics.py`)
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
│   └── super_data/
│       ├── core/
│       ├── database/
│       │   ├── models.py
│       │   ├── connection.py
│       │   └── migrations.py
│       ├── data/
│       │   ├── providers/
│       │   └── schemas/
│       ├── query/
│       │   └── engines/
│       ├── functions/
│       │   ├── library.py
│       │   ├── resolver.py
│       │   └── compiler.py
│       ├── sync/
│       │   ├── s3_sync.py
│       │   ├── lock.py
│       │   └── merge.py
│       ├── learning/
│       │   ├── proposer.py
│       │   ├── validator.py
│       │   └── approval.py
│       ├── auth/
│       │   └── identity.py
│       ├── tools/
│       └── utils/
├── database/
│   ├── tools.db
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

# External system requirements:
# - jq binary must be installed and available in PATH
# - For Ubuntu/Debian: apt-get install jq
# - For macOS: brew install jq
# - For Windows: choco install jq or download from https://jqlang.github.io/jq/

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-anyio>=0.1.0",    # For anyio testing as per CLAUDE.md
    "black>=24.0",
    "ruff>=0.4",
    "mypy>=1.10",
    "faker>=24.0",            # Test data generation
    "moto[s3]>=5.0",          # S3 mocking for tests
    "httpx>=0.27.0",          # HTTP client for testing
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
SUPER_DATA_S3_LOCK_TIMEOUT_SECONDS=45  # Query timeout + 15s buffer
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
import logging

logger = logging.getLogger(__name__)

async def get_current_user(
    x_user_id: str = Header(None),
    authorization: str = Header(None)
) -> str:
    """Extract user identity from request headers.

    Priority:
    1. X-User-ID header (team environments)
    2. Authorization header with Bearer token
    3. Fallback for stdio transport (system user)
    """
    # Direct user ID (preferred for team setups)
    if x_user_id:
        logger.debug(f"Identified user from X-User-ID header: {x_user_id}")
        return x_user_id

    # Extract from Authorization header
    if authorization and authorization.startswith("Bearer "):
        token = authorization[7:]  # Remove "Bearer " prefix
        try:
            user_id = extract_user_from_token(token)
            logger.debug(f"Identified user from token: {user_id}")
            return user_id
        except Exception:
            logger.exception("Failed to extract user from token")

    # For stdio transport, use system user
    if not authorization and not x_user_id:
        import getpass
        system_user = getpass.getuser()
        logger.debug(f"Using system user for stdio transport: {system_user}")
        return f"system:{system_user}"

    raise HTTPException(401, "User identification required")

def extract_user_from_token(token: str) -> str:
    """Extract user ID from JWT or API key token.

    Implementation depends on your auth strategy:
    - JWT: decode and extract 'sub' claim
    - API key: lookup in database/config
    """
    # For now, assume simple API key format: "user_{id}"
    if token.startswith("user_"):
        return token

    # TODO: Add JWT decoding logic if needed
    # import jwt
    # payload = jwt.decode(token, verify=False)  # Add proper verification
    # return payload.get('sub', 'unknown')

    raise ValueError("Invalid token format")
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
        lock_timeout = int(os.getenv("SUPER_DATA_S3_LOCK_TIMEOUT_SECONDS", "45"))
        lock_acquired = False
        try:
            # Acquire distributed lock with timeout
            lock_acquired = await self.acquire_lock(timeout=lock_timeout)
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

### Testing Infrastructure Setup
```python
# tests/conftest.py
import pytest
import tempfile
import asyncio
from pathlib import Path
from moto import mock_s3
import boto3

@pytest.fixture(scope="session")
def anyio_backend():
    """Use anyio backend for async tests as per CLAUDE.md."""
    return "asyncio"

@pytest.fixture
async def temp_database():
    """Create temporary SQLite database for testing."""
    with tempfile.NamedTemporaryFile(suffix=".db") as tmp:
        db_path = tmp.name
        # Initialize test database
        await setup_test_database(db_path)
        yield db_path
        # Cleanup handled by tempfile

@pytest.fixture
def mock_s3_bucket():
    """Mock S3 bucket for sync testing."""
    with mock_s3():
        s3 = boto3.client("s3", region_name="us-east-1")
        bucket_name = "test-super-data-bucket"
        s3.create_bucket(Bucket=bucket_name)
        yield bucket_name

@pytest.fixture
def jq_binary_available():
    """Check if jq binary is available for testing."""
    import shutil
    if not shutil.which("jq"):
        pytest.skip("jq binary not available in PATH")

@pytest.fixture
async def test_user():
    """Create test user in database."""
    user_id = "test_user_123"
    async with db.session() as session:
        await create_test_user(session, user_id)
    yield user_id

@pytest.fixture
def sample_functions():
    """Load sample functions for testing."""
    return [
        {
            "name": "test_function",
            "definition": "def test_function($x): $x + 1;",
            "description": "Test function"
        }
    ]

class TestConfig:
    """Test configuration that mirrors production."""
    DATABASE_PATH = ":memory:"  # Use in-memory for speed
    S3_BUCKET = "test-bucket"
    QUERY_TIMEOUT_SECONDS = 5  # Faster timeouts for tests
    S3_LOCK_TIMEOUT_SECONDS = 10
```

### Test Execution Commands
```bash
# Run with anyio as specified in CLAUDE.md
PYTEST_DISABLE_PLUGIN_AUTOLOAD="" uv run --frozen pytest

# Run specific test categories
uv run --frozen pytest tests/unit/ -v
uv run --frozen pytest tests/integration/ -v -s

# Run with coverage
uv run --frozen pytest --cov=src --cov-report=html

# Test with multiple workers (after ensuring thread safety)
uv run --frozen pytest -n 4 tests/unit/
```

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

### Database Migration and Versioning Strategy
```python
# src/database/version.py
from typing import NamedTuple

class DatabaseVersion(NamedTuple):
    major: int
    minor: int
    patch: int

    def __str__(self) -> str:
        return f"{self.major}.{self.minor}.{self.patch}"

    @classmethod
    def from_string(cls, version_str: str) -> "DatabaseVersion":
        parts = version_str.split(".")
        return cls(int(parts[0]), int(parts[1]), int(parts[2]))

    def is_compatible_with(self, other: "DatabaseVersion") -> bool:
        """Check if this version can work with another version."""
        # Same major version = compatible
        # Different major version = incompatible, needs migration
        return self.major == other.major

CURRENT_DB_VERSION = DatabaseVersion(1, 0, 0)

async def check_database_version(db_path: str) -> bool:
    """Check if database version is compatible."""
    try:
        # Read version from database metadata table
        stored_version = await get_stored_db_version(db_path)
        return CURRENT_DB_VERSION.is_compatible_with(stored_version)
    except Exception:
        # New database, version will be set during migration
        return True

async def ensure_database_version(db_path: str) -> None:
    """Ensure database is at current version."""
    stored_version = await get_stored_db_version(db_path)

    if stored_version == CURRENT_DB_VERSION:
        return  # Already current

    if not CURRENT_DB_VERSION.is_compatible_with(stored_version):
        raise ValueError(
            f"Database version {stored_version} is incompatible with "
            f"current version {CURRENT_DB_VERSION}. Manual migration required."
        )

    # Apply minor/patch migrations
    await apply_version_migration(stored_version, CURRENT_DB_VERSION)
```

### S3 Sync Version Check Pattern
```python
# src/sync/version_check.py
async def safe_s3_sync():
    """Sync with S3 only if versions are compatible."""

    # Download version file first
    s3_version = await download_s3_version()
    local_version = await get_local_db_version()

    if not local_version.is_compatible_with(s3_version):
        logger.error(
            f"Cannot sync: local version {local_version} "
            f"incompatible with S3 version {s3_version}"
        )
        raise ValueError("Database version mismatch")

    # Versions compatible, proceed with sync
    await perform_s3_sync()
```

### Initial Database Seed Script
```python
# src/database/seed.py
async def seed_database():
    """Import YAML seeds into SQLite on first run."""

    # Ensure database is at correct version first
    await ensure_database_version(db_path)

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

    # Set database version after successful seeding
    await set_database_version(CURRENT_DB_VERSION)
```

## Important Design Decisions

1. **SQLite as Single Source of Truth**: All tools, functions, and metadata live in SQLite, YAML files are only seed data

2. **Team-First Design**: Every feature considers multi-user concurrent access and team collaboration

3. **Function Compilation**: Always compile jq functions with tools into complete programs before execution

4. **Optimistic Locking**: Use version numbers and retry logic for concurrent modifications

5. **SQLite Concurrency Realistic Expectations**: While SQLite with WAL mode supports concurrent reads, writes are still serialized. The "100+ concurrent connections" refers to read operations. For write-heavy workloads, implement a write queue or consider PostgreSQL.

6. **Approval by Default**: YAML-imported tools are pre-approved, learned tools require approval

7. **Streaming First**: Process large datasets as streams, never load entire datasets unless explicitly cached

8. **User Attribution**: Track who created, modified, and approved every tool and function

9. **Fail Fast**: Validate queries, functions, and parameters before execution

## Common Pitfalls to Avoid

1. **Don't use file-based tools in production** - SQLite is the source of truth
2. **Don't ignore locking** - Always use distributed locks for S3 sync
3. **Don't trust user input** - Validate and sanitize all jq queries
4. **Don't skip function resolution** - Always resolve dependencies before execution
5. **Don't forget WAL mode** - Enable for concurrent SQLite access
6. **Don't hardcode user IDs** - Extract from request context
7. **Don't block on approvals** - Use async notifications
8. **Don't ignore S3 sync failures** - Implement proper error recovery
9. **Don't assume jq binary availability** - Check PATH and provide clear errors

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
    async def acquire(self, timeout=45):  # Default matches config
        lock_content = {
            "owner": self.instance_id,
            "acquired_at": datetime.now().isoformat(),
            "expires_at": (datetime.now() + timedelta(seconds=timeout)).isoformat()
        }
        # Use S3 conditional puts to ensure atomicity
```

## Error Recovery Procedures

### S3 Sync Failure Recovery
```python
# src/sync/recovery.py
async def recover_from_sync_failure():
    """Recover from S3 sync failures."""

    # 1. Check local database integrity
    if not await verify_database_integrity():
        logger.error("Local database corrupted, restoring from last known good backup")
        await restore_from_backup()

    # 2. Check S3 connectivity
    try:
        await test_s3_connection()
    except Exception:
        logger.exception("S3 connection failed, working offline")
        return False

    # 3. Force download latest from S3 (ignore local changes)
    if await prompt_user_for_force_download():
        await force_download_s3_database()
        return True

    # 4. Create conflict resolution file
    await create_conflict_log()
    return False

async def handle_database_corruption():
    """Handle database file corruption."""

    # 1. Create backup of corrupted file
    corrupt_path = f"{db_path}.corrupt.{datetime.now().isoformat()}"
    shutil.copy(db_path, corrupt_path)

    # 2. Try to recover using SQLite recovery tools
    try:
        await run_sqlite_recovery(db_path)
        if await verify_database_integrity():
            return True
    except Exception:
        logger.exception("SQLite recovery failed")

    # 3. Restore from S3 if available
    if await s3_backup_exists():
        await download_s3_backup()
        return True

    # 4. Last resort: recreate from YAML seeds
    logger.warning("Recreating database from seed data")
    await recreate_database_from_seeds()
    return True
```

### Transaction Rollback Strategy
```python
# src/database/transactions.py
async def safe_database_operation(operation_func):
    """Execute database operation with automatic rollback on failure."""

    async with db.session() as session:
        savepoint = await session.begin_nested()  # Create savepoint
        try:
            result = await operation_func(session)
            await savepoint.commit()
            return result
        except Exception:
            logger.exception("Database operation failed, rolling back")
            await savepoint.rollback()
            raise
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
- **Always verify jq binary availability on startup** and provide installation instructions
- **Log all sync operations** for debugging distributed team issues
