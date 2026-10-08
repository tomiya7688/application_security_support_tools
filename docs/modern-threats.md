# Modern Threats and Late-Stage Guards

## 1. Position of this project

Application Security Support Tools は、すべての攻撃を入口で検知する製品ではありません。

このプロジェクトの主要な責務は **late-stage guard** です。

つまり、入力検証、認証、認可、WAF、SAST、EDRなどをすり抜けた場合でも、危険な副作用が発生する直前に、実行可能な操作を構造化・制限・拒否します。

```text
Untrusted Input
      |
      v
Application Logic
      |
      |  earlier defenses may fail
      v
Dangerous Action
      |
      v
+-----------------------+
| Late-Stage Blocker    |
|-----------------------|
| capability check      |
| destination policy    |
| structure enforcement |
| resource budget       |
| data egress policy    |
+-----------------------+
      |
      v
OS / DB / Network / File / Tool / Parser
```

このため、Blockerは「攻撃文字列を当てる」ことよりも、「実行できる能力を狭くする」ことを優先します。

---

## 2. Modern threat design rule

新しい攻撃カテゴリを追加するときは、最初に次を問います。

> 攻撃者が最終的に利用したい dangerous sink / side effect は何か？

例:

| Attack class | 最終的な副作用 |
|---|---|
| Prompt Injection | tool execution / data exfiltration / external request |
| SSRF | attacker-controlled outbound network access |
| RCE chain | process execution / dynamic loading |
| Zip Slip | filesystem write outside intended root |
| Decompression bomb | CPU / memory / disk exhaustion |
| SSTI | template engineによるcode/data execution |
| Unsafe Deserialization | object construction / code execution |
| Mass Assignment | sensitive object field update |
| GraphQL abuse | excessive resolver / DB / CPU consumption |
| Secret exfiltration | untrusted destinationへのcredential送信 |

攻撃名そのものを判定するのではなく、その副作用を所有するBlockerを設計します。

---

## 3. High-value modern blockers

### 3.1 Agent Action Blocker

対象:

- Prompt Injection経由の危険なtool call
- indirect prompt injection
- agentによる過剰な権限利用
- tool chainingによる権限拡大
- model-generated command / URL / file operation

Agent Action Blockerは「Prompt Injectionを完全検出する」ものではありません。

代わりに、LLM/Agentが生成したactionを **untrusted execution proposal** とみなし、実行前に検証します。

```text
LLM / Agent
    |
    | proposed action
    v
Agent Action Blocker
    |
    +-- allowed tool?
    +-- allowed arguments?
    +-- capability available?
    +-- destination allowed?
    +-- sensitive data included?
    +-- approval required?
    |
    v
Tool Executor
```

例:

```json
{
  "blocker": "agent_action",
  "operation": "execute",
  "input": {
    "tool": "http.fetch",
    "arguments": {
      "url": "https://example.com/"
    }
  },
  "context": {
    "agent_id": "support-agent"
  }
}
```

基本戦略:

- tool allowlist
- argument schema
- capability-based permission
- per-tool resource limits
- network / filesystem / process Blockerとのcomposition
- sensitive-action approval hooks
- untrusted model outputを直接shell等へ渡さない

このBlockerは、Command / URL / Path / Egress Blockerの上位orchestratorとして実装できます。

---

### 3.2 Egress Blocker

対象:

- credential / token exfiltration
- SSRFを利用したデータ持ち出し
- Agent / plugin経由の情報漏えい
- 誤ったtelemetry送信
- secretsを含むwebhook / HTTP request

dangerous sink:

```text
application -> outbound network
```

基本戦略:

- destination allowlist / policy
- scheme / host / resolved IP policy
- redirect再評価
- header/bodyのsecret detection or labeled-data policy
- credential scope制限
- request size limit
- sensitive destination requiring approval

重要:

単純なsecret regexだけに依存しません。

SDKがtokenやcredentialをtyped valueとして扱える場合は、security labelを保持したままEgress Blockerへ渡す方が強い設計です。

