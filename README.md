
# Alembic Database Migration Guide

Created by: Sara Mahran
Created time: January 19, 2026 3:52 PM
Category: PostgreSQL, Version Management
Last edited by: Samer Adel
Last updated time: January 19, 2026 5:32 PM

# Alembic Database Migration Guide

# Table of Contents

1. [Migration Fundamentals](#migration-fundamentals)
2. [Table Renaming Strategy](#table-renaming-strategy)
3. [Manual Migration Creation](#manual-migration-creation)
4. [Splitting Tables (Table Normalization)](#splitting-tables-table-normalization)
5. [Adding New Tables with Data](#adding-new-tables-with-data)
6. [Migration File Naming Constraints](#migration-file-naming-constraints)
7. [Common Patterns & Best Practices](#common-patterns--best-practices)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Summary Checklist](#summary-checklist)
10. [Quick Reference Commands](#quick-reference-commands)


---

## Migration Fundamentals

### How Alembic Tracks Migrations

- **`alembic_version` table**: Stores current revision ID in database
- **Revision chain**: Each migration points to previous one via `down_revision`
- **Migration files**: Located in `migrations/versions/` directory

### Basic Commands

```bash
# Create auto-generated migration
alembic revision --autogenerate -m "description"

# Apply migrations
alembic upgrade head  # Runs once when we run our application

# Check current state
alembic current
alembic history

# Rollback one migration
alembic downgrade -1

# Mark specific version
alembic stamp <revision_id>

```

---

## Table Renaming Strategy

### ❌ WRONG: Drop and Recreate

```python
def upgrade():
    op.drop_table('old_table_name')
    op.create_table('new_table_name', ...)
    # PROBLEM: Data loss!

```

### ✅ CORRECT: Use `rename_table()`

```python
def upgrade():
    op.rename_table('old_table_name', 'new_table_name')

def downgrade():
    op.rename_table('new_table_name', 'old_table_name')

```

### Complete Rename Example

```python
"""rename_users_to_accounts

Revision ID: xxxxx
Revises: previous_revision
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op

revision = 'xxxxx'
down_revision = 'previous_revision'

def upgrade():
    # Rename table
    op.rename_table('users', 'accounts')

    # Rename foreign key constraints (if needed)
    op.execute("ALTER TABLE user_profiles RENAME CONSTRAINT users_id_fkey TO accounts_id_fkey")

def downgrade():
    # Reverse rename
    op.rename_table('accounts', 'users')
    op.execute("ALTER TABLE user_profiles RENAME CONSTRAINT accounts_id_fkey TO users_id_fkey")

```

---

## Manual Migration Creation

### Step-by-Step Process

### 1. Find Current Database Revision

```bash
# Method 1: Use Alembic
alembic current
# Output: 620bd8a4d537 (head)

# Method 2: Check database directly
psql -U postgres -d your_db -c "SELECT version_num FROM alembic_version;"

# Or in pgAdmin or similar query tool
SELECT version_num from "alembic_version"

# Method 3: Check latest migration file
cd migrations/versions
ls -la  # Find most recent file

```

### 2. Generate New Revision ID

```bash
# Generate 12-character UUID
python -c "import uuid; print(str(uuid.uuid4()).replace('-', '')[:12])"
# Example output: 5cd5f609e319

```

### 3. Create Migration File

```python
"""your_migration_description

Revision ID: 5cd5f609e319        # Generated ID
Revises: 620bd8a4d537            # Current DB version
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import text  # IMPORTANT: For raw SQL

revision = '5cd5f609e319'
down_revision = '620bd8a4d537'
branch_labels = None
depends_on = None

def upgrade():
    """Your migration logic here."""
    pass

def downgrade():
    """Rollback logic here."""
    pass

```

### 4. Apply Migration

```bash
# Apply your manual migration
alembic upgrade head 
# In our case just run 
python app.py

# Verify
alembic current  # Should show your new revision ID

```

### Critical Notes for Manual Migrations

1. **Always import `text` for raw SQL**: `from sqlalchemy.sql import text`
2. **Wrap all raw SQL with `text()`**: `conn.execute(text("SELECT ..."))`
3. **Check column/table existence** before operations
4. **Handle data preservation** carefully

---

## Splitting Tables (Table Normalization)

### Scenario: Split `user_profiles` into normalized tables

### Step 1: Create Empty Normalized Tables

```python
"""create_user_educations_and_work_experiences_tables

Revision ID: da88c8f9b1a1
Revises: previous_revision
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

revision = 'da88c8f9b1a1'
down_revision = 'previous_revision'

def upgrade():
    # Create empty normalized tables
    op.create_table('user_educations',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('user_profile_id', sa.Integer(), nullable=False),
        sa.Column('university', sa.String(255), nullable=True),
        sa.Column('degree', sa.String(100), nullable=True),
        sa.Column('field_of_study', sa.String(100), nullable=True),
        sa.Column('graduation_year', sa.Integer(), nullable=True),
        sa.ForeignKeyConstraint(['user_profile_id'], ['user_profiles.id']),
        sa.PrimaryKeyConstraint('id')
    )

    op.create_table('user_work_experiences',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('user_profile_id', sa.Integer(), nullable=False),
        sa.Column('company', sa.String(255), nullable=True),
        sa.Column('position', sa.String(100), nullable=True),
        sa.Column('start_date', sa.Date(), nullable=True),
        sa.Column('end_date', sa.Date(), nullable=True),
        sa.Column('description', sa.Text(), nullable=True),
        sa.ForeignKeyConstraint(['user_profile_id'], ['user_profiles.id']),
        sa.PrimaryKeyConstraint('id')
    )

def downgrade():
    op.drop_table('user_work_experiences')
    op.drop_table('user_educations')

```

### Step 2: Migrate Data to New Tables

```python
"""migrate_data_to_normalized_tables

Revision ID: 8f3e5a1b2c9d
Revises: da88c8f9b1a1
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op
from sqlalchemy.sql import text

revision = '8f3e5a1b2c9d'
down_revision = 'da88c8f9b1a1'

def upgrade():
    conn = op.get_bind()

    # Copy primary education data
    conn.execute(text("""
        INSERT INTO user_educations
        (user_profile_id, university, degree, graduation_year, is_current)
        SELECT
            id,
            primary_university,
            primary_degree,
            graduation_year,
            true
        FROM user_profiles
        WHERE primary_university IS NOT NULL
           OR primary_degree IS NOT NULL
           OR graduation_year IS NOT NULL
    """))

    # Copy current work data
    conn.execute(text("""
        INSERT INTO user_work_experiences
        (user_profile_id, company, position, start_date, is_current)
        SELECT
            id,
            current_company,
            current_position,
            employment_start,
            true
        FROM user_profiles
        WHERE current_company IS NOT NULL
           OR current_position IS NOT NULL
           OR employment_start IS NOT NULL
    """))

    print("✅ Data migrated to normalized tables")

def downgrade():
    # Remove migrated data (cannot restore automatically)
    conn = op.get_bind()
    conn.execute(text("DELETE FROM user_work_experiences"))
    conn.execute(text("DELETE FROM user_educations"))
    print("⚠️  Data removed - original columns not restored")

```

### Step 3: (Optional) Remove Old Columns

```python
"""remove_old_columns_from_user_profiles

Revision ID: c3bd09603406
Revises: 8f3e5a1b2c9d
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op

revision = 'c3bd09603406'
down_revision = '8f3e5a1b2c9d'

def upgrade():
    # Remove individual columns after data migration
    op.drop_column('user_profiles', 'primary_university')
    op.drop_column('user_profiles', 'primary_degree')
    op.drop_column('user_profiles', 'graduation_year')
    op.drop_column('user_profiles', 'current_company')
    op.drop_column('user_profiles', 'current_position')
    op.drop_column('user_profiles', 'employment_start')
    print("✅ Old columns removed")

def downgrade():
    # Add columns back (empty)
    op.add_column('user_profiles', sa.Column('primary_university', sa.String(255)))
    op.add_column('user_profiles', sa.Column('primary_degree', sa.String(100)))
    op.add_column('user_profiles', sa.Column('graduation_year', sa.Integer()))
    op.add_column('user_profiles', sa.Column('current_company', sa.String(255)))
    op.add_column('user_profiles', sa.Column('current_position', sa.String(100)))
    op.add_column('user_profiles', sa.Column('employment_start', sa.Date()))
    print("⚠️  Columns added back empty - data not restored")

```

### Normalization Migration Flow

```
Original Table
     ↓
Create Empty Normalized Tables
     ↓
Migrate Data to New Tables
     ↓
Optionally Remove Old Columns
     ↓
Update Application Code

```

---

## Adding New Tables with Data

### Handling NOT NULL Constraints

When adding columns with `nullable=False` to existing tables with data:

### ❌ WRONG: Causes Error

```python
op.add_column('users', sa.Column('status', sa.String(50), nullable=False))
# ERROR: column "status" contains null values

```

### ✅ CORRECT: Add with Default First

```python
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import text

def upgrade():
    # Step 1: Add column as nullable with default
    op.add_column('users',
        sa.Column('status', sa.String(50),
                  nullable=True,
                  server_default='active')
    )

    # Step 2: Update existing NULL values
    conn = op.get_bind()
    conn.execute(text("UPDATE users SET status = 'active' WHERE status IS NULL"))

    # Step 3: Make column NOT NULL
    op.alter_column('users', 'status',
        existing_type=sa.String(50),
        nullable=False,
        server_default=None  # Remove default if no longer needed
    )

```

### Adding New Table with Initial Data

```python
"""create_user_roles_with_initial_data

Revision ID: xxxxx
Revises: previous_revision
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import text

revision = 'xxxxx'
down_revision = 'previous_revision'

def upgrade():
    # Create table
    op.create_table('user_roles',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(50), nullable=False),
        sa.Column('description', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.text('NOW()')),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('name')
    )

    # Insert initial data
    conn = op.get_bind()
    conn.execute(text("""
        INSERT INTO user_roles (name, description) VALUES
        ('admin', 'Administrator with full access'),
        ('user', 'Regular user'),
        ('guest', 'Read-only access')
    """))

    # Add foreign key to users table
    op.add_column('users',
        sa.Column('role_id', sa.Integer(),
                  sa.ForeignKey('user_roles.id'),
                  nullable=True,
                  server_default='2')  # Default to 'user' role
    )

def downgrade():
    # Remove foreign key
    op.drop_constraint('users_role_id_fkey', 'users', type_='foreignkey')
    op.drop_column('users', 'role_id')

    # Drop table (data will be lost)
    op.drop_table('user_roles')

```

---

## Migration File Naming Constraints

### Auto-generated Filename Format

```
{revision_hash}_{sanitized_message}.py

```

Example: `19f4fb1de705_create_users_table.py`

### Character Restrictions

| Input | Becomes | Status |
| --- | --- | --- |
| `create users table` | `create_users_table` | ✅ Good |
| `fix: user auth` | `fix_user_auth` | ✅ Acceptable |
| `add/user/table` | `add_user_table` | ⚠️ Slash removed |
| `update*table` | `update_table` | ⚠️ Asterisk removed |
| `"quotes" test` | `quotes_test` | ⚠️ Quotes removed |

### Best Practices for Naming

```bash
# ✅ Use snake_case
alembic revision -m "add_email_column"

# ✅ Keep it descriptive but concise
alembic revision -m "create_user_profiles"

# ✅ Use action prefixes
alembic revision -m "create_table_users"
alembic revision -m "alter_table_add_email"
alembic revision -m "drop_column_legacy_data"

# ❌ Avoid special characters
alembic revision -m "fix/user: auth*system"  # Bad

# ❌ Avoid spaces (they become underscores anyway)
alembic revision -m "add user email"  # Becomes add_user_email

```

### Manual File Renaming

If you need to rename a migration file:

1. **Keep the revision ID** (first part before underscore)
2. **Only change the descriptive part**
3. **Update the `revision` variable** in the file (must match filename prefix)

Example:

```
# Original: 19f4fb1de705_create_users.py
# Rename to: 19f4fb1de705_create_users_table.py
# ✅ OK: Same revision ID, different description

# Wrong: create_users_19f4fb1de705.py
# ❌ ERROR: Revision ID not at beginning

```

---

## Common Patterns & Best Practices

### 1. Data Migration Safety Checks

```python
def upgrade():
    conn = op.get_bind()

    # Check column exists before dropping
    result = conn.execute(
        text("SELECT column_name FROM information_schema.columns "
             "WHERE table_name = 'users' AND column_name = 'old_column'")
    ).fetchone()

    if result:
        op.drop_column('users', 'old_column')

    # Check table exists before creating
    result = conn.execute(
        text("SELECT table_name FROM information_schema.tables "
             "WHERE table_name = 'new_table'")
    ).fetchone()

    if not result:
        op.create_table('new_table', ...)

```

### 2. Batch Updates for Large Tables

```python
def upgrade():
    conn = op.get_bind()

    # Process in batches to avoid locking
    batch_size = 1000
    offset = 0

    while True:
        result = conn.execute(
            text("""
                UPDATE users
                SET status = 'active'
                WHERE id IN (
                    SELECT id FROM users
                    WHERE status IS NULL
                    LIMIT :limit OFFSET :offset
                )
                RETURNING id
            """),
            {'limit': batch_size, 'offset': offset}
        ).fetchall()

        if not result:
            break

        offset += batch_size
        print(f"Processed {offset} records...")

```

### 3. Transaction Safety

```python
def upgrade():
    # Wrap in try-except for safety
    try:
        conn = op.get_bind()

        # Multiple operations in transaction
        op.add_column('users', sa.Column('new_field', sa.String(50)))

        conn.execute(text("UPDATE users SET new_field = 'default'"))

        op.alter_column('users', 'new_field', nullable=False)

        print("✅ Migration completed successfully")

    except Exception as e:
        print(f"❌ Migration failed: {e}")
        # Transaction will be rolled back automatically
        raise

```

### 4. Index Creation

## **Database Index = Speed Boost for Queries**

```sql
-- WITHOUT index: Database checks EVERY row (SLOW)
SELECT * FROM users WHERE email = 'john@example.com';
-- Checks: row1? no, row2? no, row3? no... row1000000? maybe

-- WITH index: Database jumps directly (FAST)
SELECT * FROM users WHERE email = 'john@example.com';
-- Checks index: "john@example.com" -> row 54872 ✓
```

```python
def upgrade():
    # Create indexes after data migration
    op.create_index('ix_users_email', 'users', ['email'], unique=True)
    op.create_index('ix_user_profiles_user_id', 'user_profiles', ['user_id'])

    # Composite index
    op.create_index('ix_articles_category_date', 'articles', ['category', 'created_at'])

```

### 5. Environment-Specific Migrations

```python
import os

def upgrade():
    # Only run in development
    if os.getenv('ENVIRONMENT') == 'development':
        op.add_column('users', sa.Column('debug_flag', sa.Boolean()))

    # Only run in production
    if os.getenv('ENVIRONMENT') == 'production':
        # Add performance indexes
        op.create_index('ix_production_optimization', 'large_table', ['column'])

```

### 6. Migration Dependencies

```python
"""migration_with_dependency

Revision ID: xxxxx
Revises: previous_revision
Create Date: 2026-01-19 00:00:00.000000

"""
from alembic import op

# Wait for another migration to complete
depends_on = 'other_revision_id'  # Optional dependency

def upgrade():
    # This runs AFTER the dependent migration
    pass

```

---

## Troubleshooting Guide

### Common Errors and Solutions

| Error | Cause | Solution |
| --- | --- | --- |
| `Not an executable object` | Raw SQL string without `text()` | Import and use `from sqlalchemy.sql import text` |
| `column contains null values` | Adding NOT NULL column to table with data | Add with default value first, then update, then set NOT NULL |
| `relation already exists` | Trying to create duplicate table/column | Check existence before creating |
| `foreign key violation` | Referenced data doesn't exist | Insert referenced data first or use `ondelete` |
| `revision id mismatch` | Filename revision doesn't match variable | Ensure `revision = 'xxx'` matches filename prefix |

### Debugging Migration Failures

```bash
# 1. Check current state
alembic current
alembic history

# 2. Test migration
alembic upgrade +1 --sql  # Show SQL without executing

# 3. Check failed migration
alembic heads  # See all head revisions

# 4. Fix and continue
alembic stamp <revision>  # Mark as applied
alembic upgrade head     # Continue

```

---

## Summary Checklist

### Before Creating Migration

- [ ]  Backup database
- [ ]  Check current revision: `alembic current`
- [ ]  Review model changes
- [ ]  Plan data migration strategy

### During Migration Creation

- [ ]  Use `rename_table()` instead of drop/create
- [ ]  Handle NOT NULL columns with defaults
- [ ]  Wrap raw SQL with `text()`
- [ ]  Add existence checks
- [ ]  Include proper downgrade logic

### After Migration

- [ ]  Test upgrade/downgrade cycle
- [ ]  Verify data integrity
- [ ]  Update application code
- [ ]  Document changes

### File Management

- [ ]  Use snake_case for migration names
- [ ]  Keep revision ID at filename start
- [ ]  One logical change per migration
- [ ]  Add descriptive comments

---

## Quick Reference Commands

```bash
# Generate new migration ID
python -c "import uuid; print(str(uuid.uuid4()).replace('-', '')[:12])"

# Create empty manual migration
alembic revision -m "description"  # Then edit

# Apply specific migration
alembic upgrade <revision>

# Mark as applied without running
alembic stamp <revision>

# Show migration SQL (dry run)
alembic upgrade head --sql

# Reset to base (DANGEROUS - deletes data)
alembic downgrade base

```
