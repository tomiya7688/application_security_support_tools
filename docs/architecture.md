# Architecture

## 1. Purpose

Application Security Support Tools は、アプリケーションの危険な実行点（dangerous sink）を、安全なプリミティブへ置き換えるための共通Security Blocker基盤です。

対象は、たとえば以下です。

- SQL実行
- OSコマンド実行
- ファイルパス解決
- 外部URLへのHTTPアクセス
- HTML/DOM出力
- HTTP redirect
- HTTP header生成
- ファイルupload/write
- ログ出力
- デシリアライズ

目的は、攻撃文字列を推測して弾くことではなく、**危険な操作を安全な構造でしか実行できないようにすること**です。

---

## 2. Architectural principle

基本形は以下です。

```text
Application
    |
    v
Structured Security Request
    |
    v
+--------------------------+
| Security Blocker Core    |
|--------------------------|
| Parse                    |
| Validate                 |
| Normalize                |
| Apply policy             |
| Build safe primitive     |
+--------------------------+
    |
    v
Execution Adapter
    |
    v
Dangerous Sink
```

Blocker Coreは「文字列が怪しいか」ではなく、入力を構造化して意味を理解し、安全な実行形式へ落とします。

---

## 3. Components

### 3.1 Core

全Blockerで共有する中心実装です。

責務:

- protocol versionの検証
- request / responseの共通形式
- policy適用
- error taxonomy
- audit metadata
- blocker別ロジックのdispatch
- fail-closed制御
- normalizationルール
- compatibility管理

Coreは、可能な限り言語SDKから共有します。

Coreのparser / framing / serializer / error pathはSecurity Blocker自身のTrusted Computing Baseに含まれます。詳細要件は [Input / Output Security](input-output-security.md) を参照してください。

### 3.2 Blocker modules

各脆弱性・dangerous sinkに特化したモジュールです。

例:

```text
core/
blockers/
    sql/
    command/
    path/
    url/
    html/
    redirect/
    header/
    file/
```

重要なのは、万能な `secure(input)` を作らないことです。

SQL、Shell、Path、URL、HTMLはそれぞれ異なる文法・実行モデル・安全境界を持つため、Blockerごとに安全なprimitiveを定義します。

### 3.3 Execution adapters

Blockerが作った安全な構造を、実際のdriverやOS APIへ接続します。

例:

```text
SQL Blocker
    |
    +-- PostgreSQL adapter
    +-- MySQL adapter
    +-- SQLite adapter
```

```text
Command Blocker
    |
    +-- Unix process adapter
    +-- Windows process adapter
```

### 3.4 CLI

Reference implementationとして提供します。

想定用途:

- 手動テスト
- shell pipeline
- integration test
- 他言語からの簡易呼び出し
- protocol検証

高頻度のproduction requestごとに新規processを起動することは主用途にしません。

### 3.5 Daemon

常駐processとしてCoreへアクセスする方式です。

候補transport:

- Unix Domain Socket
- Windows named pipe
- localhost HTTP
- stdio persistent session

productionでは、外部ネットワークへ公開しないローカルIPCを優先します。

### 3.6 Language SDKs

各言語の開発者が自然なAPIで使うための薄いwrapperです。

例:

```python
security.sql.execute(...)
security.command.run(...)
security.path.open(...)
```

SDKの役割:

- 言語らしいAPI
- 型変換
- native driver adapter
- lifecycle管理
- error mapping
- Core接続

防御ロジックをSDKごとに再実装することは避けます。

---

## 4. Enforcement levels

すべての統合方式が同じ強度ではありません。

### Level A: Owned execution

Blocker自身がdangerous sinkの実行まで所有します。

```text
Application -> Blocker -> Database / OS / Network
```

最も強い方式です。Blockerを通過しない限り対象操作を実行できない構成にできます。

### Level B: SDK-owned execution

SDKがBlocker Coreの結果を利用し、native driverの安全APIへ接続します。

```text
Application -> SDK -> Core -> native safe API
```

多くの言語で現実的な主要方式です。

### Level C: Plan-only

CLI/APIが安全なexecution planを返し、呼び出し側が実行します。

```text
Application -> Blocker -> safe plan -> Application -> Sink
```