例:

```text
value: API token
label: secret
allowed destinations:
  - api.example.com
```

---

### 3.3 Resource Budget Blocker

対象:

- ReDoS
- decompression bomb
- parser bomb
- oversized payload
- GraphQL query complexity abuse
- expensive template/query execution
- algorithmic complexity attack

Blockerが管理するbudget例:

- wall-clock timeout
- CPU time
- memory
- output bytes
- input bytes
- recursion depth
- nesting depth
- result count
- regex steps/time
- decompressed bytes
- archive entry count

```text
Untrusted Work
    |
    v
Resource Budget Blocker
    |
    +-- time budget
    +-- memory budget
    +-- expansion budget
    +-- depth budget
    |
    v
Parser / Regex / Query / Worker
```

単なるvalidationではなく、可能ならsandbox / worker process / cancellationと組み合わせます。

---

### 3.4 Archive Extraction Blocker

対象:

- Zip Slip
- absolute-path extraction
- symlink extraction
- archive bomb
- nested archive bomb
- huge entry count
- special file extraction

API例:

```text
security.archive.extract(
    source,
    destination_root,
    policy
)
```

基本戦略:

- extraction root containment
- path canonicalization
- absolute path拒否
- symlink / hardlink policy
- special file拒否
- entry count limit
- compressed / uncompressed size limit
- expansion ratio limit
- nesting depth limit

ファイル名をsanitizeするだけではなく、**実際のwrite sinkをBlockerが所有する**設計を優先します。

---

### 3.5 Template Blocker

対象:

- Server-Side Template Injection (SSTI)
- unsafe expression evaluation
- templateからのfilesystem/network/process access

基本戦略:

- templateとdataの分離
- untrusted template executionをdefault deny
- expression subset
- object capability制限
- filesystem/network access禁止
- execution timeout / output limit

理想形:

```text
trusted template + untrusted data
            |
            v
       Template Blocker
            |
            v
      constrained render
```

「危険なテンプレート構文をregexで削除」する方式にはしません。

---

### 3.6 Object Update Blocker

対象:

- Mass Assignment
- Over-posting
- prototype pollutionにつながるunsafe object merge
- privilege field overwrite
- internal field mutation

dangerous sink:

```text
untrusted object -> domain model update
```

例:

```json
{
  "blocker": "object_update",
  "input": {
    "schema": "profile.update",
    "values": {
      "display_name": "Alice",
      "is_admin": true
    }
  }
}
```

policy:

```text
profile.update:
  allowed:
    - display_name
    - avatar_url
```

`is_admin` はsinkへ到達する前に拒否します。

JavaScript系では `__proto__`, `constructor`, `prototype` 等を含むproperty semanticsも別途考慮します。

---

### 3.7 Deserialize Blocker

既存候補を現代的な実装へ拡張します。

対象:

- arbitrary type instantiation
- gadget chain
- unsafe polymorphic deserialization
- YAML/object tag abuse
- schema confusion

基本戦略:

- data-only format優先
- explicit schema
- type allowlist
- polymorphism default deny
- depth / size budget
- object constructionとbusiness object変換の分離

```text
bytes
  |
  v
data-only parse
  |
  v
schema validation
  |
  v
explicit conversion
```

---

### 3.8 Parser Sandbox Blocker

対象:

- image/document/media parser vulnerabilities
- malicious PDF/image/archive/file format
- parser crashes
- memory corruption impact containment

これは他のBlockerより重いですが、late-stage guardとして価値があります。

構想:

```text
Untrusted File
      |
      v
Parser Sandbox Blocker
      |
      +-- isolated worker
      +-- no network
      +-- read-only input
      +-- bounded memory
      +-- bounded CPU/time
      +-- sanitized output
      |
      v
Parser
```

「ファイルが安全か」を完全判定するより、parser exploitが起きたときの権限を最小化します。

---

### 3.9 Webhook / Replay Blocker

対象:

