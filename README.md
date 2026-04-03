# ProcessWatcher

`pwatch` is a Linux CLI that records as much process activity as user space can see without kernel code:

- `/proc` snapshots for the full process tree
- `strace` syscall logs, including network syscalls
- `lsof` snapshots for open files and sockets
- `ss` snapshots for live TCP and UDP connections
- optional `tcpdump` packet capture to a PCAP file

It also has an optional eBPF network mode:

- `--ebpf-network` starts a `bpftrace` program in the kernel
- it records high-frequency socket syscalls such as `socket`, `bind`, `connect`, `accept4`, `send*`, `recv*`, and `close`
- those events are filtered back to the watched PID tree in user space and written as `ebpf_network` records in `events.jsonl`
- this requires `root`
- it works best for longer-lived processes or when you attach to an already-running target

## Usage

Show built-in help:

```bash
./pwatch -h
./pwatch run -h
./pwatch attach -h
./pwatch net -h
./pwatch net-run -h
```

Simple network-focused usage:

```bash
./pwatch net 12345
sudo ./pwatch net 12345
./pwatch net-run -- curl https://example.com
sudo ./pwatch net-run -- curl https://example.com
```

Persist capture artifacts:

```bash
./pwatch --save --duration 60 attach 12345
```

Enable the eBPF network backend:

```bash
sudo ./pwatch --save --ebpf-network --duration 60 attach 12345
```

Capture packet-level metadata for DNS and TLS hostname extraction:

```bash
sudo ./pwatch --save --pcap --duration 60 run -- curl -I https://example.com
```

`net` and `net-run` are the simple front door. They automatically save data, and when run as `root` they also turn on packet capture and the eBPF network backend.

Write the live event stream and final summary to explicit file paths:

```bash
./pwatch \
  --save \
  --events-file ./agent-events.jsonl \
  --summary-file ./agent-summary.json \
  --duration 60 \
  attach 12345
```

Run a new command under observation:

```bash
./pwatch --save run -- curl https://example.com
```

Attach to an existing process:

```bash
./pwatch attach 12345
```

If you control how the target starts, prefer `run` over `attach`. It captures earlier activity and avoids common Linux `ptrace` restrictions.

While running, `pwatch` prints live status lines to stderr and writes the full structured capture to files.

Write to a specific output directory and stop after 30 seconds:

```bash
./pwatch --save --output ./captures/app --duration 30 attach 12345
```

Include environment variables and host-level packet capture:

```bash
sudo ./pwatch --save --include-env --pcap --pcap-filter "tcp port 443" attach 12345
```

## Output

When `--save` is used, each run creates a directory containing:

- `metadata.json`
- `summary.json`
- `events.jsonl`
- `endpoint_summary.json`
- `dns_summary.json`
- `tls_sni_summary.json`
- `strace*`
- `traffic.pcap` when `--pcap` is enabled
- `network_report.json`
- `network_report.md`

`events.jsonl` contains structured snapshots with process details, open file descriptors, `lsof` results, and `ss` connection state.

`endpoint_summary.json` rolls up where observed connections went, including:

- remote IP and port
- reverse DNS names when resolution succeeds
- DNS names learned from captured DNS responses when `--pcap` is enabled
- TLS SNI hostnames learned from captured TLS handshakes when `--pcap` is enabled

`network_report.json` and `network_report.md` are the simplest destination-focused outputs for humans and AIs.

## Agent-Friendly Usage

Agents can run the command directly the same way a human would:

```bash
./pwatch --save --duration 15 attach 12345
```

With `--save`, the command prints the capture directory path on stdout when it finishes, so an agent can open that directory and inspect the JSON and trace files.

## Limits

- TLS payloads are still encrypted unless you instrument the target app separately.
- `tcpdump` captures host traffic, not only one PID, so use `--pcap-filter` to narrow it.
- `strace` attach can be blocked by Linux ptrace restrictions or permissions.
- `run` launches the target under `strace`, so it captures earlier process activity than `attach`.
- `--ebpf-network` improves syscall-level network visibility, but it still does not give decrypted HTTPS bodies or full application-layer request semantics.
- `--ebpf-network` PID-tree filtering is best-effort for very short-lived child processes, because those tasks can exit before user-space correlation finishes.
