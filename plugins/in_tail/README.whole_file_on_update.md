# Tail Input: Whole File On Update

This document is both a user guide and a design note for Tail whole-file rereads
and the S3 output buffer replacement mode that can consume them.

## Overview

`Whole_File_On_Update` is an optional Tail input configuration that changes the
read start offset when an already monitored file grows.

By default, Tail continues to read only appended bytes from the last stored
offset. When `Whole_File_On_Update` is enabled, Tail still uses the existing
offset database to detect whether the file has grown, but it starts the next
read from byte `0` instead of the stored offset.

The option is disabled by default, so existing Tail behavior remains unchanged.

Use this mode when the desired downstream object represents the latest full
contents of a source file, rather than only the appended lines.

## Quick Guide

Use `Whole_File_On_Update` only when all of these are true:

- the source file is append-only and should be treated as one complete object
- downstream readers expect one stable object/key to contain the latest full file
- rereading and re-uploading the same bytes is acceptable for the workload

Do not use this mode when the normal Tail behavior is enough. For ordinary log
shipping, where each line only needs to be delivered once, keep
`Whole_File_On_Update Off`.

For S3 latest-file behavior, the common configuration is:

```ini
[INPUT]
    Name                    tail
    Tag                     sparkeventlog.*
    Path                    /path/to/eventlog/*
    Path_Key                full_path
    Whole_File_On_Update    On
    Inotify_Watcher         False
    progress_check_interval 5

[OUTPUT]
    Name                                s3
    Match                               sparkeventlog.*
    use_put_object                      On
    static_file_path                    On
    replace_buffer_on_whole_file_update On
    s3_key_format                       /logs/app/$TAG[5]
```

Important limits:

- `Whole_File_On_Update` is input-scoped. It only affects the Tail input where
  it is configured.
- `replace_buffer_on_whole_file_update` is S3-output-scoped. It only affects S3
  outputs where it is configured.
- Already-uploaded S3 objects cannot be removed from object storage by this
  feature. The next successful upload to the same static key overwrites the
  object body.
- The current S3 replacement mode is intended for `use_put_object On`. If one
  whole-file snapshot is split into multiple uploads to the same key, later
  uploads can overwrite earlier parts.

## Use Case: Continuously Growing Files

Some workloads write append-only files that remain open and grow for a long
time. Examples include Spark event logs, database audit logs, long-running
application logs, and other event streams written to local files.

For these workloads, users may want both:

- near real-time availability of log data in object storage
- a stable object key whose contents represent the complete file collected so far

The default Tail and S3 behavior can make this difficult when the S3 output uses
a stable object key. Tail normally forwards only newly appended records. The S3
output uploads the records it has buffered when `upload_timeout` or
`total_file_size` is reached. If each upload uses the same S3 key, the new upload
replaces the previous object body with only the records in the current upload.

For example, at time `T1` a file contains:

```text
event1
event2
event3
```

When `upload_timeout` expires, S3 uploads those records to the configured key.
Later, the application appends:

```text
event4
event5
```

With normal append-only Tail reads, the next S3 upload contains only:

```text
event4
event5
```

If the S3 key is unchanged, object storage now holds only the later upload body.
The earlier events are no longer present in that object. Using a unique object
key for every upload preserves all chunks, but it does not satisfy readers that
expect one stable key to contain the complete file.

`Whole_File_On_Update` plus S3 buffer replacement is intended for this stable-key
case. Tail re-emits the full file after growth, and the S3 output can discard
stale pending local buffers for older generations before uploading the latest
complete snapshot to the same key.

This trades lower latency and stable object names for higher local CPU, disk I/O,
and network usage, because Fluent Bit rereads and re-uploads content that was
already processed.

## Basic Tail Configuration

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

## Existing Pipeline Behavior

Tail, the Fluent Bit engine, and the S3 output plugin are append-oriented by
default.

The Tail input tracks each monitored file's current offset. When a file grows,
Tail normally reads only the appended bytes, packs those bytes into log records,
and appends the records to the engine through the input chunk API.

The engine stores records in chunks keyed by input tag. It does not keep a
source-file index and it does not replace records inside already-buffered
chunks. Once Tail appends records to the engine, they are treated as new log
events for routing, filtering, buffering, retry, and output delivery.

The S3 output receives routed engine chunks and converts them into the configured
S3 payload format. It buffers payload data in its local `store_dir` until an
upload threshold is reached, `upload_timeout` expires, or Fluent Bit shuts down.
Without the S3 replacement option described below, this local S3 buffer is also
append-only.

This design intentionally does not add source-file replacement semantics to the
core engine. Engine chunks can contain records from multiple files that share a
tag, and chunks may already be routed, retried, locked, or persisted. The
replacement behavior is therefore implemented where it is needed: in the S3
output's pending local buffer.

## Buffering And Duplicate Records

`Whole_File_On_Update` changes what Tail reads, not how the engine stores data.
Each whole-file reread is emitted as a new set of Tail records.

For example, if a file contains records `A` and `B`, and then grows with record
`C`, Tail forwards `A`, `B`, and `C` again on that update. If those records are
already present in engine chunks or in an S3 local buffer, Tail does not remove
or mutate the older copies.

This is why whole-file reread mode needs an output-side replacement strategy
when the desired destination semantics are "latest full file wins".

## S3 Overwrite Behavior

S3 object overwrite is controlled by the S3 object key. When the S3 output uses
`use_put_object On` and `static_file_path On`, the plugin sends a PutObject
request to the exact key produced by `s3_key_format`.

