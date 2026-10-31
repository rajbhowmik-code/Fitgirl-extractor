# Extraction Guardian

Crash-safe, resumable extraction for huge game repacks (FitGirl-style).

If the power goes out or the PC crashes mid-extraction, Guardian continues from the last **finished unit** instead of restarting from 0%.

## How it works

Guardian is a C++ wrapper around your existing extractor. It does not decompress anything itself.

For each archive unit (for example `fg-1.bin`, `fg-2.bin`, ...), in natural order:

1. Writes `UNIT_START` to a crash-safe journal (each line is checksummed and flushed to disk).
2. Runs the extractor with output going to `<game-folder>/.staging/<unit>/`.
3. If the extractor exits with code 0, builds a manifest (path, size, CRC32) of the staged files and verifies it.
4. Moves the verified files into the real game folder.
5. Removes the staging folder and writes `UNIT_DONE`.

After a crash, units marked `UNIT_DONE` are skipped. A unit that has `UNIT_START` but no `UNIT_DONE` is redone from scratch, after its old staging folder is deleted.

## Requirements

- Windows (Linux is a possible future goal)
- C++17 compiler and CMake
- Your repack's extractor (for example `setup.exe`)

## Build

```
cmake -B build
cmake --build build
ctest --test-dir build
```

## Configuration: `guardian.ini`

Place `guardian.ini` next to the executable:

```ini
extractor_cmd = "setup.exe /unit {unit} /out {out}"
```

| Placeholder | Replaced with |
|-------------|---------------|
| `{unit}`    | Full path of the current archive unit |
| `{out}`     | Staging folder for that unit (`<game-folder>/.staging/<unit>/`) |

Paths containing spaces are handled. Adjust the command to match your extractor's real arguments.

Archive pieces are matched by a file pattern from the config file; the default is `*.bin`.

## Usage

Start (or resume) extraction:

```
guardian resume <game-folder>
```

Run it again after any crash and it continues where it stopped. If everything is already finished it prints `all units done` and does nothing.

Auto-resume after a reboot:

```
guardian install-autostart     # creates a Task Scheduler task (runs at logon)
guardian remove-autostart      # deletes it
```

The task is also removed automatically when all units are done.

Only one Guardian can run per game folder at a time. A second copy exits with a clear message.

Example status output:

```
Unit 7/12 | 41.2 GB skipped | resumed
```

## Error handling

- **Extractor exit code not 0:** retried up to 2 times, then Guardian stops with a clear message.
- **Disk full:** Guardian stops with a clear message. Free space and run `guardian resume` again.
- **File locked (for example by antivirus):** the move is retried after a short wait.

## Testing

- `fake_extractor` is a small test program that writes deterministic files and can be killed at any moment.
- Debug flag `--crash-after N` exits immediately (no cleanup) after N seconds, to simulate a power cut.
- A chaos script repeatedly kills Guardian at random times, then runs it to completion and compares the output against the expected manifest.

## Limits

- **No mid-unit resume.** If a crash happens during a unit, that unit restarts from its beginning. Guardian never resumes inside a solid block, because that would require decompressor-level knowledge.
- Work lost on a crash is at most one unit. Smaller units mean less lost work.
- Guardian can only be as correct as the extractor it wraps. It verifies that staged files are complete and consistent, not that the extractor's output is semantically correct.

## Project layout

```
src/        guardian source (crc32, journal, units, state, process, main)
tests/      assert-based tests (run with ctest)
```
