# Input / Output Security

Status: **Normative design requirements**

This project sits directly in front of dangerous sinks. Its own input and output handling is therefore part of the security boundary.

The parser, framing layer, daemon transport, serializer, error path, log path, and SDK boundary must be treated as hostile-input surfaces.

The implementation must **fail closed** when these requirements cannot be satisfied.

---

## 1. Threat model

Assume an attacker can control all or part of:

- stdin;
- persistent IPC frames;
- localhost API requests;
- JSON object keys and values;
- request IDs and trace metadata;
- blocker-specific input and policy objects;
- malformed UTF-8 / Unicode sequences;
- extremely large or deeply nested payloads;
- duplicated JSON keys;
- very large integers / unusual numeric encodings;
- terminal control characters;
- CR / LF and log delimiters;
- request timing and concurrency;
- connection lifetime;
- partial writes;
- truncated frames;
- repeated requests intended to exhaust CPU, memory, disk, descriptors, or logs.

Also assume that a future SDK can accidentally pass malformed or attacker-controlled data to the Core.

No transport is trusted merely because it is local.

---

## 2. Parse safely, then validate

Validation must occur in layers.

```text
raw bytes
   |
   +-- byte / frame limit
   +-- timeout / connection budget
   v
strict UTF-8 decode
   |
   v
strict JSON parse
   |
   +-- duplicate key rejection
   +-- nesting limit
   +-- collection limits
   v
common envelope schema
   |
   v
blocker-specific schema
   |
   v
semantic validation
   |
   v
policy evaluation
```

A parser must never receive an unbounded payload.

### Required order

1. Apply the raw byte limit **before parsing**.
2. Validate framing.
3. Decode as strict UTF-8.
4. Parse JSON with duplicate-key detection.
5. Validate the common envelope.
6. Dispatch only to a known blocker/version/operation.
7. Validate the complete blocker-specific schema.
8. Perform semantic checks.
9. Resolve policy.
10. Only then construct or execute a dangerous primitive.

A failure at any step terminates that request.

---

## 3. Raw request size

Every transport must define a bounded maximum request size.

Initial recommended defaults:

| Surface | Default | Hard ceiling |
|---|---:|---:|
| one-shot CLI request | 256 KiB | 1 MiB |
| persistent IPC frame | 256 KiB | 1 MiB |
| local HTTP request body | 256 KiB | 1 MiB |
| common response | 256 KiB | 1 MiB |

These are control-plane messages, not arbitrary file-transfer channels.

Large files, archives, images, database BLOBs, or other bulk content should be supplied through bounded file handles, file descriptors, trusted local paths, streams, or blocker-specific mechanisms rather than embedded as giant JSON strings.

An administrator may lower limits. Raising the hard ceiling should require an explicit configuration change.

The byte limit is measured on the received representation, not after parsing or decompression.

---

## 4. JSON profile

Protocol v1 uses a deliberately restricted JSON profile.

### 4.1 UTF-8 only

Input must be valid UTF-8.

Reject:

- invalid UTF-8;
- isolated surrogate values;
- non-UTF encodings;
- unexpected encoding conversion.

For network / daemon JSON, reject an initial BOM rather than silently normalizing it.

### 4.2 duplicate JSON object keys

**duplicate JSON object keys MUST be rejected.**

Do not use a parser mode that silently chooses the first value, the last value, or an implementation-dependent value.

Example to reject:

```json
{
  "allowed": true,
  "allowed": false
}
```

Parser differentials are unacceptable at a security boundary.

### 4.3 Unknown fields

The common envelope rejects unknown top-level fields.

Each blocker-specific request schema must also reject unknown fields unless that field is explicitly defined as an extension point.

Do not silently ignore a misspelled security setting.

Example:

```json
{
  "allow_private_network": false,
  "allow_private_netwrok": true
}
```

Typos must not turn into security downgrades.

