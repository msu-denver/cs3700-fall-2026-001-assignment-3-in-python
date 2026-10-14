# CS 3700 Assignment 03: Pyro5 Protocol — Your Own Name Server

## Your name

## Your student number

## Overview

In Assignments 1 and 2 you wrote **clients** that talked to Pyro5's name
server and echo server. In this assignment you switch sides: you'll write
your own **name server**, from scratch, that the real Pyro5 tools can talk
to.

By the end, you'll have written a server (in Python) that:
1. Listens for TCP connections on `localhost:9090`
1. Answers the Pyro5 connection handshake
1. Answers a `list` call with the names it knows about
1. Keeps running, ready for the next client

When it works, the real, unmodified `pyro5-nsc list` prints exactly what it
prints for the real name server:

```
--------START LIST 
Pyro.NameServer --> PYRO:Pyro.NameServer@localhost:9090
    metadata: {'class:Pyro5.nameserver.NameServer'}
--------END LIST 
```

Your name server only has to know about itself. Registering other services
comes later.

---

## Background: The Server Side of RPC

A client builds a request, sends it, and waits. A server does the reverse:

1. **Accept** a connection
1. **Read** a complete message: a fixed-size header, then the payload
1. **Decide** what was asked for (a handshake? which method?)
1. **Build** the reply: a payload, wrapped in a header
1. **Send** it, then go back to reading

The server has to get the details exactly right, because the client is
strict: a wrong header field, sequence number or payload and `pyro5-nsc`
reports an error instead of the list.

### Language Independence

The real Pyro5 tools are written in Python, but they can't tell what
language your server is written in. All they see is bytes on the wire. A
classmate's Java name server and your Python one should be
indistinguishable to `pyro5-nsc`.

---

## Setup: Install Dependencies

```bash
pip install Pyro5==5.17
```

You'll use the Pyro5 tools to **test** your server. Your server itself must
not use them (see the rules below).

---

## Part 1: Observe

### 1.1 Watch the Real Name Server

**Terminal 1 — the real name server:**
```bash
pyro5-ns
```

**Terminal 2 — list what it knows:**
```bash
pyro5-nsc list
```

Capture this exchange with Wireshark on the loopback interface, as in
Assignment 1. You should see exactly **two** requests from the client, each
with a reply:

| # | From | Message type |
|---|---|---|
| 1 | `pyro5-nsc` | `CONNECT` (1): the handshake |
| 2 | name server | `CONNECTOK` (2): handshake accepted |
| 3 | `pyro5-nsc` | `INVOKE` (4): call a method |
| 4 | name server | `RESULT` (5): the method's return value |

### 1.2 Annotate What You Captured

