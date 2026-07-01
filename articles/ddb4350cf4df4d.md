---
title: "# Claude Code マルチエージェントで「非介入」を強制する仕組みを作った"
emoji: "🍔"
type: "tech"
topics:
  - "hook"
  - "automation"
  - "workflow"
  - "multiagent"
  - "claudecode"
published: true
published_at: "2026-02-19 14:00"
---

## はじめに

Claude Code でマルチエージェント構成を使い始めると、ある問題に直面する。

**Claude が「勝手に内容を読んで要約してしまう」問題だ。**

``` ← ここが問題
         ↓ 内容を把握・要約してから...
[エージェントB] → "Claudeが要約した内容" を受け取る
```

エージェント間の受け渡しに Claude が介入することで、

- 出力が要約・改変されるリスク
- 意図しない情報フィルタリング
- 「なぜこの結果になったか」のトレーサビリティが失われる

といった問題が生じる。

本記事では、**Hook を使って Claude の介入を検知・警告する仕組み** と、**ワークフローをパッケージ化する「Happy Set」パターン** を紹介する。

---

## 前提：マルチエージェント構成

今回の構成は以下の通り：

```
[スコープ作成エージェント]
    → テスト計画 MD + JSON を生成
    → レビューエージェントを内部で呼び出し（サイクル）
    → ドキュメントシステムへ公開
         ↓ JSON ファイルパスを出力
[見積もりエージェント]
    → JSON を読み込み
    → DITA ベースの工数見積もりを算出
```

この連鎖を Claude Code の `Task` ツールで構築している。

---

## 問題：Claude が「中間でファイルを読む」

Set C（スコープ作成 → 見積もり）を実行すると、Claude は自然に以下をやろうとする：

```
① Task(scope-agent, "PROJ-1234 のスコープを作成して")
   → 完了、出力: output.json

② Read("output.json")  ← ⚠️ ここが介入
   → "scope には5項目あります。beta/rc/real 対象で..."

③ Task(estimation-agent, "以下のスコープを見積もって\n[要約されたテキスト]")
```

これは直感的だが、**非介入原則に違反している**。

エージェントAの出力を、Claudeが仲介することで内容が変わる可能性がある。特に「100行のJSONを口頭で要約してから次エージェントに渡す」のは明らかに情報が欠落する。

---

## 解決策1：Handoff Protocol（ファイルパスのみ受け渡し）

**正しいパターン：**

```
① Task(scope-agent, "PROJ-1234 のスコープを作成して")
   → 完了。JSON_PATH: /result/Test_Scope_PROJ-1234.json

② ← Claude は Read を呼ばない ←

③ Task(estimation-agent, "Input: /result/Test_Scope_PROJ-1234.json")
   → estimation-agent が直接ファイルを読む
```

Claude の役割は **「パス文字列を次のプロンプトに転記する」だけ**。

### ルール

| ✅ Claude が行うこと | ❌ Claude がやってはいけないこと |
|---------------------|-------------------------------|
| ファイルパスを抽出する | ファイル内容を読む |
| パスをそのまま次のプロンプトに渡す | 内容を要約・検証する |
| 出力をユーザーにそのまま報告する | MCP で成果物を確認する |

---

## 解決策2：Hook による介入検知

ルールを文書化するだけでは Claude に守らせるのは難しい（Claude は「便利なことをやろうとする」）。

そこで **2段階の Hook** で機械的に検知する。

### Phase 1：PostToolUse(Task) — パスを「監視リスト」に登録

```python
# happy-set-intervention-detector.py（PostToolUse）

def handle_post_tool_use_task(input_data):
    tool_response = input_data.get("tool_response", {})
    output_text = extract_text(tool_response)

    # Task 出力から JSON ファイルパスを抽出
    json_paths = extract_json_paths(output_text)

    if json_paths:
        # /tmp/.happy_set_watched に記録（TTL: 5分）
        add_watched_paths(json_paths)
```

エージェントAのタスクが完了した直後、出力テキストから `Test_Scope_PROJ-XXXX.json` のようなパスを正規表現で拾い出し、一時ファイルに記録する。

### Phase 2：PreToolUse(Read) — 介入を検知して警告

```python
def handle_pre_tool_use_read(input_data):
    file_path = input_data["tool_input"]["file_path"]

    # Test Scope JSON ファイルか？
    if not SCOPE_JSON_PATTERN.search(file_path):
        sys.exit(0)

    # 監視リストに入っているか？
    if is_watched_path(file_path):
        print("""
⚠️  INTERVENTION DETECTED - Non-Intervention Violation
Claude is attempting to READ an agent output file directly.
...
        """, file=sys.stderr)
        sys.exit(1)  # warn but allow
```

Claude が `Read(output.json)` を呼ぼうとした瞬間、stderr に警告が出る。

### settings.json への登録

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{
          "type": "command",
          "command": "python3 ~/.claude/hooks/happy-set-intervention-detector.py"
        }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Task",
        "hooks": [{
          "type": "command",
          "command": "python3 ~/.claude/hooks/happy-set-intervention-detector.py"
        }]
      }
    ]
  }
}
```

同じスクリプトを PreToolUse / PostToolUse 双方にアタッチし、`hook_event_name` で分岐する。

---

## 解決策3：Happy Set パターン

毎回「まず スコープ作成エージェント に投げて、次に 見積もりエージェント に投げて...」と手動で指示するのは面倒。

よく使うエージェント連鎖を **「ハッピーセット」** としてパッケージ化する。

| セット | 内容 | 呼び出し |
|--------|------|---------|
| **Set A** | スコープ作成（Full） | `/happy-set-a PROJ-1234 v1.0` |
| **Set B** | 見積もりのみ | `/happy-set-b PROJ-1234` |
| **Set C** | スコープ作成 + 見積もり | `/happy-set-c PROJ-1234 v1.0` |

### skill YAML の実装

```yaml
# .claude/skills/happy-set-c.yaml
---
name: happy-set-c
description: Happy Set C - Full pipeline (Scope + Estimation)
allowed-tools: Task   # ← Task のみに制限
---