統合の自由度は高い一方、呼び出し側がplanを文字列結合などで破壊できるため、Level A/Bより保証は弱くなります。

APIと文書では、どのLevelの保証なのかを明示します。

---

## 5. Core security rules

### 5.1 Fail closed

安全性を確認できない場合は、原則として許可しません。

```text
unknown -> deny
ambiguous -> deny
unsupported -> deny
invalid policy -> deny
```

必要な互換モードを用意する場合でも、明示的opt-inにします。

### 5.2 Structure over sanitization

危険文字列の削除より、構造分離を優先します。

例:

- SQL: query + parameters
- Command: executable + argv
- Path: root + relative path
- URL: parsed URL + resolved destination + policy
- Header: header name + typed value

### 5.3 No silent mutation

入力を勝手に「それっぽく安全に修正」すると、アプリケーションの意味が変わる危険があります。

原則:

- 安全にnormalizeできるものだけnormalize
- 意味が変わる可能性がある場合はreject
- mutationを行った場合はresponse metadataで明示

### 5.4 Explicit policy

安全要件は暗黙にせずpolicyとして表現します。

例:

```json
{
  "allow_schemes": ["https"],
  "allow_private_networks": false,
  "max_redirects": 2
}
```

### 5.5 No false guarantee

Blockerが制御できない領域は、制御できるように見せません。

例:

- SQL文字列に既にuser inputが埋め込まれている場合、それを一般的に完全復元することはできない
- Pathのrace conditionは文字列canonicalizationだけでは完全に防げない
- SSRFはURL文字列検証だけでは不十分
- HTML encodingは出力contextによって異なる

その場合はAPIをより強い構造へ変更するか、保証範囲を明示します。

---

## 6. Common processing pipeline

```text
1. Enforce raw byte / frame / time limits
2. Validate framing
3. Strict UTF-8 decode
4. Strict JSON parse (duplicate keys rejected)
5. Validate common protocol schema
6. Validate blocker-specific schema
7. Normalize blocker-defined structural values only
8. Resolve policy
9. Perform blocker-specific checks
10. Build safe primitive / execution plan
11. Execute or return plan
12. Serialize bounded structured result
13. Emit redacted audit metadata
```

Blockerごとに必要な処理を追加しますが、この流れを共通化します。

---

## 7. Error model

errorは機械判定可能なcodeを持ちます。

例:

```json
{
  "allowed": false,
  "error": {
    "code": "SQL_IDENTIFIER_NOT_ALLOWED",
    "message": "Requested identifier is not permitted by policy.",
    "field": "input.identifiers.table"
  }
}
```

error codeは互換性のあるpublic APIとして扱います。

分類例:

- `PROTOCOL_*`
- `POLICY_*`
- `SQL_*`
- `COMMAND_*`
- `PATH_*`
- `URL_*`
- `EXECUTION_*`
- `INTERNAL_*`

---

## 8. Auditability

production利用では「なぜ拒否されたか」を調査できる必要があります。

ただし、audit logそのものから秘密情報が漏れないようにします。

原則:

- secret/token/passwordのraw値を記録しない
- SQL parameterのraw値はdefaultで記録しない
- user input全文をdefaultで記録しない
- policy ID / error code / blocker / operation / timingは記録可能
- debug modeは明示的に有効化

---

## 9. Performance model

Security Blockerはdangerous sinkの直前で呼ばれるため、性能は重要です。

優先順位:

1. in-process SDK
2. persistent local IPC
3. persistent stdio
4. one-shot CLI

one-shot processはreference/test用途とし、高頻度production pathでは避けます。

---

## 10. Versioning

protocolとBlocker仕様は別々にversion管理します。

例:

```json
{
  "protocol_version": "1",
  "blocker": "sql",
  "blocker_version": "0.1"
}
```

後方互換性を壊す変更はmajor versionで管理します。

---

## 11. Initial implementation strategy

最初は SQL Blocker で共通設計を検証します。

```text
Phase 1
  Core
  + common protocol
  + SQL Blocker
  + CLI
  + persistent local mode
  + test vectors

Phase 2
  Python SDK
  Node.js SDK
  additional language SDKs

Phase 3
  Command Blocker

Phase 4+
  Path / URL / HTML / ...
```

SQL Blockerで得られた設計知見を、次のBlockerへ反映します。
