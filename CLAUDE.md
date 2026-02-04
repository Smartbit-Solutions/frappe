# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Frappe Framework is a full-stack, metadata-driven web application framework built with Python (backend) and JavaScript/Vue.js (frontend). It provides a low-code platform for building enterprise applications with built-in support for multi-tenancy, permissions, workflows, and background processing. The framework was built to power ERPNext and serves as the foundation for all Frappe apps.

**Current Branding**: This repository is being rebranded from "Frappe Framework" to "Smartbits LCS" (see commit 69af979).

**Tech Stack**:
- Backend: Python 3.14, PyPika (query builder), RQ (job queue), Redis (cache/queue)
- Database: MariaDB (primary), PostgreSQL, SQLite (supported)
- Frontend: Vue.js 3, jQuery, esbuild (bundler)
- Server: Werkzeug (WSGI), Gunicorn (production)
- Testing: Python unittest, Cypress (E2E)

## Development Commands

### Environment Setup

This project uses **bench** as its development environment manager. Bench must be installed separately.

```bash
# Start development server (requires bench)
bench start

# Create a new site (one-time setup)
bench new-site <sitename>

# Access site in browser
http://<sitename>:8000/app
```

### Building Frontend Assets

```bash
# Development build with watch mode
npm run watch
# or: node esbuild --watch

# Production build
npm run production
# or: node esbuild --production

# Single build
npm run build
# or: node esbuild
```

The build system uses esbuild and is configured in `esbuild/esbuild.js`. It bundles JavaScript, Vue components, and SCSS files.

### Running Tests

#### Python Tests

```bash
# Run all tests for a site
bench --site <sitename> run-tests

# Run tests for specific app
bench --site <sitename> run-tests --app frappe

# Run tests for specific module
bench --site <sitename> run-tests --module frappe.tests.test_utils

# Run tests for specific doctype
bench --site <sitename> run-tests --doctype "User"

# Run with verbosity
bench --site <sitename> run-tests --verbose

# Run with coverage
bench --site <sitename> run-tests --coverage

# Run specific test case
bench --site <sitename> run-tests --test frappe.tests.test_utils.TestUtils.test_cint
```

Tests are located in:
- `frappe/tests/` - Core framework tests
- `frappe/*/tests/` - Module-specific tests
- `frappe/*/doctype/*/test_*.py` - DocType-specific tests

#### Frontend/Cypress Tests

```bash
# Run Cypress tests (requires test_site_ui to be running)
npx cypress run

# Open Cypress interactive mode
npx cypress open
```

Configuration: `cypress.config.js`

### Code Quality

```bash
# Lint Python code with ruff
ruff check .

# Format Python code with ruff
ruff format .

# Lint JavaScript/Vue with ESLint
npx eslint frappe/public/js

# Format JavaScript/Vue with Prettier
npx prettier --write "frappe/**/*.{js,vue,scss}"

# Run pre-commit hooks manually
pre-commit run --all-files
```

**Important**: The project uses pre-commit hooks that run ruff (Python linter/formatter), prettier, and eslint automatically. These are configured in `.pre-commit-config.yaml`.

### Type Checking

```bash
# Run mypy on typed modules (limited scope currently)
mypy frappe/types
```

Only select modules are type-checked (see `pyproject.toml` [tool.mypy] section).

### Database Operations

```bash
# Run database migrations
bench --site <sitename> migrate

# Reset database (destructive!)
bench --site <sitename> reinstall

# Enter database console
bench --site <sitename> mariadb
# or: bench --site <sitename> postgres
```

## Architecture Overview

### Request Flow

```
HTTP Request
    ↓
frappe/app.py::application() (WSGI entry point)
    ↓
init_request() → Initialize site context, authenticate user
    ↓
Route to handler:
    - /api/* → frappe/api/ (REST API)
    - /app/* → frappe/desk/ (Desk UI)
    - /files/* → frappe/utils/file_manager.py
    - Website routes → frappe/website/
    ↓
sync_database() → Commit/rollback based on HTTP method
    ↓
Return Response
```

### Core Concepts

#### 1. DocType System (Metadata-Driven Models)

