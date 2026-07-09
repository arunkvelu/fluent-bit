# Tail Input: Whole File On Update

## Overview

`Whole_File_On_Update` is an optional Tail input configuration that changes the
read start offset when an already monitored file grows.

By default, Tail continues to read only appended bytes from the last stored
offset. When `Whole_File_On_Update` is enabled, Tail still uses the existing
offset database to detect whether the file has grown, but it starts the next
read from byte `0` instead of the stored offset.

The option is disabled by default, so existing Tail behavior remains unchanged.

## Configuration

```ini
[INPUT]
    Name                  tail
    Path                  /path/to/file.log
    Whole_File_On_Update  On
```

Equivalent normalized property name:

```yaml
whole_file_on_update: true
```

Default:

```text
Whole_File_On_Update Off
```

## Default Read Flow

With the option disabled, Tail keeps its existing append-only behavior.

```text
stored_offset = offset stored in memory or DB
current_size  = current file size

if current_size > stored_offset:
    seek(stored_offset)
    read appended bytes
    forward appended records
    store current_size as the next offset
```

This path is preserved for backward compatibility.

## Whole File Read Flow

With `Whole_File_On_Update` enabled, growth detection still depends on the same
offset and file size comparison. The only behavioral difference is the read
start offset.

```text
stored_offset = offset stored in memory or DB
current_size  = current file size

if current_size > stored_offset:
    seek(0)
    read the whole file
    forward all records from the file
    store current_size as the next offset
```

No additional state is written to the database. The stored offset continues to
represent the last resumable EOF position.

## Database Behavior

The Tail database keeps the same schema and semantics.

The database still stores the current file offset after successful processing.
When Fluent Bit restarts, Tail restores that offset exactly as before. If the
file has not grown beyond the stored offset, no data is replayed. If the file
has grown and `Whole_File_On_Update` is enabled, Tail uses the restored offset
only to detect growth, then seeks to the beginning of the file for the read.

This means the database remains responsible for:

- inode tracking
- persisted offset lookup
- stale file cleanup
- rotation-related file state
- restart recovery

The new option does not add checksums, hashes, timestamps, markers, or any new
database columns.

## Implementation Details

The feature is centered around `flb_tail_file_set_pending_bytes()`.

That helper receives the current file size and decides the next read start:

```text
if file->offset >= current_size:
    pending_bytes = 0

else if ctx->whole_file_on_update and file is in event mode:
    start_offset = 0
    seek(0)
    file->offset = 0
    pending_bytes = current_size

else:
    start_offset = file->offset
    pending_bytes = current_size - file->offset
```

After the read completes, the existing Tail processing path advances
`file->offset` normally. The existing offset update logic then persists the EOF
position through the current database update path.

## Modified Code Paths

The new helper is reused by the paths that already detect file growth or prepare
pending bytes. It only rewinds to the beginning after the file has been promoted
to event mode, so startup scans and database-offset catch-up keep their normal
offset-based behavior:

- static file promotion to event mode in `tail_file.c`
- pending event collection in `tail.c`
- inotify growth reconciliation in `tail_fs_inotify.c`
- stat-based growth reconciliation in `tail_fs_stat.c`

The stat and inotify paths only apply the seek-to-zero behavior when the file
size increases. If a whole-file reread is still draining in multiple chunks,
subsequent checks continue from the current in-memory offset instead of
restarting from byte `0` again.

## Preserved Behavior

The option does not change Tail behavior outside the read start offset decision.

Unchanged areas include:

- inode tracking
- file discovery
- rotation detection
- truncation handling
- database management
- multiline parsing
- buffering
- backpressure handling
- existing configuration defaults

When `Whole_File_On_Update` is disabled, Tail reads from the stored offset just
as it did before this change.

## Test Coverage

Runtime tests were added for:

- `whole_file_on_update`: verifies that appending to a monitored file causes the
  whole current file to be forwarded.
- `db_whole_file_on_update`: verifies that the database offset is still restored
  after restart, and that whole-file forwarding only happens after the file
  grows beyond the stored offset.