### 4.4 Nesting and collection limits

The parser must enforce limits before unbounded allocation.

Initial recommended limits:

- maximum JSON nesting depth: 32;
- maximum object members: 64 unless a blocker defines a smaller bound;
- maximum array items: 256 unless a blocker defines a smaller bound;
- maximum string length: field-specific;
- maximum warning count in a response: 32.

Blocker-specific schemas should use lower limits whenever possible.

### 4.5 Numbers

Do not rely on implementation-specific conversion of arbitrary JSON numbers.

Security-sensitive integers must have explicit ranges.

Values that require exact 64-bit / arbitrary precision representation should use an explicitly typed representation, for example:

```json
{
  "type": "int64",
  "value": "9223372036854775807"
}
```

Do not allow NaN or Infinity extensions.

### 4.6 Unicode normalization

Do not normalize every string globally.

Normalization can change meaning for paths, identifiers, signatures, hashes, SQL strings, URLs, and file names.

Instead:

- preserve the protocol string value as parsed;
- apply normalization only when the specific blocker specification requires it;
- compare security identifiers using blocker-defined canonicalization rules;
- never perform one normalization for validation and a different normalization for execution.

---

## 5. Framing

### 5.1 One-shot CLI

One-shot mode accepts exactly one JSON document.

After the document, only permitted trailing whitespace may occur before EOF.

Reject:

- multiple concatenated JSON documents;
- trailing non-whitespace bytes;
- payloads over the byte limit;
- truncated JSON.

stdout contains exactly one machine-readable response.

### 5.2 Persistent IPC

Preferred production framing is:

```text
4-byte unsigned big-endian length
+
exactly N bytes of UTF-8 JSON
```

Rules:

- reject zero-length frames;
- reject a length over the configured maximum before allocating the full body;
- enforce a deadline while reading the header and body;
- reject EOF in the middle of a frame;
- do not resynchronize by scanning attacker-controlled data after a malformed frame;
- close the connection after a framing violation.

### 5.3 JSON Lines

JSONL can remain available for development and interoperability, but is not the preferred production transport.

For JSONL:

- one physical line equals one request;
- raw CR / LF inside a frame is not allowed;
- JSON string newlines must be escaped;
- apply a maximum line length while reading;
- never use an unbounded `read_line` / scanner configuration;
- malformed lines return an error and do not alter parser state for the next request.

---

## 6. Daemon transport

### 6.1 Unix

Preferred transport: Unix Domain Socket.

Requirements:

- create the socket in a trusted directory;
- restrictive directory permissions;
- socket mode should default to owner-only access;
- reject unsafe pre-existing filesystem objects at the socket path;
- avoid following attacker-controlled symlinks during setup;
- use peer credential checks where the OS provides them when identity matters;
- remove the socket safely on shutdown.

### 6.2 Windows

Preferred transport: named pipe with an explicit restrictive ACL.

Do not rely on a globally accessible default pipe namespace without an ACL.

### 6.3 Local HTTP fallback

HTTP is optional and lower priority than native local IPC.

If enabled:

- bind to loopback only by default;
- never bind `0.0.0.0` / `::` without an explicit unsafe override;
- accept only documented methods;
- require the documented `Content-Type`;
- return a fixed supported `Content-Type`;
- apply body limits before parsing;
- configure header, read, write, idle, and total request timeouts;
- cap concurrent requests;
- cap header count and header bytes;
- reject ambiguous / unsupported transfer behavior using the standard HTTP stack;
- do not expose debug or profiling endpoints;
- do not trust `X-Forwarded-*` headers in the default local mode.

If a non-local deployment is ever supported, it requires a separate threat model and authentication design.

---

## 7. Authentication and local authorization

"localhost" is not an authentication mechanism.

Where only the current user should access the daemon:

- prefer OS access control on UDS / named pipe;
- use peer credentials when available;
- do not put long-lived secrets in command-line arguments;
- do not log authentication material;
- if a fallback token is required, generate a cryptographically random token and store/pass it through a protected channel rather than embedding it in configuration committed to source control.

Policy override privileges must be distinct from ordinary request privileges.

An untrusted application caller must not be able to weaken administrator policy.

---

## 8. Request identifiers and metadata

`request_id`, `trace_id`, service names, and operation names are untrusted.

They must:

- have explicit maximum lengths;
- use a restricted character set when used in logs or metrics;
- never become filesystem paths;
- never become SQL identifiers;
- never be passed to shell commands;
- never be copied into HTTP headers without header-safe validation.

Security decisions must not trust caller-supplied identity metadata unless the transport establishes its provenance.

---

## 9. Output construction

Machine responses must be constructed with a serializer.

Never create JSON through string concatenation.

Bad:

```text
"{\"error\":\"" + user_value + "\"}"
```

Required:

```text
typed response object
        |
        v
JSON serializer
        |
        v
stdout / IPC
```

The serializer must produce valid UTF-8 JSON.

Response size is bounded. A malicious request must not cause unbounded diagnostic output.

---

## 10. Error responses

Public errors contain:

- stable machine-readable error code;
- short bounded human-readable message;
- optional safe field identifier;
- bounded non-sensitive details.

Public errors must not contain:

- stack traces;
- memory addresses;
- filesystem layout unless explicitly safe;
- environment variables;
- process arguments;
- raw SQL parameters;
- API tokens;
- Authorization values;
- cookies;
- private keys;
- raw request bodies;
- arbitrary parser exception dumps.

The raw request must not be copied into an error merely to aid debugging.

Internal details can be attached to a private diagnostic ID in a controlled debug environment, not returned to the caller.

---

## 11. stdout and stderr

For CLI and stdio protocol modes:

- stdout is protocol data only;
- stderr is diagnostic/log output only;
- no banners, progress bars, colors, warnings, or dependency messages may appear on stdout;
- a request must produce at most one protocol response frame;
- library panics/exceptions must not print uncontrolled content into stdout.

This separation is a compatibility and security invariant.

---

## 12. Log security

The default logger must use structured fields.

Do not interpolate attacker-controlled values into ad-hoc log lines.

Before rendering human-readable stderr or terminal output:

- escape CR and LF;
- escape ESC / ANSI control sequences;
- escape control characters;
- make delimiter boundaries explicit;
- consider escaping bidi control characters so log text cannot visually reorder security-relevant values.

Secret-bearing fields are redacted by construction, not by an after-the-fact regex alone.

Default logs should contain metadata such as:

- blocker;
- operation;
- decision;
- stable error code;
- policy ID/version;
- timing bucket;
- bounded validated request ID.

Default logs should not contain parameter values or the full request.

---

## 13. Backpressure and resource limits

Every server mode needs explicit budgets.

At minimum:

- maximum active connections;
- maximum requests per connection;
- maximum concurrent in-flight requests;
- per-request timeout;
- idle timeout;
- bounded input queue;
- bounded output queue;
- bounded log rate;
- bounded response bytes.

When overloaded, reject work rather than allocate unbounded queues.

Cancellation must propagate to blocker work where possible.

A timeout or cancellation must fail closed and must not accidentally continue the dangerous operation in a detached worker.

---

## 14. Desynchronization and partial failures

Protocol state must be simple.

For length-prefixed IPC:

- a malformed frame invalidates the connection;
- do not guess where the next frame begins.

For one-shot mode:

- parser failure returns one error response when safely possible, then exits non-zero.

For persistent modes:

- errors are request-scoped only if the framing boundary is still trustworthy;
- framing corruption closes the connection.

---

## 15. Output amplification

An attacker should not be able to submit a tiny request and force an enormous response.

Examples:

- parser errors return one bounded message, not every token error;
- validation returns a bounded number of violations;
- SQL parser debug trees are never returned by default;
- file/archive listings have explicit item limits;
- warnings are capped.

---

## 16. Cryptographic canonicalization

Ordinary protocol processing does not require canonical JSON.

If requests, policies, manifests, or execution plans are ever hashed or signed, use a documented canonical representation rather than serializing an arbitrary in-memory map.

JSON Canonicalization Scheme (RFC 8785) is a candidate for signed JSON material.

Do not confuse canonicalization with input validation: signed data still requires strict parsing and semantic validation.

---

## 17. Schema strategy

The repository contains:

- `schemas/common-request.schema.json`
- `schemas/common-response.schema.json`

They use JSON Schema 2020-12 for the common envelope.

The common envelope is only the first validation stage.

Every blocker must define its own complete schema for:

- `input`;
- `policy`;
- operation-specific output;
- typed parameters.

Blocker schemas must reject unknown security-sensitive fields.

---

## 18. SDK boundary

Language SDKs must not silently weaken the Core contract.

SDK requirements:

- use typed APIs where practical;
- enforce local size limits before serialization;
- do not stringify arbitrary objects by calling user-controlled display methods in privileged contexts;
- preserve type distinctions;
- never combine query/command structure with parameter values;
- map Core error codes to typed errors without exposing hidden diagnostics;
- apply connection and request timeouts;
- do not retry blocked requests automatically;
- only retry transport-safe idempotent operations under an explicit retry policy.

---

## 19. Fuzzing requirements

Once executable code exists, input handling is a mandatory fuzzing target.

Minimum fuzz targets:

1. raw frame decoder;
2. strict UTF-8 decoder boundary;
3. JSON duplicate-key detector;
4. common request parser;
5. blocker dispatcher;
6. each blocker-specific schema decoder;
7. error serializer;
8. persistent connection state machine.

Corpus categories:

- truncated UTF-8;
- overlong / invalid encodings;
- deeply nested arrays/objects;
- duplicate keys at every nesting level;
- very long keys;
- empty and oversized frames;
- integer boundaries;
- escaped control characters;
- Unicode bidi/control characters;
- null bytes;
- partial length prefixes;
- frame length/body mismatch;
- many tiny frames;
- huge declared frame length;
- malformed JSON followed by a valid frame.

Fuzzing must be combined with memory / undefined-behavior tooling appropriate to the implementation language.

---

## 20. Differential testing

Where multiple parser implementations or SDKs exist, feed the same corpus into each implementation.

A request must not be:

- accepted by one SDK and rejected by Core for an undocumented reason;
- interpreted as different key/value mappings;
- normalized differently before a security decision;
- serialized into semantically different execution plans.

Any parser differential at the security boundary is treated as a bug.

---

## 21. Required negative tests

Each transport must test at least:

- request exactly at the size limit;
- request one byte over the limit;
- invalid UTF-8;
- BOM;
- duplicate keys;
- unknown top-level field;
- unknown blocker field;
- missing required field;
- wrong JSON type;
- 33-level nesting when max depth is 32;
- oversized request ID;
- CR/LF in metadata;
- ANSI escape in metadata;
- truncated frame header;
- truncated frame body;
- very large declared frame;
- multiple one-shot documents;
- stdout contamination;
- secret value in an error path;
- timeout during parse;
- timeout during blocker processing.

---

## 22. References used for the design

- OWASP Cheat Sheet Series: Input Validation
- OWASP Cheat Sheet Series: REST Security
- OWASP Cheat Sheet Series: Logging
- OWASP Cheat Sheet Series: Denial of Service
- OWASP API Security Top 10: Unrestricted Resource Consumption / Unsafe Consumption of APIs
- RFC 8259: JSON
- RFC 8785: JSON Canonicalization Scheme
- JSON Schema 2020-12

The implementation should prefer mature parser, schema, HTTP, IPC, and serialization libraries over custom parsers.