DocTypes are the core abstraction in Frappe - they define both the schema and behavior of data models:

- **DocType**: Schema definition stored in database (`tabDocType`, `tabDocField`)
- **Document**: Instance of a DocType (like a database row, but with rich behavior)
- **Meta**: Runtime representation of DocType schema (`frappe.model.meta.Meta`)

```python
# Get a document
doc = frappe.get_doc("User", "user@example.com")

# Create a new document
doc = frappe.new_doc("Task")
doc.title = "My Task"
doc.insert()

# Get DocType metadata
meta = frappe.get_meta("User")
fields = meta.get("fields")
```

**Document Lifecycle Hooks** (defined in DocType controller or hooks.py):
- `validate`, `before_insert`, `after_insert`, `before_save`, `after_save`
- `before_submit`, `after_submit`, `before_cancel`, `after_cancel`
- `before_delete`, `after_delete`, `on_trash`, `after_delete`

#### 2. Thread-Local Context (`frappe.local`)

Frappe uses thread-local storage to maintain request-specific state without passing context through every function:

```python
frappe.local.site          # Current site name
frappe.local.user          # Current authenticated user
frappe.local.db            # Database connection for this request
frappe.local.qb            # Query builder instance
frappe.local.request       # Werkzeug Request object
frappe.local.response      # Response dict (will be JSON-encoded)
frappe.local.flags         # Request flags for state management
```

This is initialized per request in `frappe.init()` and cleaned up after response.

#### 3. Database Abstraction

The framework supports multiple databases through a common abstraction layer:

```python
# Direct database queries
frappe.db.get_value("User", "user@example.com", "full_name")
frappe.db.get_all("Task", filters={"status": "Open"}, fields=["name", "title"])
frappe.db.sql("SELECT * FROM `tabUser` WHERE enabled=1")

# Query builder (preferred for complex queries)
from frappe.query_builder import DocType

Task = DocType("Task")
tasks = (
    frappe.qb.from_(Task)
    .select(Task.name, Task.title)
    .where(Task.status == "Open")
    .run(as_dict=True)
)
```

**Database Files**:
- `frappe/database/database.py` - Abstract base class
- `frappe/database/mariadb/database.py` - MariaDB implementation
- `frappe/database/postgres/database.py` - PostgreSQL implementation

#### 4. Permission System

Multi-level permission system with role-based access control:

```python
# Check permissions
frappe.has_permission("User", "read", doc)
frappe.check_permission("User", "write")  # Raises exception if no access

# Permission types
# "read", "write", "create", "delete", "submit", "cancel", "amend"
# "print", "email", "report", "import", "export", "share"
```

**Permission Hierarchy**:
1. Administrator role (bypasses all checks)
2. Role Permissions (DocPerm table)
3. User Permissions (field-level restrictions)
4. Document Sharing (per-document sharing)
5. Document ownership

Permissions are defined in `frappe/permissions.py`.

#### 5. Background Jobs

Uses Redis Queue (RQ) for asynchronous job processing:

```python
# Enqueue a background job
frappe.enqueue(
    method="my_app.tasks.long_running_task",
    queue="default",           # "short", "default", "long"
    timeout=300,
    is_async=True,
    job_id="unique-job-id",
    **kwargs
)

# Enqueue after database commit
frappe.enqueue_after_commit(method, **kwargs)

# Publish realtime progress
frappe.publish_progress(
    percent=50,
    title="Processing",
    description="Importing records...",
    task_id=frappe.local.task_id
)
```

**Queue types**:
- `short`: Quick tasks (5 min timeout)
- `default`: Standard tasks (5 min timeout)
- `long`: Long-running tasks (25 min timeout)

Background job implementation: `frappe/utils/background_jobs.py`

#### 6. Hooks System

Apps extend Frappe without modifying core code via hooks defined in `hooks.py`:

```python
# App's hooks.py
doc_events = {
    "User": {
        "after_insert": "my_app.user.after_insert",
        "before_save": ["my_app.user.validate_email"]
    }
}

after_request = ["my_app.utils.cleanup"]
boot_session = ["my_app.boot.add_bootinfo"]
```

See `hooks.md` for complete list of available hooks.

#### 7. Whitelisting Pattern

