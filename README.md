# Application Security Support Tools

アプリケーション開発者が、危険な処理を安全な処理へ置き換えるための **実行時セキュリティツール群** を提供するプロジェクトです。

このプロジェクトは、CI上で脆弱性を検出するスキャナを主目的にはしません。  
SQL実行、OSコマンド実行、ファイルアクセス、外部URLアクセスなど、攻撃につながりやすい **dangerous sink（危険な実行点）** の直前または代わりに入り、安全な構造・ポリシー・実行方法を強制することを目的とします。

## Core idea

```text
Application
    |
    v
Security Blocker
    |
    +-- Validate
    +-- Normalize
    +-- Apply Policy
    +-- Convert to a safe primitive
    |
    v
Dangerous Sink / External Resource
```

例として SQL Injection 対策では、入力文字列から「危険そうな文字」を削除するのではなく、SQL構造とデータを分離し、パラメータバインディングを強制します。

```text
SQL text + Parameters
        |
        v
    SQL Blocker
        |
        v
Prepared / Parameterized execution
        |
        v
      Database
```

## Goals

- 開発者が脆弱性ごとの細かな落とし穴を毎回実装しなくてよい状態を作る
- 「検出」ではなく、危険な操作を安全な操作へ変換・制約する
- CLI / ローカルAPI / 各言語ライブラリから同じ防御モデルを利用できるようにする
- 防御ロジックの重複実装を減らし、共通Coreへ集約する
- デフォルトは fail closed とし、安全性を確認できない操作は拒否できるようにする
- 「安全にできないものを安全と主張しない」ことを設計原則にする

## Non-goals

- 正規表現だけで攻撃文字列を判定する万能サニタイザ
- WAFの置き換え
- SAST / DAST / CI脆弱性診断の置き換え
- すべての入力に対して使える単一の `secure(input)` 関数
- アプリケーション全体の安全性を保証すること

## Planned blockers

| Blocker | 主な対象 | 基本戦略 |
|---|---|---|
| SQL Blocker | SQL Injection | SQL構造と値の分離、parameter binding |
| Command Blocker | OS Command Injection | executable と argv の分離、shell回避 |
| Path Blocker | Path Traversal | root containment、正規化、symlink方針 |
| URL Blocker | SSRF | scheme / host / DNS / IP / redirect policy |
| HTML Blocker | XSS | 出力コンテキスト別の安全なsink/encoding |
| Redirect Blocker | Open Redirect | 許可origin/pathの強制 |
| Header Blocker | Header / CRLF Injection | name/valueの構造化と検証 |
| File Blocker | 危険なupload/write | 保存先、名前、型、サイズ、権限の制約 |
| Log Blocker | Log Injection / secret leakage | 構造化ログ、改行・機密値ポリシー |
| Deserialize Blocker | Unsafe Deserialization | 許可型・形式の制約 |

## Delivery model

各Blockerは、原則として次の順で実装します。

```text
1. Reference Core
2. CLI / stdin-stdout API / daemon API
3. Attack & regression tests
4. Language SDKs
5. Native / embedded integration where useful
6. Next blocker
```

最初の実装対象は **SQL Blocker** です。

## Integration modes

### CLI / stdin-stdout

開発・検証・他言語からの簡易利用向けです。

```bash
echo '{"version":"1","blocker":"sql","operation":"prepare","input":{...}}' \
  | security-blocker
```

### Local daemon API

高頻度利用ではプロセスを毎回起動せず、常駐プロセスへ問い合わせます。

```text
Application
    |
    | Unix Domain Socket / localhost IPC
    v
security-blocker daemon
```

### Language SDK

最終的には各言語で自然なAPIを提供します。

```python
security.sql.execute(statement, params)
security.path.open(root, user_path)
security.url.fetch(url, policy)
```

SDKは可能な限り共通Coreを利用し、言語ごとに独自の防御判定を再実装しない方針です。

## Important security boundary

Blockerは「危険な文字列を見つけたら削除する」方式ではなく、**危険な操作の入力構造そのものを安全側へ限定する**ことを重視します。

また、統合方式によって保証範囲は異なります。

- Blocker自身が実行を所有する場合: 最も強い強制が可能
- SDKが安全なdriver APIを呼ぶ場合: 実用上の主要モード
- CLIが安全なexecution planだけを返す場合: 呼び出し側がそのplanを破壊せず利用する必要がある

この違いを文書上でもAPI上でも明示します。

## Documents

- [Architecture](docs/architecture.md)
- [Common Protocol](docs/common-protocol.md)
- [Roadmap](docs/roadmap.md)
- [SQL Blocker v0.1](docs/blockers/sql-v0.1.md)

## Status

初期設計フェーズです。  
現在は共通アーキテクチャと、最初のBlockerである SQL Blocker のv0.1仕様を固めています。
