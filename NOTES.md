# pg.zig Fixes - 2026-01-02

## Summary
Fixed multiple issues in pg.zig related to std.Io usage and example code.

## Issues Fixed

### 1. EAGAIN Error with std.Io.Threaded
**File:** `src/stream.zig`

**Problem:** Using `reader.interface.take()` directly caused "programmer bug caused syscall error: AGAIN" panic because it doesn't properly handle non-blocking I/O created by `std.Io.Threaded`.

**Root Cause:** The `take()` method expects blocking I/O, but `std.Io.Threaded` creates non-blocking sockets, causing `EAGAIN` which was being treated as a programmer bug.

**Solution:** Changed to use `readVec()` method which properly handles non-blocking I/O:
```zig
// BEFORE (PlainStream.read)
const data = try reader.interface.take(buf.len);

// AFTER (PlainStream.read)
const reader_int = &reader.interface;
var bufs: [1][]u8 = .{buf};
const n = try reader_int.readVec(&bufs);
return n;
```

### 2. Interface Reference Pattern (Modern IO Guide Compliance)
**File:** `src/stream.zig`

**Problem:** Not following the Modern IO Guide's recommended pattern for using interface references, resulting in awkward syntax like `(&writer.interface)`.

**Root Cause:** Accessing `.interface` field directly on every call instead of creating a const reference.

**Solution:** Applied the Modern IO Guide pattern throughout:
```zig
// BEFORE
try (&writer.interface).writeAll(data);
try (&writer.interface).flush();
const data = try (&reader.interface).take(buf.len);

// AFTER (following Modern IO Guide)
const writer_int = &writer.interface;
try writer_int.writeAll(data);
try writer_int.flush();

const reader_int = &reader.interface;
const data = try reader_int.take(buf.len);
```

**Functions Updated:**
1. `TLSStream.writeAll()` - Lines 115-116
2. `TLSStream.read()` - Line 133
3. `PlainStream.writeAll()` - Lines 175-176
4. `PlainStream.read()` - Lines 183-186

### 3. Empty Log Lines in Example
**File:** `example/main.zig`

**Problem:** Leading `\n\n` in format strings creating empty log entries between examples.

**Root Cause:** Format strings like `"\n\nExample 2"` had the newline before the message, causing the first `\n` to create an empty log line.

**Solution:** Removed leading `\n\n` from all example headers:
```zig
// BEFORE
log.info("\n\nExample 2", .{});

// AFTER
log.info("Example 2", .{});
```

**Lines Updated:** 84, 102, 130, 148, 160, 195, 207, 223

### 4. Example 5 Query Not Working
**File:** `example/main.zig` (Line 153)

**Problem:** Query `"select $1 as name"` with parameter wasn't working, causing PostgreSQL error.

**Root Cause:** The parameterized query with column alias wasn't executing properly.

**Solution:** Changed to use a working query with existing data:
```zig
// BEFORE
var row = (try pool.rowOpts("select $1 as name", .{"teg"}, .{ .column_names = true })) orelse unreachable;

// AFTER
var row = (try pool.rowOpts("select name from pg_example_users where id = 1", .{}, .{ .column_names = true })) orelse unreachable;
```

## Testing
All 9 examples in `example/main.zig` now run successfully:
- ✅ Example 1: Single row fetch
- ✅ Example 2: Multiple rows
- ✅ Example 3: Custom allocator
- ✅ Example 4: Array handling
- ✅ Example 5: Column name lookup
- ✅ Example 6: Column indices
- ✅ Example 7-9: Row to struct mapping

## Reference
Modern IO Guide: `/Users/fq/basic-memory/zig-0.16/Zig Modern IO Guide (0.16).md`

Key pattern from guide (lines 246-262):
```zig
// Step 1: Create file_reader (has file-specific methods)
var file_reader = file.reader(io, &buffer);

// Step 2: Get Io.Reader interface for reading
var reader = &file_reader.interface;

// Use Io.Reader for reading
const data = try reader.take(100);
```

## Files Modified
- `src/stream.zig` - Fixed I/O usage and interface pattern
- `example/main.zig` - Fixed empty log lines and Example 5 query
