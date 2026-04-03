# ProcessWatcher

`pwatch` is a Linux CLI that records as much process activity as user space can see without kernel code:

- `/proc` snapshots for the full process tree
- `strace` syscall logs, including network syscalls
- `lsof` snapshots for open files and sockets
- `ss` snapshots for live TCP and UDP connections
- optional `tcpdump` packet capture to a PCAP file

## Usage

Show built-in help:

```bash
./pwatch -h
./pwatch run -h
./pwatch attach -h
```

Write the live event stream and final summary to explicit file paths:

```bash
./pwatch \
  --events-file ./agent-events.jsonl \
  --summary-file ./agent-summary.json \
  --duration 60 \
  attach 12345
```

Run a new command under observation:

```bash
./pwatch run -- curl https://example.com
```

Attach to an existing process:

```bash
./pwatch attach 12345
```

If you control how the target starts, prefer `run` over `attach`. It captures earlier activity and avoids common Linux `ptrace` restrictions.

Write to a specific output directory and stop after 30 seconds:

```bash
./pwatch --output ./captures/app --duration 30 attach 12345
```

Include environment variables and host-level packet capture:

```bash
sudo ./pwatch --include-env --pcap --pcap-filter "tcp port 443" attach 12345
```

## Output

Each run creates a directory containing:

- `metadata.json`
- `summary.json`
- `events.jsonl`
- `strace*`
- `traffic.pcap` when `--pcap` is enabled

`events.jsonl` contains structured snapshots with process details, open file descriptors, `lsof` results, and `ss` connection state.

## Agent-Friendly Usage

Agents can run the command directly the same way a human would:

```bash
./pwatch --duration 15 attach 12345
```

The command prints the capture directory path on stdout when it finishes, so an agent can open that directory and inspect the JSON and trace files.

## Limits

- TLS payloads are still encrypted unless you instrument the target app separately.
- `tcpdump` captures host traffic, not only one PID, so use `--pcap-filter` to narrow it.
- `strace` attach can be blocked by Linux ptrace restrictions or permissions.
- `run` launches the target under `strace`, so it captures earlier process activity than `attach`.