Functions must be explicitly whitelisted to be callable via HTTP API:

```python
@frappe.whitelist()
def my_api_method(arg1, arg2):
    """Callable via /api/method/my_app.module.my_api_method"""
    return {"result": arg1 + arg2}

@frappe.whitelist(allow_guest=True)
def public_api():
    """Accessible without authentication"""
    pass

@frappe.whitelist(methods=["GET", "POST"])
def restricted_method():
    """Only allow specific HTTP methods"""
    pass
```

### Directory Structure

```
frappe/
├── api/                    # REST API handlers (v1, v2)
├── app.py                  # WSGI application entry point
├── auth.py                 # Authentication (session, OAuth, API keys)
├── boot.py                 # Bootstrap data sent to frontend
├── commands/               # CLI commands (via bench)
├── core/                   # Core DocTypes (User, File, etc.)
├── database/               # Database abstraction layer
│   ├── mariadb/
│   ├── postgres/
│   └── sqlite/
├── desk/                   # Desk UI (form, list, report views)
├── email/                  # Email sending and processing
├── handler.py              # Request routing and handling
├── hooks.py                # Hook registry and executor
├── model/                  # Document model layer
│   ├── document.py         # Document base class
│   ├── meta.py             # DocType metadata
│   └── naming.py           # Auto-naming patterns
├── permissions.py          # Permission checking logic
├── public/                 # Frontend assets
│   ├── js/                 # JavaScript/Vue code
│   ├── css/                # SCSS stylesheets
│   └── dist/               # Built assets (git-ignored)
├── query_builder/          # PyPika-based query builder
├── tests/                  # Core framework tests
├── types/                  # Type definitions (Python typing)
├── utils/                  # Utility functions
│   ├── background_jobs.py  # RQ job queue
│   ├── data.py             # Data import/export
│   └── file_manager.py     # File handling
└── website/                # Website/portal rendering

esbuild/                    # Frontend build configuration
cypress/                    # E2E tests
realtime/                   # Socket.IO server for realtime updates
```

### Key Files Reference

| Component | File Path |
|-----------|-----------|
| WSGI Entry Point | `frappe/app.py` |
| Framework Init | `frappe/__init__.py` |
| Authentication | `frappe/auth.py` |
| Request Handler | `frappe/handler.py` |
| REST API | `frappe/api/__init__.py`, `frappe/api/v1.py` |
| Document Model | `frappe/model/document.py` |
| Metadata | `frappe/model/meta.py` |
| Permissions | `frappe/permissions.py` |
| Database | `frappe/database/database.py` |
| Query Builder | `frappe/query_builder/` |
| Background Jobs | `frappe/utils/background_jobs.py` |
| Form Handling | `frappe/desk/form/` |
| Client Bootstrap | `frappe/boot.py` |

## Common Development Patterns

### Creating a New DocType

1. **Via UI**: Go to Desk → DocType → New
2. **Via Code**: Create JSON files in `my_app/my_module/doctype/my_doctype/`
   - `my_doctype.json` - DocType definition
   - `my_doctype.py` - Controller class (inherits from `frappe.model.document.Document`)
   - `test_my_doctype.py` - Unit tests

```python
# my_doctype.py
import frappe
from frappe.model.document import Document

class MyDocType(Document):
    def validate(self):
        """Called before save"""
        if not self.title:
            frappe.throw("Title is required")

    def after_insert(self):
        """Called after first save"""
        frappe.log_info(f"Created {self.name}")
```

### Accessing Documents

```python
# Get existing document
doc = frappe.get_doc("User", "admin@example.com")

# Create new document
doc = frappe.new_doc("Task")
doc.update({
    "title": "My Task",
    "status": "Open"
})
doc.insert()

# Save changes
doc.title = "Updated Title"
doc.save()

# Delete document
doc.delete()
```

### Database Queries

```python
# Simple queries
users = frappe.get_all("User",
    filters={"enabled": 1},
    fields=["name", "email", "full_name"],
    order_by="creation desc",
    limit=10
)

# Get single value
full_name = frappe.db.get_value("User", "admin@example.com", "full_name")

# Complex query with query builder
from frappe.query_builder import DocType
from frappe.query_builder.functions import Count

User = DocType("User")
result = (
    frappe.qb.from_(User)
    .select(User.role, Count("*").as_("count"))
    .where(User.enabled == 1)
    .groupby(User.role)
    .run(as_dict=True)
)
```

