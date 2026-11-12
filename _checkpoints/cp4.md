---
type: checkpoint
date: 2026-11-09T15:00:00+08:00
tentative: true
title: 'Checkpoint 4: foggy-llama'
gradescope_links:
    - name: code autograder
      url: https://www.gradescope.com/courses/1384476/assignments/9041316
due_event:
    type: due
    date: 2026-11-27T23:59:00+08:00
    description: 'Checkpoint #4 due (tentative)'
---

<style>
.cp4-handout {
  max-width: 60rem;
}
.cp4-handout table {
  width: 100%;
  border-collapse: collapse;
  margin: 1rem 0 1.5rem;
}
.cp4-handout th,
.cp4-handout td {
  border-bottom: 1px solid rgba(128, 128, 128, 0.3);
  padding: 0.55rem 0.65rem;
  text-align: left;
  vertical-align: top;
}
.cp4-handout th {
  background: rgba(128, 128, 128, 0.08);
}
.cp4-handout code {
  overflow-wrap: anywhere;
}
@media (max-width: 480px) {
  .cp4-handout th,
  .cp4-handout td {
    padding: 0.45rem 0.35rem;
  }
}
</style>

<div class="cp4-handout" markdown="1">

# Introduction

In this checkpoint, you will complete two tasks using **your own CP3 code**. If you have not finished CP3, complete it first. **We do not provide a sample solution.**

- Register the model routes for `foggy-llama`.
- Extend your HTTP server to handle routing errors and `OPTIONS` requests.

From the root of your completed CP3 checkout, pull the CP4 starter branch.

```sh
git pull --no-rebase origin foggy-llama
```

# foggy-llama

`foggy-llama` provides a command-line client, a model server, and handlers for health checks, model discovery, and chat completions. The client sends HTTP requests over FoggyTCP v2. The chat handler streams server-sent events (SSE) using your CP3 chunked response writer.

When running a real model, the server uses llama.cpp, a C/C++ library that runs language models locally from GGUF files. [llama.cpp repository](https://github.com/ggml-org/llama.cpp)

# Implementation

The two submitted `.cc` files, `foggy_http.cc` and `llama_routes.cc`, compile as **C11**. The supplied application framework compiles as C++11.

## 1. Register the model routes

Complete `foggy_llama_register_routes()` in `foggy-llama/src/llama_routes.cc`.

The declaration is in `foggy-llama/inc/foggy_llama.h`.

| Method | Path | Supplied handler |
| :--- | :--- | :--- |
| GET | `/health` | `foggy_llama_health` |
| GET | `/v1/models` | `foggy_llama_models` |
| POST | `/v1/chat/completions` | `foggy_llama_chat` |

- Use `foggy_http_server_get()` for GET routes and `foggy_http_server_post()` for POST routes.
- Pass the supplied `service` pointer as each handler's context.
- Return `0` on success or `-1` with `errno` set on failure.

## 2. Extend the HTTP server

### Route lookup

Update `foggy_http_server_listen()` in `foggy-http/src/foggy_http.cc`. After parsing the request and initializing the response, match the request method to the registered method and the full request target to the registered path. Both comparisons must be exact and case-sensitive. For example, `GET /health?x=1` returns HTTP 404 because the registered path is `/health`.

A path can have both GET and POST handlers. Call the matching handler with the original request, the initialized response, and its registered context. Keep the CP3 public HTTP headers and function signatures.

### Routing errors

Implement two routing errors in `foggy_http_server_listen()`.

- Return HTTP 404 when the full request target has no exact match among the registered paths.
- For a matching path, handle `OPTIONS` as described below. Return HTTP 405 with an `Allow` header for any other unsupported method.

For both errors, set `Content-Type: application/json` and send a JSON object with an `error` field. The field can contain any valid JSON value. A short message string is enough, and you can choose its wording.

| Example request | Status | JSON body | `Allow` |
| --- | --- | --- | --- |
| `GET /missing` | `404 Not Found` | `{"error":"Not Found"}` | |
| `POST /health` | `405 Method Not Allowed` | `{"error":"Method Not Allowed"}` | `GET, OPTIONS` |

Fill the initialized `foggy_http_response_t` and pass it to the CP3 code in `foggy_http_server_listen()` that serializes and sends a successful handler response. If you put that code in a helper such as `write_response()`, reuse that helper so the response is framed correctly on a persistent connection.

The supplied model handlers generate their own application error responses.

### OPTIONS requests

For `OPTIONS` on a registered path, return HTTP 200 with an empty body and an `Allow` header directly from the HTTP server. Use `Content-Length: 0` to delimit the empty response on a persistent HTTP/1.1 connection. You can omit `Content-Type` for this response. For `OPTIONS` on an unknown path, return the same JSON 404 as for other methods.

For both HTTP 200 responses to `OPTIONS` and HTTP 405 responses, list `GET` and `POST` if registered, in that order, then append `OPTIONS`. The possible `Allow` values are `GET, OPTIONS`, `POST, OPTIONS`, and `GET, POST, OPTIONS`. For example, `OPTIONS /health` returns `Allow: GET, OPTIONS`.

# Build and test

## Run with the mock engine

From the project root, build the client and server with `make cp4`. The first build initializes the submodules, including the new `thirdparty/llama.cpp` submodule, prepares the pinned library, and downloads the course GGUF model (about 105 MB).

On Ubuntu arm64, Make compiles the llama.cpp CPU backend from source to support the course Ubuntu 22.04 VMs. Allow a few minutes for the first build, depending on your VM and download speed. Subsequent builds reuse the compiled library.

Use `--mock` to test your routes and streaming responses with the deterministic backend.

```sh
make cp4
./build/foggy-server --mock
```

In another terminal, try the three routes.

```sh
./build/foggy-client health
./build/foggy-client models
./build/foggy-client chat --prompt 'Say hello.'
```

By default, the client connects to `localhost` on UDP port 8083. Use `--host` to connect to your server VM.

## Run the local checks

After completing both tasks and running `make cp4`, run the test script from the project root.

```sh
python3 foggy-llama/tests/smoke.py
```

The script starts its own `--mock` server on `127.0.0.1`, selects an available UDP port, and stops the server when the checks finish. It checks the three model routes, JSON routing errors, and `OPTIONS` responses with `Allow`.

## Run with a real model

Start the server on UDP port 8084 with the course model prepared by `make cp4`.

```sh
./build/foggy-server --model foggy-llama/models/SmolLM2-135M-Instruct-Q4_K_M.gguf --port 8084
```

In another terminal, send a prompt to the real model.

```sh
./build/foggy-client --port 8084 chat --prompt 'Say hello.'
```

# What to submit

Run `python3 foggy-llama/submit.py` from the project root and upload the resulting `foggy-llama/submit.zip` to the CP4 programming assignment on Gradescope. The ZIP contains `foggy_http.cc`, `llama_routes.cc`, and the generated `SHA256SUMS` file. Submit code only.

</div>