Create `ANNOTATED.md` showing all four messages. For each one, label every
header field and explain the payload. Pay particular attention to:
- what the handshake reply has to contain for the client to accept it
- which method the client calls, with which arguments
- how the reply's sequence number relates to the request's
- how the metadata is written in serpent (it's a **set**)

### 1.3 The Header

Every Pyro5 5.17 message starts with a **40-byte** header, in network
(big-endian) byte order:

| Bytes | Size | Field |
|---|---|---|
| 0–3 | 4 | Tag: the ASCII bytes `PYRO` |
| 4–5 | 2 | Protocol version: 502 |
| 6 | 1 | Message type |
| 7 | 1 | Serializer id: 1 for serpent |
| 8–9 | 2 | Flags |
| 10–11 | 2 | Sequence number |
| 12–15 | 4 | Payload size |
| 16–19 | 4 | Annotations size |
| 20–35 | 16 | Correlation id |
| 36–37 | 2 | Reserved: 0 |
| 38–39 | 2 | Magic number: `0x4dc5` |

The annotations, if any, come right after the header, followed by the
payload.

---

## Part 2: Implement — Name Server

### 2.1 Design

Your server must:

1. **Listen** on `localhost:9090`
1. **Accept** a connection
1. **Read** a message: the 40-byte header, then exactly as many bytes as
   the header says follow
1. **Check** the header (tag, version) and reply to:
   - `CONNECT` with `CONNECTOK`, including the metadata the client needs
   - `INVOKE` of `list` with a `RESULT` mapping each name to its URI and
     metadata
1. **Use serpent** for every payload you send
1. **Close** the connection when the client does, and **keep listening**
   for the next one

Requests you don't support should get an error reply, not a crash.

### 2.2 Implement

**Files:**
- `name_server.py` — starts the server; `python name_server.py` with no
  arguments must listen on `localhost:9090`
- more modules if you like, such as `pyro_message.py` for building and
  parsing messages
- `tests/test_*.py` — your unit tests

**Use only the Python standard library** in your server: `socket`,
`struct`, and so on.
- Don't import `Pyro5`, and don't start `pyro5-ns`. The CI checks for both.
- Don't use the `serpent` package either: serpent payloads are text in
  Python literal syntax, so build them yourself. CI flags `serpent` imports
  for the instructor to review.
- Your **tests** may use `Pyro5`, for example to call your running server.

**Suggested pieces** (organize your code however you like):

1. **`read_message(sock)`** — read one complete message; return its header
   fields and payload
1. **`build_message(msg_type, seq, payload)`** — wrap a payload in a
   40-byte header
1. **`handshake_reply(...)`** — the `CONNECTOK` payload
1. **`list_reply(...)`** — the `RESULT` payload for `list`
1. **`serve_forever()`** — accept connections and dispatch messages

Small, separate pieces like these are much easier to unit test than one
long loop.

### 2.3 Test Locally

```bash
# Terminal 1: your name server
python name_server.py

# Terminal 2: the real client
pyro5-nsc list
```

The output must match the list at the top of this README **exactly**,
including the space after `START LIST` and `END LIST` (`pyro5-nsc` prints
those for you).

Stop the real `pyro5-ns` before you start yours; only one program can
listen on port 9090.

Run your unit tests with:

```bash
python -m pytest -v
```

---

## What the CI Checks

Every push runs four checks. Open the **Actions** tab on GitHub to see the
results; the **Summary** page for a run shows your static analysis score.

| Check | What it does |
|---|---|
| **integrity** | `SOURCES.md` exists, and the server doesn't use Pyro5 |
| **static-analysis** | `ruff` lint and security rules; **20% of your grade** |
| **unit-tests** | Runs your tests with coverage |
| **name-server** | Starts `name_server.py` and compares `pyro5-nsc list` with the expected output |

Run the same static analysis locally before you push:

```bash
pip install ruff
ruff check . --select E,W,F,B         # lint
ruff check . --select S --extend-exclude 'tests,test_*.py'   # security
```

---

## Resources

Cite every resource you used in `SOURCES.md`. Include:
- Pyro5 wire protocol: <https://pyro5.readthedocs.io/en/latest/api/protocol.html>
- Pyro5 command-line tools: <https://pyro5.readthedocs.io/en/latest/commandline.html>
- Serpent: <https://github.com/irmen/Serpent>
- Any AI you used, and include the prompts so it can be replicated
- Any other sources

---

## Academic Integrity & Professional Practices

**You are encouraged to use all available resources.** This mirrors real
professional development:

✅ **DO:**
- Search Stack Overflow, documentation, GitHub
- Use AI (Claude, ChatGPT, Copilot) for explanations, debugging, ideas
- Copy code from tutorials, examples, open-source projects
- Ask classmates, TAs, instructor for help

❌ **DON'T:**
- Copy code without understanding it
- Submit work entirely generated by AI without your input
- Claim you wrote code you didn't write

**The requirement:** **Cite all sources in a `SOURCES.md` file.** For AI,
include your prompts and say what you changed in what it gave you.

**Penalty for missing citations:** Automatic zero on assignment, regardless
of code quality.

**Why?** Learn to work like a real engineer: use all available tools, but
be transparent about sources.
