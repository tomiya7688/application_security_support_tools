# Common Protocol

## 1. Purpose

すべてのSecurity Blockerで共通して利用するrequest / response envelopeを定義します。

目標は、CLI、daemon、各言語SDKが同じ意味のrequestを扱えるようにすることです。

Blocker固有の入力は `input` と `policy` の内部で定義し、transportや共通error modelは共有します。

入力・出力そのものがsecurity boundaryであるため、wire format・framing・parser制約は [Input / Output Security](input-output-security.md) のnormative requirementsに従います。

---

## 2. Request envelope

基本形:

```json
{
  "protocol_version": "1",
  "request_id": "optional-caller-generated-id",
  "blocker": "sql",
  "blocker_version": "0.1",
  "operation": "prepare",
  "input": {},
  "policy": {},
  "context": {}
}
```

### Fields

| Field | Required | Description |
|---|---:|---|
| `protocol_version` | yes | 共通protocol version |
| `request_id` | no | caller側のtrace用ID |
| `blocker` | yes | `sql`, `command`, `path`, `url` など |
| `blocker_version` | yes | blocker固有仕様version |
| `operation` | yes | `prepare`, `validate`, `execute` など |
| `input` | yes | blocker固有入力 |
| `policy` | no | request単位policy。省略時はdefault policy |
| `context` | no | traceやadapter情報。security decisionに使う場合は仕様で明示 |

---

## 3. Response envelope

```json
{
  "protocol_version": "1",
  "request_id": "optional-caller-generated-id",
  "blocker": "sql",
  "blocker_version": "0.1",
  "decision": "allow",
  "allowed": true,
  "output": {},
  "warnings": [],
  "metadata": {}
}
```

拒否例:

```json
{
  "protocol_version": "1",
  "request_id": "req-123",
  "blocker": "sql",
  "blocker_version": "0.1",
  "decision": "block",
  "allowed": false,
  "error": {
    "code": "SQL_MULTIPLE_STATEMENTS_NOT_ALLOWED",
    "message": "Only one SQL statement is allowed.",
    "field": "input.statement"
  },
  "warnings": [],
  "metadata": {}
}
```

protocol error例:

```json
{
  "protocol_version": "1",
  "request_id": "req-123",
  "blocker": "sql",
  "blocker_version": "0.1",
  "decision": "error",
  "allowed": false,
  "error": {
    "code": "PROTOCOL_INVALID_REQUEST",
    "message": "Required field 'operation' is missing."
  },
  "warnings": [],
  "metadata": {}
}
```

---

## 4. Decision semantics

`decision` は以下のいずれかです。

### `allow`

Blockerの仕様とpolicy上、その操作を続行できます。

### `block`

requestは構文上理解できるものの、security policyにより拒否されました。

例:

- private networkへのURL access
- root外へ出るpath
- 許可されていないcommand
- 複数SQL statement

### `error`

request自体が無効、未対応、または内部処理に失敗しています。

例:

- protocol field不足
- unsupported version
- invalid encoding
- adapter failure
- internal error

`allowed` はcallerが簡単に判定するための補助fieldです。

```text
decision == "allow" -> allowed == true
otherwise           -> allowed == false
```

---

## 5. Error object

基本形:

```json
{
  "code": "SQL_PARAMETER_COUNT_MISMATCH",
  "message": "Parameter count does not match placeholders.",
  "field": "input.parameters",
  "details": {}
}
```

### Rules

- `code` は機械判定用で安定したpublic interfaceとする
- `message` は人間向け
- `field` は可能な場合のみ指定
- `details` にsecretを入れない
- error message内へraw secret / token / passwordを埋め込まない

---

## 6. Warnings

security上即時拒否する必要はないが、callerが認識すべき事項を返します。

例:

```json
{
  "code": "SQL_DEPRECATED_DIALECT_OPTION",
  "message": "This dialect option will be removed in a future version."
}
```

warningは `allow` decisionと同時に返せます。

security上危険な状態をwarningだけで通す設計は避けます。

---

## 7. Metadata

security decisionの監査や性能計測に使います。

例:

```json
{
  "policy_id": "default",
  "policy_version": "3",
  "processing_time_us": 184,
  "adapter": "postgresql",
  "normalized": false
}
```

defaultではuser dataやsecretを含めません。

---

## 8. Policy model

policyは3段階で解決することを想定します。

```text
built-in secure defaults
        |
        v
configured application policy
        |
        v
request policy overrides
```

request overrideを許可するかどうか自体もapplication policyで制御できるようにします。

