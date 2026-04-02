# SmartNotes PostgreSQL Schema (notes_database)

This container stores notes, tags, and their relationship.

## Connection (required rule)

Always connect using:

- `db_connection.txt` (contains a full `psql postgresql://...` command)

Example:

```bash
cd smartnotes-338888-338902/notes_database
CONN="$(cat db_connection.txt)"
$CONN -c "SELECT 1;"
```

**Important:** Apply SQL statements **one-at-a-time** using `psql -c "..."`.

---

## Enabled extensions

- `pg_trgm` (for fast partial-text search via trigram GIN indexes)

---

## Tables

### `notes`
Stores the main note content and pin/favorite flags.

Columns:
- `id` UUID PK (default generated)
- `title` TEXT (non-null, default empty)
- `content` TEXT (non-null, default empty)
- `is_pinned` BOOLEAN
- `pinned_at` TIMESTAMPTZ (nullable)
- `is_favorite` BOOLEAN
- `created_at` TIMESTAMPTZ
- `updated_at` TIMESTAMPTZ (auto-maintained by trigger)
- `deleted_at` TIMESTAMPTZ (nullable; soft delete / future sync support)

### `tags`
Stores tags.

Columns:
- `id` UUID PK (default generated)
- `name` TEXT UNIQUE (case-sensitive uniqueness)
- `created_at` TIMESTAMPTZ
- `updated_at` TIMESTAMPTZ (auto-maintained by trigger)

### `note_tags`
Join table for many-to-many relationship between notes and tags.

Columns:
- `note_id` UUID FK -> `notes(id)` ON DELETE CASCADE
- `tag_id` UUID FK -> `tags(id)` ON DELETE CASCADE
- `created_at` TIMESTAMPTZ
- Primary key: `(note_id, tag_id)`

---

## Triggers

A shared trigger function maintains `updated_at` on updates:

- Function: `set_updated_at()`
- Triggers:
  - `notes_set_updated_at` on `notes`
  - `tags_set_updated_at` on `tags`

---

## Indexes

### Notes listing / sorting
- `idx_notes_created_at` on `notes(created_at DESC)`
- `idx_notes_updated_at` on `notes(updated_at DESC)`
- `idx_notes_pinned_sort` on `(is_pinned DESC, pinned_at DESC NULLS LAST, updated_at DESC)`
- `idx_notes_favorite` on `(is_favorite DESC, updated_at DESC)`

### Text search (ILIKE / partial matching)
Uses `pg_trgm`:
- `idx_notes_title_trgm` GIN on `notes(title gin_trgm_ops)`
- `idx_notes_content_trgm` GIN on `notes(content gin_trgm_ops)`
- `idx_tags_name_trgm` GIN on `tags(name gin_trgm_ops)` (autocomplete/search)

### Tag filtering
- `idx_note_tags_tag_id` on `note_tags(tag_id)`
- `idx_note_tags_note_id` on `note_tags(note_id)`

---

## Optional dev seed data

Idempotent seed inserts may be used in development:
- Tags: `work`, `personal`, `ideas`
- One sample note: "Welcome to SmartNotes"
- The sample note is tagged with `ideas`

To remove seed data, just delete from the tables (respecting FK cascades).
