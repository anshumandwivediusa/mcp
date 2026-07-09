# Model Context Protocol (MCP)

The open standard for connecting AI applications to tools, data and prompts — architecture, message lifecycle and wire-format reference in one page.

`Spec: 2025-11-25` · `Transport: stdio / Streamable HTTP` · `Primitives: tools · resources · prompts` · `Docs: modelcontextprotocol.io`

---

## Table of contents

1. [Architecture](#1-architecture)
2. [The four primitives](#2-the-four-primitives)
3. [Connection lifecycle](#3-connection-lifecycle)
4. [JSON-RPC 2.0 message anatomy](#4-json-rpc-20-message-anatomy)
5. [Tool discovery & invocation](#5-tool-discovery--invocation)
6. [Transports](#6-transports)
7. [Error codes & tool errors](#7-error-codes--tool-errors)
8. [Quick glossary](#8-quick-glossary)

---

## 1. Architecture

MCP follows a client–server model. A **host** application runs one **client** per connected **server**, each a stateful 1:1 session. Servers expose capabilities without needing to know anything about the host.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/2a3fde43-5f7c-462c-b831-40b17a8aeba4" />


| Role | Color-coded as | Description |
|---|---|---|
| **Host** | Violet | Runs the LLM, owns the UI |
| **Client** | Blue | 1:1 session manager |
| **Server** | Teal | Exposes capabilities |
| **Backend** | Amber | The real system of record |

---

## 2. The four primitives

Everything a server offers falls into one of these. Who controls invocation differs by primitive — that distinction drives most design decisions.

| Primitive | Controlled by | Description | Key methods |
|---|---|---|---|
| **Tools** | Model-controlled | Executable functions with a JSON-Schema input. The model decides when to call them; a human should approve. | `tools/list` · `tools/call` |
| **Resources** | App-controlled | Read-only data identified by a URI — files, rows, tickets. Attached by the host, not chosen by the model. | `resources/list` · `resources/read` |
| **Prompts** | User-controlled | Reusable message templates with slots — surfaced as slash-commands or menu items the user picks. | `prompts/list` · `prompts/get` |
| **Sampling** | Server-initiated | Lets a server ask the host's model to complete a prompt — for agentic behaviour without its own API key. | `sampling/createMessage` |

---

## 3. Connection lifecycle

Every session moves through three phases. Nothing in the operation phase is valid until both sides have completed the handshake.

| Phase | Message | Description |
|---|---|---|
| **Init** | `client → initialize` | Client sends its protocol version and capabilities. Server responds with its own version, capabilities and instructions. |
| **Init** | `client → notifications/initialized` | One-way notification confirming the handshake is complete. The session is now live. |
| **Operation** | `↔ tools / resources / prompts` | Normal request-response traffic for as long as the session is open — discovery calls and invocations in any order. |
| **Operation** | `server ⇢ notifications/tools/list_changed` | Server-pushed notification when its capability set changes; client re-fetches the affected list. |
| **Shutdown** | `— transport closes` | stdio: client closes the subprocess's stdin, then terminates it. HTTP: connection or session token is dropped. |

---

## 4. JSON-RPC 2.0 message anatomy

Every MCP message is one of three shapes. All carry `"jsonrpc": "2.0"`; only requests and responses share an `id`.

### Request — has `id`

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": { }
}
```
Sender expects exactly one matching response.

### Response — same `id`

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": { }
}
```
```json
// or on failure:
{ "error": { "code": 0, "message": "" } }
```
Exactly one of `result` / `error`, never both.

### Notification — no `id`

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```
Fire-and-forget — no reply is ever sent.

---

## 5. Tool discovery & invocation

The two calls that make up almost every MCP interaction, shown wire-format.

### 1 — Discover: `tools/list`

```json
→ { "method": "tools/list", "id": 1 }

← {
  "id": 1,
  "result": { "tools": [{
    "name": "get_weather",
    "description": "Current conditions",
    "inputSchema": {
      "type": "object",
      "properties": {
        "location": { "type": "string" }
      },
      "required": ["location"]
    }
  }]}
}
```

### 2 — Invoke: `tools/call`

```json
→ {
  "method": "tools/call", "id": 2,
  "params": {
    "name": "get_weather",
    "arguments": { "location": "NYC" }
  }
}

← {
  "id": 2,
  "result": {
    "content": [{ "type": "text", "text": "72°F, partly cloudy" }],
    "isError": false
  }
}
```

---

## 6. Transports

| Transport | How it moves bytes | Best for |
|---|---|---|
| **stdio** | Server runs as a local subprocess. Newline-delimited JSON-RPC on stdin/stdout; logs go to stderr only. | Local tools — filesystem, git, desktop apps launched by the host. |
| **Streamable HTTP** | Single HTTP endpoint accepts POSTed messages and can upgrade to SSE for streaming server → client traffic. | Remote / hosted servers, multi-client fan-out, cloud deployments. |

---

## 7. Error codes & tool errors

Protocol-level failures use standard JSON-RPC codes. Failures *inside* a tool's own logic are reported differently — as a normal result with `isError: true` — so the model can see and react to them.

| Code | Meaning |
|---|---|
| `-32700` | Parse error — invalid JSON |
| `-32600` | Invalid request shape |
| `-32601` | Method not found |
| `-32602` | Invalid params (also: unknown tool) |
| `-32603` | Internal error |

**Tool execution error** (not a protocol error):

```json
{
  "id": 4,
  "result": {
    "content": [{ "type": "text", "text": "Rate limit exceeded" }],
    "isError": true
  }
}
```
Call still succeeded at the protocol level — the model reads the error text.

---

## 8. Quick glossary

| Term | Definition |
|---|---|
| **Host** | The AI application end-users interact with — owns the model and the UI. |
| **Client** | A host-owned object holding exactly one session with one server. |
| **Server** | A process exposing tools, resources and/or prompts over MCP. |
| **Capability negotiation** | The initialize exchange where both sides declare what they support. |
| **Session** | The stateful connection lifetime between initialize and shutdown. |
| **Roots** | Filesystem boundaries a client tells a server it may operate within. |
| **Sampling** | A server requesting an LLM completion back from the host. |
| **listChanged** | Capability flag: server will notify when its tool/resource/prompt list changes. |
| **N × M problem** | The integration explosion MCP replaces with N + M by standardising the interface. |

---

*MCP reference sheet · JSON-RPC 2.0 · spec 2025-11-25*
*Full spec: [modelcontextprotocol.io/specification](https://modelcontextprotocol.io/specification)*
