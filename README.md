# BBR-Review

> **Bug Bounty Report Review** — バグバウンティレポートの下書きを体系的に簡易レビューする [Claude Code](https://claude.com/claude-code) スキル。

提出前のレポートを **13 観点 / 100 点満点** でスコアリングし、`Submit-ready` から `Do not submit` までのレディネス、報告蓋然性、そして **Accepted / Informative / Duplicate / OOS / N/A / Spam** の処理結果確率まで予測します。HackerOne / Bugcrowd / Intigriti / YesWeHack の公式ガイドラインを統合した観点で、**Informative・Not Applicable・Spam・Reputation 棄損・アカウント停止のリスクを提出前に潰す**ことを目的としています。

---

## なぜ必要か

バグバウンティで初心者が報酬を逃す理由のほとんどは、脆弱性そのものではなく **レポートの品質** にあります。

- 「再現できない」「攻撃シナリオが非現実的」「定型文・スキャナ・AI コピペ」が、Informative / N/A / アカウント停止の最大要因
- HackerOne / Bugcrowd / Intigriti / YesWeHack はいずれも、近年 **「自動スキャナの出力垂れ流し」「AI 生成文の未検証提出」をスパム扱いで取り締まり**、Reputation スコアの永続的な棄損 → アカウント停止のペナルティを科している
- Intigriti は **48 時間以上の脆弱性蓄積、第三者リークデータ購入、明示的スコープ外での能動的テスト、実 PII の検証** を明示的に禁止しており、違反は重大ペナルティ

BBR-Review は、こうしたペナルティリスクと品質欠落を **提出前の段階で機械的に検出** することを目的としています。

### 本スキルの題材

- Book: [脱初心者のための実践バグバウンティ登竜門](https://zenn.dev/scgajge12/books/06d5b176dfe0d7)
  - Chapter 24: 【5.1. 高品質なレポートの書き方】

---

## 主な機能

### 13 観点・100 点満点での体系的スコアリング

| # | 観点 | 重み |
|---|------|------|
| 1  | スコープ・所有権 | 6 |
| 2  | タイトル | 5 |
| 3  | Description（概要） | 6 |
| 4  | 再現手順 + Environment / Test Accounts | 12 |
| 5  | PoC（HTTP / スクリプトの品質） | 12 |
| 6  | 影響度（Impact） | 10 |
| 7  | **攻撃シナリオの解像度（Attacker→Victim→System）** | 8 |
| 8  | Severity / CVSS | 7 |
| 9  | CWE / 参考文献 | 3 |
| 10 | Recommended Fix（修正提案） | 5 |
| 11 | Supporting Material（スクショ・動画） | 6 |
| 12 | トーン・体裁・Markdown | 10 |
| 13 | **品質汚染チェック（スキャナ / AI / 汎用テンプレ）** | 10 |

> **再現手順 + PoC + Impact + 攻撃シナリオ + 品質汚染で 52 点を占める** 配分。トリアージャー視点での「却下・差戻し・Reputation 棄損リスク」の大きさに比例しています。

### スコアリング基準

| スコア | レディネス | 推奨アクション |
|--------|-----------|---------------|
| 90-100 | **Submit-ready** | このまま提出推奨。微修正のみ。 |
| 75-89  | **Almost ready** | 致命傷なし。指摘点を反映してから提出。 |
| 60-74  | **Needs work** | トリアージャーから差戻し質問が来る可能性が高い。 |
| 40-59  | **Major rewrite** | 構造から書き直し推奨。Informative リスクあり。 |
| 0-39   | **Do not submit** | 報告に値する根拠が不足、または Reputation 棄損リスク。 |

> **観点 13（品質汚染）が 5 以下、または観点 12 でゼロ点（Beg Bounty / 脅迫）の場合は、合計スコアに関係なくレディネスを `Do not submit` に上書き** します。

### 品質汚染インジケータの自動検出

近年プラットフォームが厳格に取り締まっている、以下を最初にチェックします。

- **スキャナ出力臭** — Nuclei / Nessus / Burp Scanner のテンプレ的出力をそのまま貼り付けたパターン
- **AI 生成臭** — 実在しない CWE、辻褄の合わない攻撃シナリオ、対象アプリ非依存のテンプレ Impact
- **汎用テンプレ臭** — 人間執筆でも、対象アプリの固有事情に踏み込まない汎用文

検出された場合は **TL;DR の冒頭で警告** し、観点 13 でゼロ点 + `Do not submit` 上書き。

### プラットフォーム規約違反の検出

Intigriti が明示的に禁止している以下の兆候を検出し、観点 12 / 13 でゼロ点 + 規約違反リスク `重大` を表示します。

- 48 時間以上の脆弱性蓄積（`Date Tested` と現在日付の差分から自動判定）
- 第三者リークデータ購入の示唆
- 明示的スコープ外での能動的テスト
- 実 PII の検証
- 脅迫 / 報酬要求 / 公開威嚇（Beg Bounty）

### 処理結果の確率予測

`Submit-ready` という判定だけでは不足です。Intigriti 公式トリアージスタンダード（v1.3）と HackerOne / Bugcrowd の実運用に基づき、確率分布で予測します。

```
予測される処理結果:
  Accepted     30%
  Informative  50%
  Duplicate    15%
  その他        5%
```

### 攻撃シナリオの 3 段再構成（Attacker → Victim → System）

Bugcrowd 公式の指摘 — **初心者の却下理由は「再現できない」よりも「攻撃シナリオが非現実的」が多い** に対応するため、レポート本文に散逸している情報を 3 段で再構成します。欠落箇所は `[要追記]` で明示。

```
1. 準備 (Attacker)  - 必要な権限／前提、用意する payload・URL・リスナー
2. 実行 (Victim)    - 被害者に求められるアクション、必要な前提条件
3. 結果 (System)    - 直接的な技術的結果、CIA 影響、ビジネス的損害
```

### CVSS / CWE の妥当性検証

- CVSS v3.0 / 3.1 / 4.0 をサポート、8 メトリクスを脆弱性の実態と突き合わせて訂正案を提示
- プログラム固有の **Severity Cap** も考慮
- 実在しない CWE-ID（AI ハルシネーションでよく発生）を検出

### PoC スクリプトのレビュー

JavaScript / Python / cURL の PoC を、以下 5 原則で評価します。

1. **無害性** — 破壊的コマンド、他人の実データ抽出を行わない
2. **最小性** — 必要最低限のコード、余計なヘッダ・トラッキング Cookie を削除
3. **明確性** — 成功/失敗が分岐ロジックで判定可能
4. **自己完結性** — 標準ライブラリのみ、または依存物のインストール手順を明記
5. **スクリプトとペイロードの分離** — `XSS_PAYLOAD` 等の変数で識別可能

---

## インストール

Claude Code のスキルとして利用します。

### Project スキル（このリポジトリ内のプロジェクトでのみ使用）

```sh
# 任意のプロジェクトの .claude/skills/ 配下にコピー
mkdir -p path/to/your/project/.claude/skills
cp -r .claude/skills/bbr-review path/to/your/project/.claude/skills/
```

### User スキル（全プロジェクトで使用）

```sh
mkdir -p ~/.claude/skills
cp -r .claude/skills/bbr-review ~/.claude/skills/
```

その後、Claude Code を起動してスキルが読み込まれていることを確認します。

```sh
claude
> /skills
```

---

## 使い方

### 推奨フロー：input / output ディレクトリ運用

このプロジェクトは **`input/`（レビュー対象を置く）/ `output/`（レビュー結果が保存される）** のディレクトリ運用が前提です。

```
BBRReview/
├── input/      ← ここにレポートと画像を置く
└── output/     ← レビュー結果がここに保存される
```

#### 1. `input/` にファイルを置く

```sh
# 例：レポート本文と Burp のスクショ、view-source の PNG を配置
cp ~/Downloads/draft.md         input/
cp ~/Downloads/burp-request.png input/
cp ~/Downloads/view-source.png  input/
```

受け付けるファイル：

| 種類 | 拡張子 | 用途 |
|------|--------|------|
| レポート本文 | `.md` / `.txt` | 主レポートと補助テキスト（PoC スクリプト等） |
| 画像 | `.png` / `.jpg` / `.jpeg` / `.gif` / `.webp` | Burp スクショ、アラート、view-source、PoC 動画のサムネイル等 |

> Markdown が複数ある場合、`report.md` → `draft.md` → 残りで最終更新が新しいもの、の順で主レポートが決定されます。

> ⚠️ **PII（実在ユーザーの本名・住所・電話番号・決済情報・メール本文等）が写った画像は配置しないでください**。Intigriti 等では「実 PII の検証」自体が規約違反対象です。レビューでも観点 11 / 13 でゼロ点になります。

#### 2. レビューを依頼する

Claude Code でこのプロジェクト直下にいる状態で、以下のいずれかを伝えるとスキルが起動します。

```
input のレポートをレビューして
```

```
/bbr-review
```

トリガーされる典型的なフレーズ：

- 「このレポート見て」「下書きをチェックして」「提出前にレビューして」
- 「H1 / Bugcrowd / Intigriti に出していい？」
- 「Informative で返されないか確認したい」
- 「PoC スクリプトをレビューして」
- 「攻撃シナリオが弱くないか」
- 「これ AI が書いた感じが残ってない？」
- 「Severity これでいい？ CVSS 見て」

#### 3. `output/` でレビュー結果を確認する

レビュー結果は実行日時をファイル名として `output/` に保存されます。

```
output/review_YYYYMMDD_HHMMSS.md
例：output/review_20260506_083045.md
```

実行のたびに新規ファイルが書き出されるため、**「直す → 再レビュー」のループで前後比較**ができます。不要になった古いレビューは適宜削除してかまいません。

### フォールバック：チャット直接貼り付け

`input/` を使わず、チャットに直接レポート本文を貼り付ける従来のフローも引き続きサポートしています。

```
このレポートを提出前にレビューして

[report.md の中身をチャットに直接貼り付け]
```

この場合もレビュー結果は `output/` に保存されます。

### 受け付ける入力（まとめ）

- `input/` に配置した Markdown / テキストファイル + 画像（**推奨**）
- チャットへの直接貼り付け
- Burp の HTTP リクエスト/レスポンス（テキスト化 or スクショで）
- PoC スクリプト単体（JS / Python / cURL）

### 出力フォーマット（抜粋）

```markdown
# Bug Bounty Report Review

## TL;DR
- **総合スコア**: 67 / 100（レディネス：Needs work）
- **報告蓋然性**: Likely
- **予測される処理結果**: Accepted 30% / Informative 50% / Duplicate 15% / その他 5%
- **品質汚染**: 軽度
- **規約違反リスク**: なし
- **最重要の指摘 3 つ**:
  1. ...
  2. ...
  3. ...

## 推定情報
- 想定プログラム / 脆弱性タイプ / 推定 CVSS / 推定 CWE / 攻撃シナリオ要約

## 観点別スコア（13 観点のテーブル）

## 攻撃シナリオの再構成（Attacker → Victim → System）

## 詳細レビュー
（13 観点すべてについて、良い点 → 問題点 → 直し方の具体例）

## 提出前チェックリスト
- [ ] ...

## 提出可否の最終判定
```

「直し方」は **そのままレポートに貼り付けられるレベル** まで具体的に提示するため、レビュー → 修正 → 再レビューのループが高速で回せます。

---

## ディレクトリ構成

```
BBRReview/
├── input/                            # レビュー対象（Markdown / テキスト / 画像）を配置
│   └── README.md                     # input/ の使い方
├── output/                           # レビュー結果が review_YYYYMMDD_HHMMSS.md で保存される
│   └── README.md                     # output/ のファイル名規則・構造
└── .claude/skills/bbr-review/
    ├── SKILL.md                      # スキル本体（13 観点、ワークフロー、出力フォーマット）
    ├── evals/
    │   └── evals.json                # スキル評価用テストケース
    └── references/
        ├── checklist.md              # 13 観点の合格基準・典型 NG 例
        ├── cvss-guide.md             # CVSS v3.0/3.1/4.0、8 メトリクス、Severity Cap
        ├── vuln-types.md             # 脆弱性タイプ別の必須要素・PoC 例・典型 OOS リスト
        ├── examples.md               # 良い/悪いレポートの Before/After 実例、状態予測例
        ├── pitfalls.md               # 初心者の失敗パターン、Intigriti 禁止事項
        ├── report-template.md        # プラットフォーム共通の標準テンプレート（英語版）
        ├── poc-scripts.md            # JS / Python / cURL の PoC 設計と 5 原則
        ├── quality-pollution.md      # スキャナ / AI 生成検出指標、HackerOne 公式ポリシー、Hai Triage
        └── triage-process.md         # 状態判定マトリクス、Intigriti 禁止行為一覧
```

---

## 設計思想

### 「トリアージャーは敵ではなくパートナー」

トリアージャーは、バグハンターの敵ではなく、**企業へ脆弱性を売り込むためのパートナー**です。彼らが開発者を説得しやすいように「読みやすく、再現しやすく、リスクが明確な」資料を提供することが、結果として報奨金を最大化し、支払までの時間を短縮します。

レビューでも常にこの姿勢を貫きます。レポートを「採点」するのではなく、「ハンターとトリアージャー双方の負担を減らし、開発者の修正に直結する資料」へと **共同制作** する立場で改善案を提示します。

### 高品質レポートの 3 原則

主要 4 プラットフォームの公式ガイドラインに共通する原則。判断に迷ったらここに立ち返ります。

1. **再現性（Reproducibility）** — 誰が、いつ、どの環境で実行しても、100% 同じ現象が発生する手順
2. **明確さ（Clarity）** — 冗長な表現を避け、事実と手順が簡潔に。最初の **30 秒〜2 分** で「再現できそうか / 即却下か」が決まる
3. **影響の証明（Impact Proven）** — 「脆弱性がある」だけでなく、「ビジネスへの損害」を論理的に説明し、現実的な攻撃シナリオで証明

### AI 使用時の「責任の所在」原則

AI 補助そのものは禁止されていません（HackerOne 公式）。ただし提出時の責任は **提出者本人** にあります。

「AI を使うこと」ではなく「**未検証で提出すること**」が問題。レビューでも、AI 起源を機械的に否定せず、未検証 / ハルシネーション / 対象アプリ非依存の汎用文を具体的に指摘します。

---

## 参考にした公式ガイドライン

- [HackerOne — Quality Reports](https://docs.hackerone.com/en/articles/8475116-quality-reports)
- [Hacker101 — Writing a Report and CVSS](https://www.hacker101.com/resources/articles/writing_a_report_and_cvss.html)
- [Bugcrowd — How to Write Excellent Reports](https://www.bugcrowd.com/resources/levelup/how-to-write-excellent-reports-techniques-that-save-triagers-time-and-mistakes-that-should-be-avoided-in-reports/)
- [Intigriti — How to Write a Good Report](https://www.intigriti.com/researchers/hackademy/guides/how-to-write-a-good-report)
- [Intigriti — Writing Effective Bug Bounty Reports](https://www.intigriti.com/researchers/blog/hacking-tools/writing-effective-bug-bounty-reports)
- [YesWeHack — Write Effective Bug Bounty Reports](https://www.yeswehack.com/learn-bug-bounty/write-effective-bug-bounty-reports)
- [脱初心者のための実践バグバウンティ登竜門](https://zenn.dev/scgajge12/books/06d5b176dfe0d7)

---

## 免責事項

- 本スキルは、レポートの「品質」と「報告に値する蓋然性」を評価するものであり、**脆弱性の真偽を独断で判定するものではありません**。最終判定は対象プログラムのトリアージ次第です。
- 処理結果の確率予測は **参考値** であり、各プラットフォームの実運用を保証するものではありません。
- 各プラットフォームの規約は変更される可能性があります。提出前に必ず最新の公式ドキュメントを確認してください。

---

## 動作環境

- [Claude Code](https://claude.com/claude-code) v2.x 以上
- Skills 機能が有効化されたバージョン

---

## ライセンス

MIT License

---