## Phase 1: スコープ作成

Task(scope-agent):
  "PROJ-{JIRA_ID} v{VERSION} のスコープを作成して。
   完了時に JSON_PATH: {絶対パス} 形式で出力すること。"

## Handoff（パス抽出のみ）

スコープ作成エージェントの出力から JSON_PATH を抽出する。
Read ツールは呼ばない。

## Phase 2: 見積もり

Task(estimation-agent):
  "Input JSON: {JSON_PATH} を直接読んで見積もりを実行して。"
```

`allowed-tools: Task` により、スキル実行中に Claude が `Read` や `Edit` を呼べなくなる。

---

## 設計のポイント

### 1. エージェントの「出力契約」を決める

介入検知 Hook が機能するには、エージェントが **予測可能な形式でパスを出力** する必要がある。

```
# エージェントが出力する標準フォーマット
完了しました。
JSON_PATH: /Users/.../Test_Scope_PROJ-1234.json
```

`JSON_PATH:` マーカーが検知の起点になる。

### 2. TTL で誤検知を防ぐ

監視パスには 5分間の TTL を設定している。長時間セッションで「過去のファイルを編集用に読む」場合に誤検知しないための措置。

```python
WATCH_TTL_SECONDS = 300  # 5 minutes

watched = {
    path: ts for path, ts in data.items()
    if now - ts < WATCH_TTL_SECONDS  # TTL 以内のみ有効
}
```

### 3. exit 1（warn）vs exit 2（block）

今回は介入を **ブロックせず警告のみ** にしている（exit 1）。

理由は、「ファイルを読む正当な理由」が存在する可能性があるから（デバッグ、トラブルシュート等）。

強制したい場合は exit 2 に変えれば完全にブロックできる。

---

## 実際の動作

### ✅ 正常ケース（Set C 実行）

```
1. Task(scope-agent) 完了
   → PostToolUse hook: "JSON_PATH を検出。監視リストに登録"

2. Claude: Task(estimation-agent, "Input: /path/to/Test_Scope_PROJ-1234.json")
   → hook: Read でないので素通り ✅

3. 完了。両エージェントの出力パスをユーザーに報告
```

### ⚠️ 介入検知ケース

```
1. Task(scope-agent) 完了
   → PostToolUse hook: 監視リストに登録

2. Claude: Read("/path/to/Test_Scope_PROJ-1234.json")  ← 介入
   → PreToolUse hook 発火:

   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   ⚠️ INTERVENTION DETECTED - Non-Intervention Violation

   Claude is attempting to READ an agent output file directly.
   File: /path/to/Test_Scope_PROJ-1234.json

   CORRECT:  Task(estimation-agent, "Input: {path}")
   INCORRECT: Read("{path}") ← これをやっている
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## まとめ

| 課題 | 解決策 |
|------|--------|
| Claude がエージェント出力を勝手に読む | Handoff Protocol（パスのみ受け渡し） |
| ルールが守られない | Hook で機械的に検知（PostToolUse + PreToolUse） |
| 毎回指示が面倒 | Happy Set（ワークフローのパッケージ化） |
| スキル実行中の逸脱 | `allowed-tools: Task` で制限 |

### Claude Code でマルチエージェントを設計するとき

```
Claude = DISPATCHER（配車係）
エージェント = WORKER（現場担当者）

配車係は：
✅ どこに何を送るか決める
✅ 伝票（パス）を次の現場に渡す
❌ 荷物の中身を開けて確認しない
❌ 自分で作業しない
```

この「配車係モデル」を Hook で機械的に強制するのが今回の設計の核心。

---

## 補足：Hook スクリプトの全体構造

```
hook_event_name == "PostToolUse" && tool_name == "Task"
  → handle_post_tool_use_task()
      ├── Task 出力からパス抽出（正規表現）
      └── /tmp/.happy_set_watched に書き込み（JSON + timestamp）

hook_event_name == "PreToolUse" && tool_name == "Read"
  → handle_pre_tool_use_read()
      ├── file_path が Test_Scope_*.json か？
      ├── 監視リストと照合（TTL チェック含む）
      └── マッチ → stderr に ⚠️ マルチエージェント構成を使い始めると DETECTED（exit 1）
```

ひとつのスクリプトを PreToolUse・PostToolUse 双方にアタッチし、`hook_event_name` で分岐させるシンプルな設計。

---

## おわりに

「エージェントに任せる」という方針を徹底するには、**仕組みとして強制する**必要がある。

ドキュメントに「読まないこと」と書くだけでは不十分で、Hook を使って「読もうとしたら警告が出る」状態にすることで、設計の意図が実装レベルで保証される。

Happy Set パターンは「よく使う手順をコマンド一発に束ねる」というシンプルなアイデアだが、それ単体だけでなく Handoff Protocol + 介入検知 Hook をセットにして初めて「信頼できるパイプライン」として機能する。