# tscript

`tscript` is a lightweight, dependency‑free alternative to `script(1)` focused on **clean, per‑line timestamped logging** of interactive terminal sessions (e.g. router / network equipment output). It preserves an interactive shell while writing a structured log suitable for audits, troubleshooting and archival.

## Key Features
- **Newline‑centric design:** Optimized for devices and applications that emit discrete, newline‑terminated status / log lines (routers, switches, network daemons). Full-screen / cursor-moving TUIs will still work interactively but their visual dynamics (incremental redraw regions, progress spinners) are intentionally normalized into finalized newline records only.
- ISO‑8601 timestamps with milliseconds and timezone offset
- Custom timestamp format with milliseconds, timezone offset or Zulu UTC
- Optional ANSI/VT100 stripping (only in the log)
- Append mode for continuous session accumulation
- Run a one‑off command or start a login shell
- Terminal size sync + live resize propagation (SIGWINCH)
- Signal forwarding (`SIGINT`, `SIGTERM`, `SIGHUP`) to the child session
- Clean shutdown with accurate child exit status propagation

## Installation
Copy the script to a directory on your `PATH` and make it executable:
```bash
cp bin/tscript /usr/local/bin/tscript
chmod +x /usr/local/bin/tscript
```
No external Python packages are required. Python ≥ 3.8 recommended.

## Usage
```bash
tscript [options] [outfile]
```
If `outfile` is omitted a filename like `session-2025-09-04_145512.log` is generated.

### Common Examples
```bash
# Start an interactive login shell, auto‑named log
tscript

# Log to a specific file
tscript router-a.log

# Append to an existing file (do not overwrite)
tscript -a router-a.log

# Force UTC timestamps
tscript --utc core-switch.log

# Custom timestamp format
tscript --ts-format "%F %T.{ms3} {tz}" switch.log

# Run a single command then exit
tscript --cmd "show interfaces status" iface.log

# Strip colors / escape sequences from the saved log only
tscript --no-color clean.log
```

## Options
- `-a, --append` : Append instead of truncate the log file
- `--utc` : Use UTC (timestamp ends with `Z`)
- `--no-color` : Strip ANSI / VT100 escape sequences in the log output
- `--ts-format <FORMAT>` : Custom `strftime` format with extras `{ms3}` and `{tz}` (default `%Y-%m-%dT%H:%M:%S.{ms3}{tz}`)
- `--cmd <STRING>` : Run command via your shell (`$SHELL -lc <STRING>`) instead of spawning a login shell
- `outfile` : Destination log file (optional)

## Timestamp Formatting
Placeholders:
- `{ms3}` → zero‑padded milliseconds (000‑999)
- `{tz}`  → RFC3339 timezone offset (`+HH:MM`) or `Z` when `--utc` is used
- Can be mixed with normal `strftime` tokens.

Example:
```bash
tscript --ts-format "%F %T.{ms3} {tz}" demo.log
# 2025-09-04 14:23:19.137 +02:00 Interface Gi0/1 up
```

## Color / ANSI Stripping
`--no-color` applies a conservative regex pass that removes:
- CSI sequences (ESC [ ...)
- OSC sequences (ESC ] ... BEL/ST)
- Short single ESC forms
The live terminal still shows full color; only the saved log is cleaned.

## How It Works
1. Allocates a PTY pair (`master`, `slave`).
2. Forks. Child makes the slave its controlling TTY (using `os.login_tty` if available or a manual `setsid` + `ioctl(TIOCSCTTY)` fallback) and execs a shell (login or `--cmd`).
3. Parent places the user terminal in raw mode, relays input → child and child output → user + log.
4. Output is buffered until a `\n` arrives; then a timestamped line is written.
5. On exit (EOF or signals) the remaining partial line (if any) is flushed once with a timestamp.

## Window Resize Handling and Signal Forwarding
`SIGWINCH` in the parent triggers ioctl to copy the current terminal size onto the PTY and signals the child process group so full‑screen tools (vim, less, etc.) redraw correctly.
`SIGINT`, `SIGTERM`, and `SIGHUP` received by the parent are forwarded to the child process group, mirroring direct interactive behavior. The parent then continues relaying until the PTY closes (EOF / EIO) so final output is not lost.

## Exit Codes
- If the child shell / command exits normally, `tscript` exits with the same code.
- If terminated by signal `N`, `tscript` exits with `128+N` (conventional shell practice).

### When To Prefer `tscript`
Use `tscript` when you need to:
- Correlate multiple parallel sessions (e.g. capture logs from several routers while reproducing a failure) and later diff or merge them based on precise millisecond timestamps.
- Compare command execution ordering across terminals to diagnose race conditions or timing-sensitive behaviors.
- Track operational procedures where each command/result pair must be auditable.

If you need pixel/character‑accurate replay of an interactive TUI’s evolution, or want to preserve transient carriage-return updates (progress bars), stick with classic `script`.

## Roadmap / Ideas
- `--raw-copy` second file (exact byte stream)
- Timing capture for replay
- Improved partial line flush policies (idle timeout, immediate flush mode)
- Integrity footer (hash of log)
- Timing file for future replay capability

## Security Considerations
- Timestamps are generated locally; ensure system clock / timezone is correct. Start all sessions on one device to be sure timestamps are in sync
- Passwords typed at a silent prompt are not logged (terminal suppresses echo). If a device echoes secrets they will be captured.
- Protect log file permissions (e.g. adjust `umask`).

## Performance
Efficient for typical network device output. Each read chunk → one terminal write + timestamp parsing of complete lines.

## Contributing / Local Tweaks
The utility is a single Python file; modify directly as needed. Keep patches focused to retain audit clarity.

---