### Creating API Endpoints

```python
# In my_app/api.py
import frappe

@frappe.whitelist()
def get_user_stats(user):
    """
    Callable via:
    POST /api/method/my_app.api.get_user_stats
    Body: {"user": "admin@example.com"}
    """
    return frappe.get_all("Task",
        filters={"assigned_to": user},
        fields=["status", "count(*) as count"],
        group_by="status"
    )
```

### Working with Files

```python
# Get file
file_doc = frappe.get_doc("File", {"file_url": "/files/myfile.pdf"})

# Save file from URL
file_doc = frappe.get_doc({
    "doctype": "File",
    "file_url": uploaded_file_path,
    "attached_to_doctype": "User",
    "attached_to_name": "user@example.com"
})
file_doc.insert()
```

### Frontend (Desk) Customization

Frontend code is in `frappe/public/js/`. Key global objects:

- `frappe.ui.form.Form` - Form view controller
- `frappe.ui.page.Page` - Page controller
- `frappe.listview.ListView` - List view controller
- `frappe.call()` - Make API calls to backend

```javascript
// Add custom button to form
frappe.ui.form.on('User', {
    refresh: function(frm) {
        frm.add_custom_button('Send Email', () => {
            frappe.call({
                method: 'my_app.api.send_email',
                args: {user: frm.doc.name},
                callback: (r) => {
                    frappe.msgprint('Email sent!');
                }
            });
        });
    }
});
```

## Important Notes

### Code Style

- **Python**: Uses ruff for linting and formatting (tabs for indentation, 110 char line length)
- **JavaScript**: ESLint + Prettier (check `.eslintrc` and prettier config in `.pre-commit-config.yaml`)
- **Commit Messages**: Uses conventional commits format (enforced by commitlint)

### Pre-commit Hooks

The project uses pre-commit hooks that will:
- Format Python code with ruff
- Sort Python imports
- Format JS/Vue with prettier
- Lint JS with eslint
- Check for trailing whitespace, merge conflicts, etc.
- **Prevent direct commits to `develop` branch**

### Testing Conventions

- All test files must be named `test_*.py`
- Test classes should inherit from `unittest.TestCase` or `frappe.tests.IntegrationTestCase`
- Use `frappe.set_user("Administrator")` in tests to set context
- Clean up test data in `tearDown()` or use `frappe.delete_doc()` with `force=True`

### Database Migrations

When changing DocType schemas:
1. Make changes via Desk UI or modify JSON files
2. Run `bench migrate` to apply changes
3. For data migrations, create patch files in `frappe/patches/`
4. Add patch to `frappe/patches.txt`

### Multi-tenancy

Frappe is multi-tenant by design:
- Each site has its own database
- Sites are identified by hostname or domain
- Site data is in `sites/<sitename>/`
- Use `frappe.init(site)` to switch context programmatically

### Caching

Multi-level caching strategy:
- `frappe.cache` - Redis-based site cache
- `frappe.db.get_value(..., cache=True)` - Query result caching
- `frappe.get_cached_doc()` - Document caching
- Clear cache: `bench clear-cache` or `frappe.clear_cache()`

### Debugging

```python
# Set breakpoint (works with bench start)
import pdb; pdb.set_trace()

# Or use frappe's debugger
frappe.debug()

# Log debugging info
frappe.log_error("Error message", "Error Title")
frappe.msgprint("User-facing message")
```

### Common Gotchas

1. **Always use `frappe.db.commit()` carefully**: Most operations auto-commit, manual commits can cause issues
2. **Check permissions**: Use `frappe.has_permission()` before accessing documents in API methods
3. **Whitelist methods**: API methods must have `@frappe.whitelist()` decorator
4. **Site context**: Always ensure `frappe.init(site)` is called before using frappe methods in CLI scripts
5. **Child table access**: Child tables use `doc.append("child_table_fieldname", {})` not direct assignment
