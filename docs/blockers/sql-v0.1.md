# SQL Blocker v0.1

Status: **Draft**

SQL Blockerは、本プロジェクト最初のSecurity Blockerです。

目的は「SQL Injectionらしい文字列を検出する」ことではなく、**SQL構造とuntrusted dataを分離し、parameterized executionを強制すること**です。

---

## 1. Threat model

SQL Blocker v0.1が主に対象とするもの:

- user inputのSQL文字列への直接連結
- quote escapeに依存したSQL生成
- stacked statementの混入
- parameter count / placeholder misuse
- runtimeで任意SQLを組み立てる設計
- dynamic identifier経由の注入
- DB driverのunsafe string execution利用

代表的な危険例:

```python
sql = "SELECT * FROM users WHERE email = '" + email + "'"
cursor.execute(sql)
```

目標形:

```python
cursor.execute(
    "SELECT * FROM users WHERE email = ?",
    [email]
)
```

ただしSQL Blockerでは、単にこの書き方を推奨するだけでなく、Blocker APIとして構造分離を要求します。

---

## 2. Core security property

SQL Blockerが保証したい中心性質:

> untrusted dataがSQL syntaxとして解釈されず、DB driverへparameter valueとして渡されること。

この保証を成立させるため、SQL BlockerはSQL textとparameter valuesを別fieldとして扱います。

```text
SQL structure
    +
Parameters
    |
    v
SQL Blocker
    |
    v
Parameterized execution plan
```

---

## 3. Important limitation

次の入力だけを渡された場合:

```json
{
  "statement": "SELECT * FROM users WHERE id = 1 OR 1=1"
}
```

Blockerは、`OR 1=1` が開発者が意図したSQLなのか、user inputを連結した結果なのかを一般には判定できません。

したがって:

> **既にSQL textへ埋め込まれたuntrusted dataを、後段のfilterだけで完全に復元・無害化することはv0.1の保証対象外です。**

この問題に対して、v0.1では次の2モードを定義します。

1. **registered statement mode** — 推奨・強い保証
2. **inline parameterized mode** — 柔軟だがstatement textのtrust boundaryが必要

---

## 4. Registered statement mode

productionで最も推奨するモードです。

実行可能なSQL statementを、application configurationとして事前登録します。

例:

```yaml
statements:
  user.find_by_email:
    dialect: postgresql
    sql: "SELECT id, email, name FROM users WHERE email = $1"
    parameters:
      - type: string

  user.find_by_id:
    dialect: postgresql
    sql: "SELECT id, email, name FROM users WHERE id = $1"
    parameters:
      - type: integer
```

runtime requestではSQL textを渡しません。

```json
{
  "protocol_version": "1",
  "blocker": "sql",
  "blocker_version": "0.1",
  "operation": "prepare",
  "input": {
    "mode": "registered",
    "statement_id": "user.find_by_email",
    "parameters": [
      "alice@example.com"
    ]
  }
}
```

Blockerは登録済みstatementを取得し、parameter schemaを検証してexecution planを作ります。

### Security advantage

runtime inputが制御できるのはparameter valueだけです。

```text
untrusted request
    |
    +-- statement_id
    +-- parameters
             |
             v
       registered SQL
             |
             v
      parameter binding
```

statement ID自体も登録済み集合からしか選べません。

これにより、runtime requestからSQL構造を書き換える余地を大幅に減らします。

---

## 5. Inline parameterized mode

開発時・移行時・dynamic query builderとの統合向けです。

```json
{
  "protocol_version": "1",
  "blocker": "sql",
  "blocker_version": "0.1",
  "operation": "prepare",
  "input": {
    "mode": "inline",
    "dialect": "postgresql",
    "statement": "SELECT id, email FROM users WHERE email = $1",
    "parameters": [
      "alice@example.com"
    ]
  }
}
```

このモードでもparameter bindingを強制しますが、次の前提があります。

> `statement` はdeveloper-controlledであり、untrusted valueを文字列連結済みではないこと。

この前提を満たせない場合、registered modeを使用すべきです。

---

## 6. Request schema

v0.1 draft:

```json
{
  "protocol_version": "1",
  "request_id": "req-123",
  "blocker": "sql",
  "blocker_version": "0.1",
  "operation": "prepare",
  "input": {
    "mode": "registered",
    "statement_id": "user.find_by_email",
    "parameters": []
  },
  "policy": {}
}
```

### `input.mode`

- `registered`
- `inline`

### Registered mode fields

| Field | Required |
|---|---:|
| `statement_id` | yes |
| `parameters` | yes |

### Inline mode fields

| Field | Required |
|---|---:|
| `dialect` | yes |
| `statement` | yes |
| `parameters` | yes |

---

## 7. Dialects

初期対象候補:

- PostgreSQL
- MySQL / MariaDB
- SQLite