productionでは、untrusted callerがpolicyを弱められないことが重要です。

### Recommended precedence

```text
hard security invariant
    > administrator policy
    > application policy
    > request override
```

hard invariantはrequestから解除できません。

---

## 9. Context

`context` はsecurity decision以外のtrace情報を格納できます。

例:

```json
{
  "trace_id": "abc",
  "service": "user-api",
  "operation_name": "get_user"
}
```

tenant / user roleなどをpolicy decisionに利用する場合は、値の信頼元を定義する必要があります。

callerが自由に偽装できるcontextをsecurity authorizationに直接使ってはいけません。

---

## 10. Transport

すべてのtransportで、raw byte limitを**parse前**に適用し、strict UTF-8、duplicate JSON key rejection、schema validationを共通要件とします。

### 10.1 One-shot stdin/stdout

1 request / 1 response。

```bash
printf '%s\n' '{"protocol_version":"1", ...}' | security-blocker
```

one-shot modeはJSON documentをちょうど1個だけ受理します。

- payload sizeを読む段階で制限
- trailing whitespace以外の追加documentを拒否
- stdoutはmachine-readable responseのみ
- human-readable logsはstderrのみ
- parser / internal errorでもstdoutへstack trace等を混入させない

### 10.2 Persistent local IPC

production向けpersistent transportの第一候補です。

wire framing:

```text
4-byte unsigned big-endian length
+
exactly N bytes of UTF-8 JSON
```

受信側はlengthを読んだ時点で上限を確認し、巨大な宣言長に基づいて無制限allocationしてはいけません。

framing violation、途中EOF、oversized frameではconnectionを閉じます。破損したstreamから次frameの位置を推測してresynchronizeしません。

UnixではUnix Domain Socket、Windowsではrestrictive ACLを持つnamed pipeを優先します。

### 10.3 Persistent JSON Lines

JSONLは開発・debug・interop用途として提供できますが、productionの第一選択にはしません。

```text
request 1\n
request 2\n
request 3\n
```

要件:

- maximum line lengthを読む前から強制
- 1 physical line = 1 request
- JSON string内部の改行はescape必須
- unbounded `read_line` を使わない
- malformed JSONを次requestへ連結しない

### 10.4 Local HTTP daemon

必要な環境向けのfallbackです。

例:

```text
POST /v1/block
POST /v1/sql/prepare
POST /v1/path/resolve
```

ただし内部では共通envelopeへ変換します。

default:

- loopback bindのみ
- strict method / Content-Type
- parse前body limit
- header size/count limit
- read/write/idle/total timeout
- concurrency limit
- debug/profiling endpointなし
- `X-Forwarded-*` を信頼しない

non-local bindは別security modelとして扱い、暗黙には有効化しません。

---

## 11. Serialization rules

初期実装は UTF-8 JSON を基準とします。

Protocol v1のJSON profileでは以下を必須とします。

- valid UTF-8 only
- duplicate object keys reject
- unknown common-envelope fields reject
- blocker schemaでもunknown security-sensitive fields reject
- nesting / collection / string lengthに明示的上限
- NaN / Infinityなど非標準JSON number拡張を受理しない
- serializerでresponseを生成し、文字列連結でJSONを組み立てない

global Unicode normalizationは行いません。canonicalizationが必要な値はBlocker固有仕様で定義し、validationとexecutionで同一規則を使用します。

将来的な候補:

- MessagePack
- CBOR
- protobuf

ただしserialization形式を変えても、logical protocolは同じ意味を維持します。

---

## 12. Sensitive data handling

Blockerはsecurity boundaryに位置するため、入力自体が機密情報を含む可能性があります。

原則:

- request bodyをdefaultでlogしない
- parameter valuesをdefaultでlogしない
- token / password / cookie / Authorization headerをlogしない
- debug modeでも明示的redactionを行う
- crash reportへraw requestを載せない

---

## 13. Compatibility

### Protocol version

transportや共通envelopeのbreaking change。

### Blocker version

個別Blockerのinput / policy / decision semanticsのbreaking change。

例:

```json
{
  "protocol_version": "1",
  "blocker": "sql",
  "blocker_version": "0.1"
}
```

SDKは対応version範囲を明示します。

---

## 14. Security invariant

共通protocol全体で最も重要な原則:

> Blockerが理解できない入力を、推測で安全扱いしない。

unknown / unsupported / ambiguousな状態では、原則として `block` または `error` を返します。

加えて、parser differential、framing ambiguity、resource limit超過、timeout、cancellationもallowへfallbackしてはいけません。
