# httpxx

A C++20 HTTP/1.1 server library with routing, static file serving, and templated response rendering.

---

## Overview

httpxx is a framework for building HTTP/1.1 servers in C++20. It exposes a declarative routing API where paths and methods are bound to handler functions. Handlers receive a parsed request object and return a response — plain text, HTML, JSON, or template-rendered HTML.

The repository is split into two parts: the `lib/v2/httpxx` library, which is the active implementation, and an `example/` application that demonstrates the full request lifecycle against a real configuration and static content directory. A deprecated predecessor exists under `lib/deprecated/` and is not the active API target.

---

## Architecture

Incoming connections are accepted by the socket layer, which constructs a `Request` object from the raw HTTP stream. The `Server` dispatches that object through the `Router`, which matches against registered `Endpoint` definitions by path and method. The matched handler executes, produces a `Response`, and the socket layer serializes it back to the client.

```
client
  └── socket (network I/O)
        └── Request (parsed HTTP model)
              └── Router (path + method dispatch)
                    └── Endpoint / Handler
                          ├── static files (filesystem)
                          ├── JSON (nlohmann_json)
                          └── templates (inja)
```

Configuration is a startup-time concern. The `Server` is constructed with a `Router` and a `Configuration` loaded from `config.toml`. There are no per-request configuration dependencies.

---

## Dependencies

**External** — must be present on the build system:

| Dependency | Purpose |
|---|---|
| `nlohmann_json` | JSON response serialization |
| `fmt` | String formatting |

**Vendored** — included in `lib/v2/include/`:

| Dependency | Purpose |
|---|---|
| `inja` | Server-side HTML template rendering |
| `tomlpp` | TOML configuration parsing |

---

## Building

httpxx uses [Meson](https://mesonbuild.com). The library and example application are separate build targets.

```sh
meson setup build
cd build
ninja
```

To build only the library:

```sh
meson setup build --buildtype=release
ninja -C build httpxx
```

Requires a C++20-capable compiler and the external dependencies listed above to be findable by the build system.

---

## Configuration

The example application reads its runtime settings from `example/config.toml`. At minimum this controls the server address, port, and the path from which static files are served.

```toml
# example/config.toml
host = "127.0.0.1"
port = 8080
www_path = "example/static"
```

`Configuration` is parsed at startup via the vendored `tomlpp` and passed directly to the `Server` constructor. It is not reloaded at runtime.

---

## Usage

```cpp
#include <httpxx/server.hh>
#include <httpxx/router.hh>
#include <httpxx/configuration.hh>

int main() {
    auto config = httpxx::Configuration::from_file("config.toml");
    httpxx::Router router;

    router.get("/", [](const httpxx::Request& req) {
        return httpxx::Response::html("<h1>hello</h1>");
    });

    router.get("/api/status", [](const httpxx::Request& req) {
        return httpxx::Response::json({ {"status", "ok"} });
    });

    router.post("/submit", [](const httpxx::Request& req) {
        // handle request body
        return httpxx::Response::text("received");
    });

    httpxx::Server server(router, config);
    server.run();
}
```

Handlers return typed responses. The library supports four response shapes:

| Factory | Content-Type |
|---|---|
| `Response::text(...)` | `text/plain` |
| `Response::html(...)` | `text/html` |
| `Response::json(...)` | `application/json` |
| `Response::render(template, data)` | `text/html` via inja |

Static files are served by registering a route that reads from the configured `www_path`. The example application shows this pattern in full.

---

## Project Structure

```
httpxx/
├── lib/
│   ├── v2/
│   │   ├── httpxx/          # active library headers and implementation
│   │   │   ├── server.hh
│   │   │   ├── router.hh
│   │   │   ├── endpoint.hh
│   │   │   ├── request_handlers.hh
│   │   │   ├── configuration.hh
│   │   │   ├── objects.hh
│   │   │   ├── enums.hh
│   │   │   ├── socket.hh
│   │   │   ├── socket_enums.hh
│   │   │   └── httpxx_assert.hh
│   │   ├── include/         # vendored dependencies (inja, tomlpp)
│   │   └── meson.build
│   └── deprecated/          # previous architecture, not the active API
├── example/
│   ├── main.cc              # example application entry point
│   ├── config.toml          # runtime configuration
│   └── static/              # static assets served by the example app
└── meson.build
```

The `lib/deprecated/` tree is retained for reference. It has its own internal layout but is not compiled as part of the primary build target.

---

## License

See `LICENSE`.