v0.1では、dialectごとにplaceholder semanticsを分離します。

例:

```text
PostgreSQL: $1, $2, ...
MySQL:      ?, ?, ...
SQLite:     ?, ?, ...
```

実装では、placeholder countingやstatement boundary検出を単純な正規表現だけで行わず、dialect-aware parserまたは十分に限定したtokenizerを利用する方針です。

---

## 8. Parameters

parameterはSQL textとは別に渡します。

基本的なJSON value:

- string
- integer
- number
- boolean
- null

binary / date / timestamp / decimalなど、JSONだけでは型が曖昧な値にはtyped valueを検討します。

候補形式:

```json
{
  "type": "timestamp",
  "value": "2026-09-29T12:34:56Z"
}
```

または:

```json
{
  "$type": "binary",
  "encoding": "base64",
  "value": "..."
}
```

typed valueの最終schemaは実装前に固定します。

---

## 9. Identifier handling

SQL parameter bindingでは、通常table名やcolumn名などのidentifierをparameterとして渡せません。

危険例:

```python
sql = "SELECT * FROM " + user_table
```

v0.1では、**inline modeでuntrusted dynamic identifierを直接受け取る機能は提供しません。**

推奨:

- registered statementを使う
- application側でknown constantへmapする
- query variationをstatement IDとして分ける

例:

```text
sort=name  -> user.list.sort_name
sort=date  -> user.list.sort_date
```

将来versionでは、allowlist + dialect-aware identifier quotingを持つstructured identifier slotを検討します。

---

## 10. Statement restrictions

default policyでは以下を採用します。

### Single statement only

1 requestで1 SQL statementのみ許可します。

```text
SELECT ...; DROP TABLE ...
```

のようなstacked statementを拒否します。

### Empty statement deny

空statementは拒否します。

### Unsupported syntax deny

parserが理解できない、または対象dialectとして安全に処理できない構文はfail closedします。

### Parameter mismatch deny

placeholder数とparameter数が一致しない場合は拒否します。

### No value interpolation

Blocker自身がparameter valuesをSQL literalへ文字列変換して埋め込む機能は提供しません。

つまり次のようなoutputは作りません。

```text
SELECT * FROM users WHERE email = 'alice@example.com'
```

outputは常にSQL structureとparametersを分離したまま維持します。

---

## 11. Comments

SQL comment自体を一律禁止することはしません。

```sql
SELECT id
FROM users
-- active users only
WHERE active = $1
```

commentは正規のSQL syntaxでもあるため、「`--` が含まれているから攻撃」といった単純判定は行いません。

registered statement modeでは、commentを含むstatementも登録可能です。

必要であればapplication policyでcomment禁止を追加できます。

---

## 12. Policy draft

例:

```json
{
  "allow_inline_statements": false,
  "allow_multiple_statements": false,
  "allow_unknown_statement_ids": false,
  "max_statement_bytes": 65536,
  "max_parameters": 256,
  "allowed_statement_ids": [
    "user.find_by_email",
    "user.find_by_id"
  ]
}
```

### Secure defaults

推奨default:

```text
allow_inline_statements      = false in strict production profile
allow_multiple_statements    = false
allow_unknown_statement_ids  = false
max_statement_bytes          = bounded
max_parameters               = bounded
```

開発profileではinlineを許可できますが、production profileはregistered mode優先とします。

---

## 13. Response

prepare成功例:

```json
{
  "protocol_version": "1",
  "request_id": "req-123",
  "blocker": "sql",
  "blocker_version": "0.1",
  "decision": "allow",
  "allowed": true,
  "output": {
    "execution_plan": {
      "dialect": "postgresql",
      "statement": "SELECT id, email, name FROM users WHERE email = $1",
      "parameters": [
        "alice@example.com"
      ],
      "statement_id": "user.find_by_email"
    }
  },
  "warnings": [],
  "metadata": {
    "mode": "registered"
  }
}
```

重要:

`execution_plan.statement` と `execution_plan.parameters` を結合してはいけません。

SDK / adapterはこれらを別引数のままnative driverへ渡します。

---

## 14. Execution model

### v0.1 CLI: prepare / gate

CLI reference implementationでは、まず安全なexecution planの生成と拒否判定を担当します。

```text
Application
    |
    v
SQL Blocker CLI / daemon
    |
    v
allow + execution plan
    |
    v
Application adapter
    |
    v
DB driver parameter binding
```

これは runtime gate として利用できますが、呼び出し側がexecution planを破壊しないことが必要です。

### SDK: execute

言語SDKでは、より強い形を目指します。

```text
Application
    |
    v
security.sql.execute(...)
    |
    +--> SQL Blocker Core
    |
    +--> native DB driver
    |
    v
Database
```

SDKがdriver呼び出しまで所有することで、parameterized executionを強制します。

### Future daemon executor

