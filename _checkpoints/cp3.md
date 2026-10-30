---
type: checkpoint
date: 2026-10-30T15:00:00+08:00
tentative: false
title: 'Checkpoint 3: HTTP over FoggyTCP'
due_event:
    type: due
    date: 2026-11-15T23:59:00+08:00
    description: 'Checkpoint #3 due'
attachment: /static_files/checkpoints/cp3/cp3-student.zip
---

# Introduction

In this checkpoint, you will implement HTTP over FoggyTCP with:

- HTTP/1.0 and HTTP/1.1 requests and responses for a date query service and an echo service,
- Persistent connections (keep-alive) and idle timeouts,
- Chunked responses that deliver output incrementally.

The starter code provides basic date query and echo services that exchange raw data over FoggyTCP. You will extend it to support the HTTP features above, and then expose both services through NGINX as a reverse proxy on the default HTTP port 80.


# Starter code

Get the starter code from the [foggy-http repository](https://github.com/HKUST-Network/foggy-http). This checkpoint uses the newer **FoggyTCP v2** transport, supplied as a submodule. **Do not use the FoggyTCP code or your implementation from CP2.**

```sh
git clone --recurse-submodules https://github.com/HKUST-Network/foggy-http.git
cd foggy-http
git submodule update --init --recursive
```

### `date-client.cc` and `date-server.cc`

These files in `foggy-http/src/` implement the date query service. The client sends GET `/checkpoint-3/date`; the server listens on UDP 8081 and returns the current date as `YYYY-MM-DD\n`.

### `echo-client.cc` and `echo-server.cc`

These files in `foggy-http/src/` implement the echo service. The client sends POST `/checkpoint-3/echo`; the server listens on UDP 8082 and returns the request body. The server's `--chunked` option enables a streaming response producer.

### `foggy_http.cc` and `foggy_http.h`

`foggy-http/src/foggy_http.cc` contains the HTTP library you need to complete: message serialization and parsing, connection management, and chunked transfer. The public C API is declared in `foggy-http/inc/foggy_http.h`; keep its signatures unchanged. 

### `transport.cc`

`foggy-http/src/transport.cc` adapts the supplied FoggyTCP transport to the HTTP library's C API, providing byte reads and writes, timed reads, and connection closing. Keep this file unchanged.

### `nginx-stream.conf`

`foggy-http/conf/nginx-stream.conf` is a complete NGINX configuration template. It loads the required modules and contains the `events` and `stream` blocks. Complete the two `TODO` path-to-backend mappings so that requests arriving on UDP 80 reach the date or echo server.


# Run the starter code

Use your client and server VMs from CP2, with the `foggy-http` repository available in both. The commands below assume the project root (the folder containing the top-level `Makefile`) is `/vagrant/foggy-http`; adjust the path if you put the repository elsewhere.

In each VM, install the build dependencies once, then build the applications and the NGINX extension:

```bash
cd /vagrant/foggy-http
sudo bash lab/setup.sh
make -j2
```

The setup script installs NGINX, its standard Stream module, and the required compilers and development headers. Rerun `make -j2` as an ordinary user whenever you change your code. The build also produces the NGINX extension at `build/ngx_stream_foggy_http_module.so`.

The starter code **does not support keep-alive**. It exchanges raw application data, and the server closes the connection after every response.

Both clients connect to UDP 80, the NGINX entry point, by default. You will use it after implementing HTTP and proxy routing. For the initial tests, **connect directly to the backend** with `--port 8081` for the date service or `--port 8082` for the echo service.

In the server VM, from `/vagrant/foggy-http`, start the echo server:

```bash
./build/echo-server
```

In the client VM, from `/vagrant/foggy-http`, run the echo client:

```bash
./build/echo-client 10.0.1.1 hello --port 8082 --no-keep-alive
```

The client sends `hello` to the echo server and should print the same message back.

To try the date query service, start the date server in another server VM terminal from `/vagrant/foggy-http`:

```bash
./build/date-server
```

In the client VM, from `/vagrant/foggy-http`, run the date client:

```bash
./build/date-client 10.0.1.1 --port 8081 --no-keep-alive
```

The client should print the current Hong Kong date as `YYYY-MM-DD`, followed by a newline. Replace `10.0.1.1` with your server VM's IPv4 address if it differs.

The following sections add HTTP support.

# HTTP

HTTP is an application protocol in which a client sends a request and a server returns a response. A request specifies a method and a target path; a response contains a status code, headers, and an optional body. See [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112.html) for the HTTP/1.1 message format and connection rules.

In this lab, HTTP runs over the ordered FoggyTCP byte stream. A single read may return part of a message or bytes from several messages, so your HTTP layer must determine message boundaries itself.

## Requests and responses

Implement serialization and parsing in `foggy-http/src/foggy_http.cc`. A message consists of a start line, headers, a blank line, and an optional body. Each line ends with the two bytes `\r\n`. For example, an ordinary echo exchange through NGINX looks like this:

```text
POST /checkpoint-3/echo HTTP/1.1\r\n
Host: 10.0.1.1:80\r\n
Connection: keep-alive\r\n
Content-Length: 5\r\n
\r\n
hello
```

```text
HTTP/1.1 200 OK\r\n
Content-Type: text/plain; charset=utf-8\r\n
Content-Length: 5\r\n
Connection: keep-alive\r\n
\r\n
hello
```

Here, `\r\n` denotes the carriage-return and line-feed bytes, not four printable characters. No newline follows the body `hello`. When connecting directly to a backend, the `Host` port is the backend port.

## Keep-alive and idle timeout

HTTP/1.1 connections are persistent by default; your implementation must honor `Connection: close`. HTTP/1.0 connections close by default; supporting explicit HTTP/1.0 keep-alive is optional.

With keep-alive, the server keeps the FoggyTCP connection open after a response so that the client can send another request on it. The server waits for the next request and closes the connection if none arrives before the idle timeout. Each completed response starts a new idle interval. The autograder checks actual connection reuse and closure after the timeout; merely sending a `Connection: keep-alive` header is not enough.

Implement the following behaviors in `foggy-http/src/foggy_http.cc`:

1. **Keep-alive:** three consecutive GETs from one initialized client reuse a single FoggyTCP connection.
2. **Idle timeout:** after completing a response, the server waits up to **3 seconds** for the next request to begin, then closes the idle connection. The timeout applies to each idle interval, not to the total connection lifetime. For example, a client that waits 10 seconds between GETs establishes a new connection and still receives a valid second response.

The client's `--interval 10` option controls the delay between requests; it does not affect the server's timeout.

With `date-server` running, run the following in the client VM:

```sh
./build/date-client 10.0.1.1 --port 8081 --repeat 3
./build/date-client 10.0.1.1 --port 8081 --repeat 2 --interval 10
```

Both commands write `RESPONSE <status> local_port=<port>` to stderr for each response. The first command should reuse one connection; the second should show a new local UDP port for the second request. Confirm this with the logs and a packet capture.

## Chunked responses

Streaming is a **server-side** option. Stop the ordinary echo server and restart it in the server VM with:

```sh
./build/echo-server --chunked
```

The supplied echo handler's `foggy_http_next_t` producer generates one payload byte every **100 ms**, about **10 bytes per second**. Your HTTP server must send each generated piece as soon as it is available. In HTTP/1.1, use `Transfer-Encoding: chunked` and omit `Content-Length`. Encode each nonempty piece as:

```text
<hexadecimal byte count>\r\n
<payload>\r\n
```

and finish with `0\r\n\r\n`. The producer returns `1` when more output remains, `0` for the last output, and `-1` on error. A call that returns `0` may still produce a nonempty final piece; send that piece before the terminating zero-length chunk. An empty piece by itself does not indicate completion.

In the client VM, run:

```sh
./build/echo-client 10.0.1.1 0123456789abcdefghij --port 8082
```

`echo-client` chooses its decoder based on the response headers and prints each decoded payload immediately through its receiver callback. No client-side flag or special request header is needed. Without the server's `--chunked` option, the same client handles an ordinary `Content-Length` response.

The 20-byte example should arrive gradually over about two seconds. stdout contains only the decoded body; stderr contains `CHUNK <milliseconds> <bytes>` records, timed from the start of the request. Check both the content and the incremental arrival times. Note that a UTF-8 character may span several single-byte chunks.

Your decoder must handle chunk-size lines, payloads, and delimiters that are split across reads. Response chunking is required; general trailer semantics are out of scope. Chunked transfer coding is an HTTP/1.1 feature; an HTTP/1.0 streamed response can instead be terminated by closing the connection.

# NGINX

NGINX can forward HTTP requests at the application layer with its HTTP module, or forward TCP/UDP traffic at the transport layer with its [Stream module](https://nginx.org/en/docs/stream/ngx_stream_proxy_module.html).

In this lab, foggy-http runs over FoggyTCP, which is carried in UDP datagrams. NGINX's HTTP module cannot interpret this transport, so we use the Stream module to forward the datagrams. To select a backend based on the HTTP request path inside FoggyTCP, we provide an NGINX extension that parses the initial packet and exposes the path to the NGINX configuration.

## Stream routing with the NGINX extension

Once direct HTTP access to the backends works, complete `foggy-http/conf/nginx-stream.conf`. Both clients use the same UDP entry port:

| Request | NGINX entry | Backend address |
| --- | --- | --- |
| GET `/checkpoint-3/date` | Server UDP 80 | `127.0.0.1:8081` |
| POST `/checkpoint-3/echo` | Server UDP 80 | `127.0.0.1:8082` |

```text
client VM                                   server VM

date-client ---- FoggyTCP / UDP ----> NGINX Stream :80
                                            | /checkpoint-3/date
                                            +--> date-server UDP :8081
                                            | /checkpoint-3/echo
                                            +--> echo-server UDP :8082
```

The NGINX extension provides the variable `$foggy_http_path`. Your configuration uses standard [map](https://nginx.org/en/docs/stream/ngx_stream_map_module.html) rules to set `$backend` from it, and `proxy_pass $backend` then forwards the session. Fill in the two `TODO` rules in the template; case-sensitive regular expression keys can match each exact path. Keep the supplied module-loading directives, the `events` block, and the `stream`/`server` structure unchanged.

The extension validates the FoggyTCP identifier and length fields, locates the payload, and reads the complete HTTP request line from the initial packet. It neither consumes nor modifies the original data: NGINX still forwards the UDP datagrams as-is, and the HTTP path only determines which backend is chosen. The extension is course-provided support code, not part of your implementation.


## Build the NGINX extension

The setup command in **Run the starter code** installs NGINX and its build dependencies. From the project root, `make` compiles the applications and the NGINX extension together:

```sh
make -j2
```

The extension is built at:

```text
build/ngx_stream_foggy_http_module.so
```

The build uses matching NGINX development headers, with bundled, SHA-256-verified NGINX 1.18.0 sources for Ubuntu 22.04; the installed NGINX executable is not modified. Because the extension must match the NGINX runtime and CPU architecture, rerun `make` after upgrading NGINX. Do not modify the extension's C source.

## Start NGINX and test the routes

Keep `date-server` and `echo-server` running in separate server VM terminals; use `echo-server --chunked` to test streaming. In another server terminal, from the project root, run:

```sh
sudo /usr/sbin/nginx -t -p "$PWD/" \
    -c "$PWD/foggy-http/conf/nginx-stream.conf" \
    -g 'pid build/nginx.pid; error_log stderr info;'
sudo /usr/sbin/nginx -p "$PWD/" \
    -c "$PWD/foggy-http/conf/nginx-stream.conf" \
    -g 'pid build/nginx.pid; error_log stderr info; daemon off;'
```

The first command checks your configuration; the second runs NGINX in the foreground with it. The `-p` option makes the extension's `build/` path relative to the project root. The PID is written to `build/nginx.pid`, and diagnostics are printed to the terminal. `sudo` is required to bind UDP 80; run the backend applications as an ordinary user.

The clients use UDP 80 by default:

```sh
./build/date-client 10.0.1.1
./build/echo-client 10.0.1.1 0123456789abcdefghij
./build/date-client 10.0.1.1 --repeat 3
./build/date-client 10.0.1.1 --repeat 2 --interval 10
```

Verify the response contents, connection reuse, reconnection after the idle interval, and incremental echo delivery. NGINX prints `FOGGY_PATH` selections and rejected initial packets to the terminal. Stop the foreground NGINX instance with Ctrl+C before starting another one on UDP 80.

# How to debug

## Packet capture

In a separate server terminal, start a capture before running a fresh client exchange:

```sh
sudo tcpdump --immediate-mode -U -i any -s 0 -w /tmp/cp3.pcap \
    'udp and (port 80 or port 8081 or port 8082)'
```

Capturing on `any` includes both the client-facing traffic and the local proxy-to-backend traffic. To capture only the external leg, use the server's LAN interface (find it with `ip -brief address`); to capture only the local backend leg, use `lo`. Stop the capture with Ctrl+C after the test.


## Wireshark analysis

The Lua files under `foggy-http/utils/` are Wireshark dissectors for decoding the captured traffic.

**The supplied dissectors decode traffic as HTTP only if the messages are correctly formatted.** The unmodified starter exchanges raw application data, so you can inspect FoggyTCP packets but will not see decoded HTTP fields until you implement valid start lines, headers, CRLF delimiters, and message framing.

```sh
wireshark -X lua_script:foggy-http/utils/foggy-tcp.lua \
          -X lua_script:foggy-http/utils/foggy-http.lua -r cp3.pcap
```


| Display filter | Inspect |
| --- | --- |
| `foggytcp` | Transport sequence numbers, acknowledgements, and payloads |
| `foggyhttp` | Reassembled HTTP over FoggyTCP |
| `http` | Decoded HTTP request and response fields |
| `udp.port == 80` | Client-facing NGINX traffic |
| `udp.port == 8081` | Date backend traffic |
| `udp.port == 8082` | Echo backend traffic |

Check the request method and path, status line, framing headers, body bytes, and `Connection` value.

Wireshark may display a chunked response as a single complete HTTP message in the frame where reassembly finishes. To verify incremental arrival, use the individual UDP packet timestamps and the echo client's `CHUNK` records.

# What to submit

Your implementation goes in two files: `foggy-http/src/foggy_http.cc` and `foggy-http/conf/nginx-stream.conf`. After rebuilding with `make -j2`, run `python3 submit.py` from the project root to generate `submit.zip`, and submit it to Gradescope. The ZIP contains these two files and an automatically generated `SHA256SUMS` file at its root. The autograder copies your two files into a fresh copy of the starter code and runs the tests.

Your submission must fully implement everything described above: HTTP requests and responses, persistent connections, idle timeouts and reconnection, chunked responses, and NGINX forwarding.

Submit the complete `nginx-stream.conf` template with your routing rules filled in. The autograder adjusts the `.so` loading paths for the grading environment and then tests your rules. The grader supplies the compiled extension, so do not include it in your ZIP.

Keep the supplied applications, public headers, transport, NGINX extension, and build files unchanged.
