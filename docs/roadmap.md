# Roadmap

## 1. Development sequence

このプロジェクトでは、各Security Blockerを次の順序で実装します。

```text
Blocker N
  |
  +--> 1. Core specification
  +--> 2. CLI / API reference implementation
  +--> 3. Security test vectors
  +--> 4. Language libraries / SDKs
  +--> 5. Native execution adapters
  |
  v
Blocker N + 1
```

つまり、複数Blockerを浅く並行実装するより、**1つのBlockerを実際に使える状態まで通してから次へ進む**方針です。

---

## 2. Definition of done for one blocker

次の条件を満たした時点で、そのBlockerの初期実装を「一通り完成」とみなします。

### Specification

- threat modelが書かれている
- 守れる範囲と守れない範囲が明記されている
- input schemaが定義されている
- policy schemaが定義されている
- error codeが定義されている
- fail-closed behaviorが定義されている

### CLI / API

- stdin/stdoutでrequestを受けられる
- persistent modeで複数requestを処理できる
- structured JSON responseを返せる
- machine-readable error codeがある
- stdoutとlog出力が分離されている

### Security tests

- 正常系
- 代表的attack payload
- encoding variation
- boundary case
- malformed request
- policy bypass attempt
- regression test

### SDK

各言語SDKは、単なるHTTP clientではなく、その言語でdangerous sinkを安全に置き換えやすいAPIを提供します。

最低限:

- Core呼び出し
- typed error
- lifecycle
- adapter
- examples
- tests

---

## 3. Initial blocker order

初期優先順位:

### 1. SQL Blocker

対象:

- SQL Injection
- stacked statement misuse
- unsafe dynamic SQL patterns
- unsafe identifier handling

主な防御:

- statementとparameterの分離
- parameter binding
- statement registration mode
- statement policy
- dialect-aware validation
- single statement restriction by default

SQL Blockerを最初に実装し、Core / protocol / SDK方針を固めます。

### 2. Command Blocker

対象:

- OS Command Injection
- shell metacharacter injection
- executable substitution
- unsafe environment inheritance

主な防御:

- executableとargvの分離
- shellをdefaultで使用しない
- executable allowlist
- cwd / env policy
- timeout / resource policy

### 3. Path Blocker

対象:

- Path Traversal
- absolute path escape
- encoded traversal
- symlink / junction経由のescape

主な防御:

- trusted root
- path component validation
- canonical containment
- OS別path semantics
- handle-based executionの検討

### 4. URL Blocker

対象:

- SSRF
- local network access
- cloud metadata access
- unsafe redirect
- protocol abuse

主な防御:

- scheme policy
- hostname policy
- DNS resolution
- resolved IP policy
- redirect再検証
- private/link-local/loopback制御

### 5. HTML Blocker

対象:

- reflected XSS
- stored XSS
- unsafe DOM output

主な防御:

- output contextの明示
- context-aware encoding
- safe sink
- raw HTMLの明示的制限

---

## 4. Later candidates

順序は実装知見に応じて変更します。

- Egress Blocker
- Agent Action Blocker
- Resource Budget Blocker
- Archive Extraction Blocker
- Object Update Blocker
- Deserialize Blocker
- Template Blocker
- Parser Sandbox Blocker
- Dynamic Load Blocker
- Webhook / Replay Blocker
- Redirect Blocker
- Header Blocker
- File Blocker
- Log Blocker
- XML Blocker

新しいBlockerを追加するときは「入力フィルタ」ではなく、**どのdangerous sinkを安全なprimitiveへ置き換えるのか**を最初に定義します。

現代的な攻撃、特にAI/Agent・複雑なparser・resource exhaustionについても同じ原則を適用します。Prompt Injectionのように完全検出が困難な攻撃では、検出器をsecurity boundaryにせず、最終的なtool execution、outbound network、filesystem、process、resource consumptionをBlockerで制約します。

詳細は [Modern Threats and Late-Stage Guards](modern-threats.md) を参照してください。

---

## 5. SQL Blocker milestones

### SQL-M0: Specification

- [x] project architecture
- [x] common protocol draft
- [x] SQL Blocker v0.1 draft
- [ ] threat model review
- [ ] initial error code freeze
- [ ] test vector definition

### SQL-M1: CLI Core

- [ ] executable skeleton
- [ ] JSON request parser
- [ ] JSON response writer
- [ ] protocol validation
- [ ] blocker dispatch
- [ ] SQL request schema
- [ ] dialect abstraction
- [ ] parameter validation
- [ ] single-statement policy
- [ ] registered statement mode
- [ ] structured errors

### SQL-M2: Runtime modes

- [ ] one-shot stdin/stdout
- [ ] persistent JSONL
- [ ] local daemon transport
- [ ] Unix Domain Socket
- [ ] localhost transport where needed
- [ ] benchmark harness

### SQL-M3: Security tests

- [ ] quote-based injection payloads
- [ ] boolean-based injection payloads
- [ ] comment payloads
- [ ] stacked query attempts
- [ ] encoding edge cases
- [ ] dialect-specific cases
- [ ] parameter count mismatch
- [ ] unsupported syntax
- [ ] registered statement bypass attempts
- [ ] fuzzing harness

### SQL-M4: Language SDKs

候補順:

1. Python
2. Node.js / TypeScript
3. Go
4. Java / Kotlin
5. PHP
6. Ruby
7. .NET

順番は利用需要とadapter実装コストに応じて変更できます。

各SDKは共通Coreと同じtest vectorを共有します。

### SQL-M5: Native execution integration

- [ ] PostgreSQL adapter
- [ ] MySQL adapter
- [ ] SQLite adapter
- [ ] connection lifecycle design
- [ ] transaction semantics
- [ ] cancellation / timeout
- [ ] error mapping

この段階で、SDKがvalidationだけでなく安全なDB driver APIの呼び出しまで所有できる状態を目指します。

---

## 6. Per-blocker repository layout

想定:

```text
/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── common-protocol.md
│   ├── roadmap.md
│   └── blockers/
│       ├── sql-v0.1.md
│       ├── command-v0.1.md
│       ├── path-v0.1.md
│       └── ...
├── core/
├── blockers/
│   ├── sql/
│   ├── command/
│   └── ...
├── cli/
├── sdk/
│   ├── python/
│   ├── node/
│   ├── go/
│   └── ...
└── tests/
    ├── vectors/
    └── integration/
```

実装言語決定前のため、現時点では概念的な構成です。

---

## 7. Release strategy

各Blockerは独立したversionを持てるようにします。

例:

```text
Core Protocol: v1
SQL Blocker:   v0.1
Command:       v0.1
Path:          v0.1
```

初期段階では `0.x` とし、API / policy / guaranteeの変更を許容します。

`1.0` へ上げる条件:

- threat modelが安定
- public APIが安定
- security test corpusが十分
- production利用のfeedbackがある
- guarantee boundaryが明文化されている

---

## 8. Guiding rule

開発順序を決めるときは、実装量より次を優先します。

> そのBlockerを使うことで、開発者が危険なAPIを直接触る必要をどれだけ減らせるか。

これが小さいものは、単なるvalidation utilityになりやすいため優先度を下げます。
