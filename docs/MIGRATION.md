# Migration and downgrade

The on-disk facts for moving between LibreDB versions. Durability of a single
file is in [`RELIABILITY.md`](./RELIABILITY.md). This page is where a search
for "downgrade" should land.

## v0.1.x to v0.2.0

New databases begin with an 8-byte `LRDB` magic and version header. Each
record header also checksums its own length field.

Files written by v0.1.x are headerless. `open()` still reads them through a
legacy path, and later appends on that file keep the legacy record framing.
Nothing rewrites an old file into the v1 layout just because a newer LibreDB
opened it.

The header is what lets `open()` refuse a file that is not a LibreDB database
(`NOT_A_DATABASE`) and leave it byte-for-byte untouched. The length checksum
is what lets recovery refuse a damaged length field instead of treating it as
a torn tail.

## Downgrade warning

A file written by 0.2.0 or newer must never be opened by 0.1.3 or older.

The old recovery cannot parse the header. It classifies the whole file as a
torn tail and silently truncates it to zero bytes. Back up before any
downgrade. A file copy taken while no writer has the database open is the
byte-exact backup; see [Backup and restore](./CLI.md#backup-and-restore).

## Legacy behavior that changed in 0.2.0

- A headerless file whose only record is torn or incomplete now refuses to
  open as `NOT_A_DATABASE`. 0.1.3 recovered that file as an empty database.
  Refusing is the safe reading: such a file is indistinguishable from a
  foreign one.
- Any file shorter than the 8-byte header is refused untouched. A crash inside
  the first bytes of a brand-new database's first commit therefore needs a
  manual delete. Nothing in that file was acknowledged.
- A damaged length field in a headerless v0.1.x file still reads as a torn
  tail. The legacy format has no header checksum. The v1 format exists to
  close that gap.

## Converting a legacy file to v1

Opening a headerless file does not upgrade it. To get a v1 file on purpose,
copy the data into a database created by 0.2.0 or newer:

- `libredb export` writes the key-value layer as JSON, and `libredb import`
  into a fresh path writes a new file. That dump is logical, not byte-exact:
  it carries the keys `import` can write back, not the catalog or the log.
- Or read through one `open()` and write through another into a new path.

A non-mutating diagnosis command is tracked in
[#51](https://github.com/libredb/libredb-database/issues/51). There is no
in-place upgrade flag today.

## Compatibility going forward

v1 files stay readable by later 0.2.x releases. A newer on-disk version would
be a new header version, called out in the changelog the same way 0.2.0 was,
with the same rule: do not open a newer file with an older release that
cannot parse its header. The bar for calling the format a production store is
tracked separately in
[#58](https://github.com/libredb/libredb-database/issues/58).