For example:

```ini
[OUTPUT]
    Name              s3
    Match             sparkeventlog.*
    use_put_object    On
    static_file_path  On
    s3_key_format     /logs/dex/app/events_1
```

With `static_file_path On`, Fluent Bit does not append a random suffix when the
format lacks `$UUID` or `$INDEX`. Every successful PutObject for that tag writes
to the same S3 key. S3 stores one current object for that key, so the latest
successful PutObject overwrites the previous object body.

This overwrite happens only after upload. It does not prevent older records from
being appended into the local S3 buffer before upload. If a whole-file reread is
split across multiple S3 uploads and all uploads target the same key, later
segments can overwrite earlier segments. For latest-full-file semantics, the
formatted whole-file payload must fit in one PutObject buffer, or the output
must use a multipart-aware replacement design.

The current S3 replacement mode requires `use_put_object On`. PutObject mode is
best for a stable "latest object wins" key, but it also means the complete
formatted payload for a snapshot should fit within the configured PutObject
buffer. If the source event log can grow beyond that limit, use unique object
keys for chunks or design multipart replacement semantics separately.

## S3 Buffer Replacement Behavior

The S3 output can opt in to replacing stale pending buffers from Tail whole-file
updates:

```ini
[OUTPUT]
    Name                                s3
    Match                               sparkeventlog.*
    use_put_object                      On
    static_file_path                    On
    replace_buffer_on_whole_file_update On
```

When Tail starts a whole-file reread, it marks the emitted log records with
internal log-event metadata:

```text
flb.tail.whole_file_on_update.active
flb.tail.whole_file_on_update.source
flb.tail.whole_file_on_update.generation
```

The S3 output inspects this metadata before formatting and buffering a chunk. If
`replace_buffer_on_whole_file_update` is enabled and S3 finds an unlocked local
buffer for the same Tail source with an older generation, it deletes that local
buffer before appending the newer generation. Chunks from the same generation
continue to append to the same local buffer, so a whole-file reread that spans
multiple engine chunks is preserved.

This replacement is intentionally scoped to S3 local buffers. It does not remove
records already flushed to S3, records in engine chunks, or buffers that are
locked because an upload is already in progress.

If the same Tail records are also routed to a Forward output, consider disabling
Forward metadata retention for compatibility with older Fluentd pipelines:

```ini
[OUTPUT]
    Name                            forward
    Match                           dex-app.log.*
    Require_ack_response            true
    retain_metadata_in_forward_mode false
```

This keeps Fluent Bit internal log-event metadata out of the Forward protocol
payload sent to Fluentd. It does not change Tail read behavior.

## Inotify And Polling For Whole-File Updates

Tail can detect file growth through inotify or through periodic stat polling.
`Refresh_Interval` only controls path rescans for discovering new files; it does
not control how quickly an already-monitored file is read after growth.

For whole-file reread workloads, a configuration like this can be useful:

```ini
[INPUT]
    Name                    tail
    Path                    /path/to/eventlog/*
    Whole_File_On_Update    On
    Inotify_Watcher         False
    progress_check_interval 5
```

Disabling inotify makes Tail use periodic file-state checks instead of reacting
to every filesystem modify notification. With frequently-updated files, such as
Spark event logs, this can reduce the number of whole-file reread generations
started while the writer is actively appending.

Using `progress_check_interval 5` means Tail checks monitored files for growth
roughly every five seconds. Multiple rapid appends can be coalesced into one
larger reread generation instead of many immediate rereads. This reduces:

- repeated seeks to byte `0`
- repeated parsing and encoding of the same file contents
- S3 local buffer discard/recreate churn
- CPU and memory pressure from re-emitting the same file many times
- S3 upload attempts for transient intermediate file states

The trade-off is latency. Tail may wait up to the polling interval before it
observes new bytes in an already-monitored file. This is usually acceptable when
the destination should contain the latest full event-log snapshot rather than
every intermediate append.

## Database Behavior

The Tail database keeps the same schema and semantics.

The database still stores the current file offset after successful processing.
When Fluent Bit restarts, Tail restores that offset exactly as before. If the
file has not grown beyond the stored offset, no data is replayed. If the file
has grown while Fluent Bit was stopped, startup processing catches up from the
stored offset. Whole-file rereads are applied only after the file is already
being monitored in event mode.

This means the database remains responsible for:

- inode tracking
- persisted offset lookup
- stale file cleanup
- rotation-related file state
- restart recovery

The new option does not add checksums, hashes, timestamps, markers, or any new
database columns.

## Implementation Details

The Tail feature is centered around `flb_tail_file_set_pending_bytes()`.

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

When a whole-file reread starts, Tail increments a per-file generation counter
and marks emitted records with internal log-event metadata. S3 uses this metadata
to decide whether a pending local buffer belongs to an older generation of the
same source file.

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
- `whole_file_on_update_metadata`: verifies that Tail emits source and
  generation metadata for whole-file rereads.
- `db_whole_file_on_update`: verifies that the database offset is still restored
  after restart, and that whole-file forwarding only happens after the file
  grows beyond the stored offset.
- `db_whole_file_on_update_startup_offset`: verifies that startup catch-up after
  a restart reads only from the restored database offset, while later event-mode
  growth still forwards the whole file.
- `store_tail_source_generation_replace`: verifies that S3 store state appends
  chunks from the same Tail generation and can replace stale unlocked buffers for
  newer generations.