将来的には、DB connection profileをBlocker daemon側へ持たせ、Blocker自身が実行を所有するモードも検討します。

ただし次の設計課題があります。

- credential management
- transaction ownership
- connection pooling
- network boundary
- failure semantics
- observability

そのためv0.1では必須にしません。

---

## 15. Error codes draft

### Protocol / schema

- `SQL_INVALID_REQUEST`
- `SQL_UNSUPPORTED_VERSION`
- `SQL_UNSUPPORTED_DIALECT`
- `SQL_UNSUPPORTED_MODE`

### Statement

- `SQL_EMPTY_STATEMENT`
- `SQL_STATEMENT_TOO_LARGE`
- `SQL_MULTIPLE_STATEMENTS_NOT_ALLOWED`
- `SQL_UNSUPPORTED_SYNTAX`
- `SQL_INLINE_STATEMENT_NOT_ALLOWED`

### Registered statements

- `SQL_UNKNOWN_STATEMENT_ID`
- `SQL_STATEMENT_ID_NOT_ALLOWED`
- `SQL_STATEMENT_REGISTRY_ERROR`

### Parameters

- `SQL_PARAMETER_COUNT_MISMATCH`
- `SQL_PARAMETER_TYPE_MISMATCH`
- `SQL_TOO_MANY_PARAMETERS`
- `SQL_UNSUPPORTED_PARAMETER_TYPE`

### Execution

- `SQL_ADAPTER_ERROR`
- `SQL_EXECUTION_TIMEOUT`
- `SQL_EXECUTION_CANCELLED`

---

## 16. CLI examples

### Registered statement

```bash
printf '%s\n' '{
  "protocol_version":"1",
  "blocker":"sql",
  "blocker_version":"0.1",
  "operation":"prepare",
  "input":{
    "mode":"registered",
    "statement_id":"user.find_by_email",
    "parameters":["alice@example.com"]
  }
}' | security-blocker
```

### Inline statement

```bash
printf '%s\n' '{
  "protocol_version":"1",
  "blocker":"sql",
  "blocker_version":"0.1",
  "operation":"prepare",
  "input":{
    "mode":"inline",
    "dialect":"sqlite",
    "statement":"SELECT id FROM users WHERE email = ?",
    "parameters":["alice@example.com"]
  }
}' | security-blocker
```

inline modeはpolicyで明示的に許可されている場合のみ利用可能にできます。

---

## 17. Attack behavior examples

parameterとして次のような文字列が来ても:

```text
' OR 1=1 --
```

BlockerはSQL textへ埋め込みません。

```text
statement:
  SELECT id FROM users WHERE email = ?

parameter[0]:
  ' OR 1=1 --
```

driverへ別々に渡されるため、この値はSQL syntaxではなくdataとして扱われることを期待します。

一方、callerが最初から次のSQL textを作ってしまった場合:

```sql
SELECT id FROM users WHERE email = '' OR 1=1 --'
```

inline modeだけでは、それが「意図されたSQL」か「注入後のSQL」かを完全には判断できません。

この差がregistered modeを用意する理由です。

---

## 18. Test categories

### Positive

- string parameter
- numeric parameter
- boolean parameter
- null parameter
- Unicode
- long but allowed values
- valid comments
- valid dialect-specific syntax

### Negative

- unknown statement ID
- forbidden inline mode
- multiple statements
- placeholder mismatch
- malformed SQL
- unsupported dialect
- oversized statement
- too many parameters
- wrong registered parameter type

### Injection-oriented regression

- quote payloads
- boolean expression payloads
- comment payloads
- semicolon payloads
- encoded quote variants
- Unicode edge cases
- null byte handling
- dialect-specific escaping edge cases

重要なのは「payloadを見つけたらblock」ではなく、**payloadがparameterのままSQL syntaxへ昇格しないこと**を確認することです。

---

## 19. Success criteria for v0.1

SQL Blocker v0.1は、少なくとも以下を満たすことを目標とします。

- common protocolでrequestを受けられる
- registered statement modeが動く
- inline parameterized modeが動く
- dialect-awareにsingle statementを確認できる
- parameter count/typeを確認できる
- parameter valuesをSQL textへrenderしない
- fail-closedでunsupported inputを拒否する
- JSONL persistent modeで利用できる
- security test vectorが用意されている
- SDKが同じexecution plan semanticsを利用できる

---

## 20. Open design questions

実装開始前に決める項目:

1. Core実装言語
2. SQL parser/tokenizerの選定
3. statement registryのformat
4. typed parameter schema
5. policy file format
6. Unix Domain Socket / local HTTPの優先順位
7. dialectごとのplaceholder normalization方針
8. transactionとprepared statement cacheをCoreの責務に含めるか
9. daemon executorをいつ導入するか
10. SDKでraw inline SQL APIをどこまで許可するか

この文書をSQL Blocker v0.1の設計議論の起点とします。