- forged webhook
- replay attack
- stale signed request
- duplicated financial/event action

基本戦略:

- signature verification
- timestamp window
- nonce/event ID replay cache
- canonical signing input
- body size limit
- algorithm policy

これは入口側に近いBlockerですが、「副作用を起こすhandlerの直前に置く」場合は本プロジェクトの思想と適合します。

---

### 3.10 Dynamic Load Blocker

対象:

- attacker-controlled module/plugin load
- unsafe dynamic import
- DLL/shared library hijacking
- extension execution

基本戦略:

- approved module IDs
- canonical path
- signature/hash policy
- trusted roots
- no search-path ambiguity
- explicit version/integrity binding

---

## 4. Composite guards

現代的な攻撃は、単一Blockerよりcompositionが重要です。

### Agent example

```text
LLM
 |
 v
Agent Action Blocker
 |
 +--> URL Blocker
 |       |
 |       +--> Egress Blocker
 |
 +--> Command Blocker
 |
 +--> Path Blocker
 |
 +--> Object Update Blocker
```

Prompt Injectionそのものを完全に見抜けなくても、攻撃者が最終的に使える能力を狭められます。

### File processing example

```text
Upload
 |
 v
File Blocker
 |
 v
Archive Extraction Blocker
 |
 v
Parser Sandbox Blocker
 |
 v
Resource Budget Blocker
```

---

## 5. Threats that are poor fits

すべてのセキュリティ問題をBlocker化すべきではありません。

### Broken Access Control

一部はObject Update BlockerやCapability modelで補助できますが、アプリケーション固有のauthorization全体を後段Blockerだけで解決することはできません。

### Authentication failures

password / session / MFAなどは専用authentication systemの責務が大きく、本プロジェクトの主要sink modelとは異なります。

### HTTP Request Smuggling / Desync

HTTP stack / proxy / server間のparser差異が中心で、一般的なアプリケーションライブラリとして完全防御するのは難しい領域です。

### Supply-chain compromise

artifact integrityやdynamic loadingを一部防御できますが、dependency selectionやbuild pipeline全体は別レイヤーです。

### Business logic abuse

操作頻度・順序・権限などのdomain policyが必要で、汎用Blockerだけでは十分ではありません。

---

## 6. Priority based on late-stage value

実装優先度を「流行している攻撃名」ではなく、「最後の危険な副作用をどれだけ確実に止められるか」で決めます。

### Tier A — strong fit

- SQL Blocker
- Command Blocker
- Path Blocker
- URL Blocker
- Egress Blocker
- Archive Extraction Blocker
- Object Update Blocker
- Deserialize Blocker
- Resource Budget Blocker

### Tier B — strong but more complex

- Agent Action Blocker
- Template Blocker
- Parser Sandbox Blocker
- Dynamic Load Blocker
- File Blocker

### Tier C — supporting / context-dependent

- Webhook / Replay Blocker
- Header Blocker
- Redirect Blocker
- HTML Blocker
- Log Blocker

---

## 7. Core principle for AI-era security

AI/Agent対応でも、モデルが「安全な判断をする」ことをsecurity boundaryにはしません。

```text
LLM output = untrusted proposal
```

として扱います。

つまり:

- modelがURLを生成してもURL Blockerを通す
- modelがcommandを生成してもCommand Blockerを通す
- modelがfile pathを生成してもPath Blockerを通す
- modelがtool callを生成してもAgent Action Blockerを通す
- modelが外部へdataを送ろうとしてもEgress Blockerを通す

これにより、Prompt Injectionやmodel behaviorの完全な検知に依存せず、実世界への副作用を制約できます。

---

## 8. Guiding statement

このプロジェクトにおける modern security blocker の基本方針:

> Do not try to perfectly recognize the attack. Own the dangerous action and constrain what it can do.

攻撃を100%識別できなくても、危険なsinkを所有し、能力・宛先・データ・資源を制約できれば、last-line defenseとして意味のある安全性を提供できます。
