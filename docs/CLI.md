# CLI

**Binary name in all archives:** `relog-net` (Windows: `relog-net.exe`)

## Modes

- **`run`** — run a single `.rl` file locally until quiescence or limits.
- **`serve`** — start an HTTP + WebSocket API to run/control `.rl` files in a base directory (pass a directory **or** a file; if a file is given, its parent becomes the base).

---

## `run` command

`relog-net run <path-to-file.rl> [options]`

| Arg / Flag          | Description                                | Default               |
| ------------------- | ------------------------------------------ | --------------------- |
| `<path-to-file.rl>` | DSL file to execute (must be a file)       | —                     |
| `--log <level>`     | `off`\|`error`\|`warn`\|`info`\|`debug`    | `info`                |
| `--mode <mode>`     | Override runtime: `natural`\|`reactive`    | from DSL or `natural` |
| `--max_ticks <N>`   | Stop after N ticks                         | from DSL or unlimited |
| `--delay <MS>`      | Inter-step delay in **reactive** mode (ms) | from DSL or `0`       |

**Notes:** CLI flags override the corresponding values defined in the `.rl` file (except `--log`, which is CLI-only).
`--delay` applies only to `reactive`; `natural` ignores it.

**Examples**

```
relog-net run ./samples/hello.rl --log debug --max_ticks 1000
```

---

## `serve` command

`relog-net serve <base-path-or-file> [options]`

| Arg / Flag                  | Description                                                   | Default |
| --------------------------- | ------------------------------------------------------------- | ------- |
| `<base-path-or-file>`       | Directory to serve, or a file inside it (parent becomes base) | —       |
| `--port <P>`                | HTTP port                                                     | `8080`  |
| `--autostart <rel-file.rl>` | Autostart this file **relative to base**                      | —       |
| `--log <level>`             | Log level                                                     | `info`  |

**Examples**

```
relog-net serve ./samples --port 9000 --autostart demo.rl --log debug
```

---

## Serve-mode API

Base: `http://HOST:PORT/api` • WS: `ws://HOST:PORT/api`

| Method | Path                   | Body (JSON)                              | Returns                                                                      | Notes                                                                                                    |
| ------ | ---------------------- | ---------------------------------------- | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| WS     | `/api`                 | —                                        | Stream of `{schema,type,payload}` where `type` = `status`\|`event`\|`error`. | Realtime status/events/errors.                                                                           |
| POST   | `/api/start`           | `{file, paused?, delay_ms?, max_ticks?}` | `200` or `400`                                                               | `file` is **relative to base**; optional immediate pause; applies `delay_ms` if provided (**reactive**). |
| POST   | `/api/stop`            | —                                        | `200`                                                                        | Stops current run.                                                                                       |
| POST   | `/api/pause`           | —                                        | `200` or `400`                                                               | Pauses run.                                                                                              |
| POST   | `/api/resume`          | —                                        | `200` or `400`                                                               | Resumes run.                                                                                             |
| POST   | `/api/step`            | —                                        | `200` or `400`                                                               | Executes a single manual step.                                                                           |
| POST   | `/api/set_delay`       | `{delay_ms}`                             | `200` or `400`                                                               | Sets inter-step delay (**reactive**).                                                                    |
| GET    | `/api/status`          | —                                        | `200`, `{ "running": true \| false }`                                        | Current run state.                                                                                       |
| GET    | `/api/file?path=<rel>` | —                                        | `200` (text) or `404/500`                                                    | Reads a `.rl` file inside base.                                                                          |
| GET    | `/api/files`           | —                                        | `200`, `["foo.rl","bar.rl"]`                                                 | Lists `.rl` files in base.                                                                               |
| POST   | `/api/save`            | `{path, content}`                        | `200` or `403/500`                                                           | Saves a `.rl` file **inside base**.                                                                      |
| GET    | `/api/net?path=<rel>`  | —                                        | `200` (NetStateDto) or `400/500`                                             | Uses provided `path` or the current run if omitted.                                                      |

**I/O note:** The API operates on `.rl` files. All file paths must be inside the base directory (relative to base).
