# Relog Net (Preview)

**Toy, no-I/O runner** for a tiny DSL for **algebraic token networks**.

No persistence, no files, no sockets — just load a .rl and run it locally.

## Quick Start

Grab the archive for your OS from [latest release](https://github.com/relogrun/relog/releases/latest):

- **macOS (Universal)** — `relog-macos-universal.zip`
- **Linux x86_64 (musl)** — `relog-linux-x86_64.tar.gz`
- **Windows x86_64 (gnu)** — `relog-windows-x86_64.zip`

### macOS

```bash
unzip relog-macos-universal.zip
cd relog-*
chmod +x ./relog-net

# run a sample
./relog-net ./samples/buffer_backpressure.rl --log debug --delay 500
```

> If macOS shows a quarantine warning:
> `xattr -dr com.apple.quarantine ./relog-net`

### Linux (x86_64)

```bash
tar -xzf relog-linux-x86_64.tar.gz
cd relog-*
chmod +x ./relog-net

# run a sample
./relog-net ./samples/buffer_backpressure.rl --log debug --delay 500
```

### Windows (x86_64)

```powershell
Expand-Archive .\relog-windows-x86_64.zip -DestinationPath .\relog
cd .\relog

# run a sample
.\relog-net.exe .\samples\buffer_backpressure.rl --log debug --delay 500
```

> If SmartScreen warns about an unknown publisher, choose “More info” → “Run anyway”.

## Run via Docker (read-only sandbox)

Don’t want to run a local binary? Use a locked-down Linux container.

<details>
<summary>Details</summary>

### macOS / Linux

```bash
docker run --rm --platform=linux/amd64 \
  --read-only --cap-drop=ALL --pids-limit=256 \
  --memory=512m --cpus=1 --network none \
  --security-opt no-new-privileges:true \
  --tmpfs /tmp:rw,nosuid,nodev \
  -v "$PWD:/app:ro" \
  -w /app \
  -u "$(id -u):$(id -g)" \
  debian:stable-slim \
  /app/relog-net /app/samples/buffer_backpressure.rl --log debug --delay 500
```

### Windows (PowerShell)

```powershell
docker run --rm --platform=linux/amd64 `
  --read-only --cap-drop=ALL --pids-limit=256 `
  --memory=512m --cpus=1 --network none `
  --security-opt no-new-privileges:true `
  --tmpfs /tmp:rw,nosuid,nodev `
  -v "${PWD}:/app:ro" `
  -w /app `
  debian:stable-slim `
  /app/relog-net /app/samples/buffer_backpressure.rl --log debug --delay 500
```

> Notes:
>
> - The command expects you extracted the **Linux** archive into the current directory, so `relog-net` and `samples/` are present under `./`.
> - `--platform=linux/amd64` makes it work on Apple Silicon too.
> - The container is read-only, has dropped capabilities, no network, a tmpfs `/tmp`, and (on macOS/Linux) runs as your user via `-u`.

</details>

---

## Usage

There are two modes:

**Direct mode (`run`)** — execute a single `.rl` locally.

```bash
./relog-net run ./samples/buffer_backpressure.rl --log debug --delay 500
```

**Server mode (`serve`)** — start an HTTP + WebSocket API for remote control (pass a base directory or a file inside it).

```bash
./relog-net serve ./samples --port 9000 --log debug
```

Help:

```bash
./relog-net run --help
./relog-net serve --help
```

For full details (all flags, API endpoints) see [docs/CLI.md](./docs/CLI.md).

---

## DSL (tiny overview)

A program declares **stores**, **transitions** and **initial tokens**:

```relog
// Stores
store produced
store free
store buf
store done

// Move one produced item into the buffer only if guard passes.
transition push {
  in  produced(item(let x))
  in  free(slot)
  out buf(item(let x))
  guard allowed(let x)
}

transition pop_pair {
  in  buf(item(let y)) * 2
  out done(pair(let y, let y))
  out free(slot) * 2
}

init {
  produced item(hello)
  produced item(world) * 2
  free slot * 2
}
```

Full DSL reference: [docs/DSL.md](./docs/DSL.md)

Samples: [samples/](./samples)

---

## License

Free for Non-Commercial Use. Commercial use requires a license — [LICENSE.md](./LICENSE.md).
