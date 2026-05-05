# 13 観点 詳細チェックリスト

各観点について、(a) 合格基準、(b) 典型 NG 例、(c) スコアリング目安、(d) Quick Fix の順で示す。スコアは観点ごとに独立に評価する。

---

## 1. スコープ・所有権 (6 点)

### 合格基準
- 対象ホスト/IP/アプリがプログラムの **In-Scope** に明示されている、または明示記載と整合する。
- ワイルドカードスコープ（例：`*.example.com`）の場合、所有権の証拠（WHOIS / SSL 証明書 SAN / IP 範囲 / 公式 GitHub 等）が示されている。
- 対象がブランドの実本番環境であり、CDN 越しのサードパーティ（Cloudflare、Akamai、AWS）由来ではないことが示されている。
- アウト・オブ・スコープ（例：マーケサイト、サードパーティ、Pre-prod のテスト環境）でないか確認されている。
- プログラム規約の「Out-of-scope vulnerability types」（自己 XSS、Rate Limit Bypass、SPF/DMARC のみ等）に該当しないか検討されている。

### 典型 NG 例
- 「`example.com` に脆弱性があります」だけで、スコープ照合の言及なし。
- ワイルドカードで未知サブドメインを発見したが、所有権の証明がない。
- 実は対象がプログラム外の SaaS（例：SendGrid、Zendesk、Auth0）で、報告先間違い。
- スコープ外と分かった後も能動的テストを継続している痕跡（Intigriti 明示禁止）。
- 第三者リークデータを使った PoC（Intigriti 明示禁止）。

### スコアリング目安
- 6: スコープ明示 + 所有権証明（必要な場合）+ Out-of-scope 種別でないことの検討
- 4: スコープ明示あり、所有権証明は不要なケース
- 2: スコープ照合への言及がない（読み手が自分で確認しないといけない）
- 0: 明らかにスコープ外

### Quick Fix
冒頭に「Asset & Scope」セクションを 1 段落追加：
```markdown
**Target**: `https://api.example.com/v2/users/{id}`
**In-scope confirmation**: Listed under `*.example.com` in the program brief.
**Ownership evidence**: Resolves to AWS IP within Example Inc. ASN (AS12345); SSL cert CN matches.
```

---

## 2. タイトル (5 点)

### 合格基準
- `[VulnType] in [Location/Parameter] leading to [Impact]` のフォーマットに準拠。
- トリアージャーが**目次画面で見ただけで**重複検索や優先度付けができる粒度。
- 100〜140 文字以内が目安。

### 典型 NG 例
- ❌ `XSS found`（情報不足）
- ❌ `Bug in search bar`（脆弱性の種類が不明）
- ❌ `I can hack your admin panel!!!`（非専門的、感情的）
- ✅ `Stored XSS in "Profile Name" leading to Account Takeover`
- ✅ `IDOR on /api/v1/invoice allows viewing other users' PII`

### スコアリング目安
- 5: 推奨フォーマット完全準拠（タイプ + 場所 + 影響）
- 4: タイプ + 場所はあるが影響不明
- 2: タイプのみ
- 0: 「Bug」「Vulnerability」だけ等の無情報タイトル

### Quick Fix
テンプレ：`[VulnType] in [Endpoint/Component] leading to [Impact]`

---

## 3. Description（概要）(6 点)

### 合格基準
- 3〜5 行の簡潔な要約（YesWeHack 推奨）：
  - 「どの URL（エンドポイント）に」
  - 「どのような脆弱性が存在し」
  - 「最終的に何が起きるか」
- 技術的根拠（脆弱な処理、欠落している検証、信頼境界の越境ポイント）が記述されている。
- CWE-ID への言及がある（推奨）。

### 典型 NG 例
- 一行で「XSS があります」だけ。
- 200 行のリクエストログを Description に貼り付け（Steps 側に分けるべき）。

### スコアリング目安
- 6: 何・どこ・なぜが揃い、CWE と発見コンテキストもある
- 4: 何・どこ・なぜが揃う
- 2: 何のみ
- 0: 一行で意味が取れない

### Quick Fix
```markdown
## Summary

The `name` query parameter on `GET /search` is reflected unescaped into
the HTML response body, allowing arbitrary HTML/JavaScript to execute in
any visitor's browser session under `www.example.com` origin (CWE-79).
```

---

## 4. 再現手順 + Environment / Test Accounts (12 点)

### 合格基準
- **Environment / Test Accounts セクションが独立**：URL / Browser / Attacker Account / Victim Account が明記。
- IDOR・認可検証では **2 アカウント** の認証情報を `Attacker: user1 / pass1`、`Victim: user2 / pass2` 形式で提示。
- **「対象システムを見たことがない人」が再現できる**粒度（Intigriti の指針）。
- 番号付きで段階的、1 ステップ 1 動作。
- 前提条件（必要なアカウント種別、認証情報、ブラウザ、特殊設定）が冒頭に明記されている。
- 短縮版（Triager 用：3-5 ステップ）と完全版（顧客用：A→Z）の両方があるとベスト（Bugcrowd 推奨）。
- 完全な HTTP リクエスト（メソッド、パス、ヘッダ、ボディ）が含まれる。
- ツール（sqlmap、Burp 等）使用時はフルコマンド付き。

### 典型 NG 例
- 「`payload を入れて送る → XSS が出る」のような飛躍。
- ステップ 3 で初めて「あ、認証が必要です」と書いてある（前提を冒頭に書け）。
- スクショだけで HTTP リクエストの本文がない。
- IDOR で 1 アカウントしか使っていない（権限境界の証明にならない）。
- 「テスト用認証情報」の提供がなく、トリアージャーが自分でアカウント作成しないといけない。

### スコアリング目安
- 12: Environment 明示 + Test Credentials 明示 + 番号付き前提 + フル HTTP + ツールコマンド
- 9: Environment + 前提 + フル HTTP（Test Credentials は不要なケース）
- 6: 番号付きだが前提や HTTP の一部が抜ける
- 3: 散文で書かれているが情報は揃っている
- 1: 致命的に飛躍があり再現不能
- 0: 手順がない

### Quick Fix
```markdown
## Environment / Test Accounts
- **URL**: `https://app.example.com/api/v2/users/{id}/email`
- **Browser**: Chrome 130.0 on macOS 14.6 (Burp Suite v2024.x)
- **Attacker**: `attacker@evil.com / Test123!` (user_id 99001)
- **Victim**:   `victim@example.com / Test456!` (user_id 99002)

## Steps to Reproduce

**Prerequisites**
- Two test accounts (above), HTTP client capable of PATCH requests.

**Reproduction**
1. Log in as the attacker; capture the `session=` cookie.
2. Send the following request:
   ```http
   PATCH /api/v2/users/99002/email HTTP/1.1
   Host: api.example.com
   Cookie: session=<attacker_session>
   Content-Type: application/json

   {"email":"pwned@evil.com"}
   ```
3. Observe `200 OK` confirming the email was changed on victim's account.
4. Log in as victim; verify the email field has changed.
```

---

## 5. PoC（Proof of Concept）(12 点)

### 合格基準
- 動作確認済みの PoC（タイポ、不正な URL、改行ミスがない）。
- リクエスト → レスポンス → 影響の **証拠の鎖** が閉じている。
- HTTP リクエスト/レスポンスは**スクリーンショットではなくテキスト（コードブロック）**として貼られている（Bugcrowd / Intigriti 共通推奨）。
- 不要なヘッダ（大量のトラッキング Cookie、`User-Agent` 以外の不要ヘッダ）が削除されている。
- XSS など視覚的な脆弱性は静止画 + 動画、サーバ側脆弱性はリクエスト/レスポンスで示す。
- 補助ファイル（XXE の DTD、SSRF のコールバック URL、CSRF の HTML フォーム等）がインラインまたは添付されている。
- 安全な PoC ペイロード（実害を出さないもの）が使われている。
- **PoC スクリプトを含む場合**：5 原則（無害・最小・明確・自己完結・スクリプト/ペイロード分離）を満たす。詳細は `poc-scripts.md`。

### 典型 NG 例
- `<script>alert(1)</script>` だけ（影響が伝わらない、Intigriti が明示的に NG）。
- リクエストはあるがレスポンスがない、またはその逆。
- スクショに DevTools コンソールしか写っておらず、ブラウザのアドレスバーが見えない。
- HTTP リクエストにトラッキング Cookie が 30 個入っている（可読性低下）。
- Burp ログをスクショで貼り、コピペ検証できない。
- PoC として「他ユーザーのメアドを実際に変更しました」と報告（規約違反リスク）。

### スコアリング目安
- 12: 動作 PoC + テキスト形式 HTTP + ノイズ削減 + リクエスト/レスポンス両方 + 必要なら PoC スクリプト
- 9: 動作 PoC + テキスト HTTP + リクエスト/レスポンス両方
- 6: PoC はあるが片方だけ、またはノイズが多い
- 3: PoC が記載されているが再構築する必要がある
- 0: PoC なし、または動かない

### Quick Fix
PoC スクリプトを添える場合のテンプレートは `poc-scripts.md` を参照。

---

## 6. 影響度（Impact）(10 点)

### 合格基準
- **対象アプリ固有のビジネスインパクト**が書かれている（汎用テンプレ NG）。
- 「攻撃が完全に成立した場合の最悪シナリオ」が記述されている。
- 機密データの種類（PII、決済情報、社内情報、API キー、認証トークン等）が具体的に特定されている。
- 影響を受けるユーザー範囲（全ユーザー / 同テナント / 認証ユーザー / 管理者）が明確。
- CIA の侵害（Confidentiality / Integrity / Availability）が論理的に説明されている。
- 過大評価も過小評価もしない。

### 典型 NG 例
- ❌「アラート `1` が表示されます」（技術的事象のみ、ビジネス文脈ゼロ）
- ❌「This vulnerability allows attackers to compromise the application」(汎用)
- ❌「攻撃者は機密情報を取得できます」(具体性ゼロ)
- ✅「攻撃者はこの Stored XSS を利用して、管理者を含む閲覧者のセッション Cookie を窃取できます。これにより、管理者アカウントの乗っ取りが可能となり、システム全体のデータを操作できるリスクがあります」

### スコアリング目安
- 10: ビジネスインパクト具体 + 最悪シナリオ + データ種別 + 影響範囲 + CIA 言及
- 8: ビジネスインパクト + 最悪シナリオ + データ種別
- 5: 一般的な Impact だがアプリ固有性が弱い
- 3: 定型文に少しだけアプリ情報を加えた程度
- 1: 完全な定型文コピペ
- 0: Impact セクション自体がない

### Quick Fix
具体例は `examples.md` の「After」セクションを参照。

---

## 7. 攻撃シナリオの解像度（Attacker → Victim → System）(8 点)

### この観点が独立している理由（最重要）

**Bugcrowd の公式指摘**：トリアージャーが初心者レポートを却下する理由として最多なのは「**再現できないから**」ではなく、「**攻撃シナリオが非現実的だから**」。多くのバグハンターは技術的な挙動（アラートが出た、エラーが出た）に集中するが、セキュリティチームが見ているのは「**そのバグを使って、攻撃者が被害者に対して何を行えるか**」という**ストーリー**。

この観点で減点されたレポートは、Severity 評価が高くても Informative / N/A 判定される確率が劇的に上がる。レビュー時、技術的な PoC が完璧でもこの 3 段が破綻していれば**遠慮なく低スコア**にする。

### 合格基準
独立したセクション、または Impact / Steps の中で**3 段が明確に区別**されている：

1. **準備（Attacker）**：攻撃者が何を準備するか（payload、攻撃用 URL、リスナー設置）
2. **実行（Victim）**：被害者に求められる最低限のアクション（リンククリック、ファイルアップロード、特定機能の使用）と前提条件（ログイン状態、特定権限）
3. **結果（System）**：直接的な技術的結果と、最終的なビジネス的損害

- 攻撃成立の前提条件（被害者のログイン状態、ブラウザ設定、ネットワーク条件等）が明記。
- **現実性**：「メールやSNSで配布」「リンククリックさせる」が現実的に可能な経路で説明されている。

### 典型 NG 例
- 「攻撃者は機密情報を取得できる」（Attacker / Victim / System が分離されていない）
- 「ユーザーが攻撃を受ける」（Victim の操作が抽象的）
- 自己 XSS なのに「攻撃者がこれを使ってユーザーを攻撃できる」（Attacker → Victim の経路が無い）
- CSRF だが SameSite Cookie の確認なしに「攻撃が成立する」と主張

### スコアリング目安
- 8: 3 段すべて具体 + 前提条件明記 + 現実的な配布経路
- 6: 3 段は揃うが前提条件か現実性のどちらかが弱い
- 4: 2 段のみ（System / Victim どちらかが曖昧）
- 2: 技術的事象のみで攻撃シナリオなし
- 0: 攻撃シナリオが論理破綻（自己 XSS で他者攻撃を主張等）

### Quick Fix
```markdown
## Attack Scenario

### 1. Attacker Preparation
- The attacker crafts a URL embedding the XSS payload:
  `https://www.example.com/search?q=...`
- The attacker hosts a listener at `https://attacker.example.com/c/`
  to receive exfiltrated cookies.
- Distribution channel: email phishing, social media link, or
  malicious ad inventory.

### 2. Victim Action
- Required victim state: any visitor (no auth required) using a
  modern browser with cookies enabled.
- Required action: clicking the attacker URL once.

### 3. System Outcome
- The payload renders unescaped → JavaScript executes in
  `www.example.com` origin.
- Session cookie (no `HttpOnly` flag) is exfiltrated to the listener.
- Attacker hijacks the session → access to victim's order history,
  saved payment methods, ability to change account email →
  full account takeover.
```

---

## 8. Severity / CVSS (7 点)

### 合格基準
- CVSS ベクトル文字列が記載されている（プラットフォーム指定の版：通常 v3.1、HackerOne なら v3.0/3.1/4.0 すべて可、Bugcrowd は VRT 連携、Intigriti は v3.x/4.x、YesWeHack は v3.1）。
- ベクトル例：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`
- 各メトリクスが脆弱性の実態と整合している。
- 計算スコア（Base Score）が記載されている。
- Severity ラベル（Critical / High / Medium / Low / None）も併記。HackerOne は 5 段階（None / Low / Medium / High / Critical）。
- ベクトルの選択根拠が短く書かれている（Hacker101 が推奨：「なぜこのスコアになるのか」を Impact 欄で補足）。
- **プログラムの Severity Cap**（最大 severity 上限）への配慮：アセットによっては Critical 主張しても Cap で Medium に下がるため、規約再確認推奨。

### 典型 NG 例
- CVSS 記載なし。
- ベクトルは書いてあるが Score が書いてない（または逆）。
- 自己 XSS なのに `UI:N`。
- CSRF を `S:C/C:H/I:H/A:H` で Critical にしている（Scope:Changed の根拠なし）。
- 「Severity: Critical」だけで CVSS なし。

### スコアリング目安
- 7: ベクトル + スコア + Severity + 根拠
- 5: ベクトル + スコア + Severity
- 3: Severity のみ（CVSS なし）
- 1: 「Critical」だけ書いてある
- 0: Severity 記載なし

詳しいベクトル選択は `cvss-guide.md`。

---

## 9. CWE / 参考文献 (3 点)

### 合格基準
- CWE-ID が脆弱性の実態と一致している。
- **CWE が実在する**（AI ハルシネーション NG）。`CWE-` のあとの番号が MITRE 公式リスト[https://cwe.mitre.org/](https://cwe.mitre.org/) と一致。
- OWASP Top 10 の該当カテゴリへの言及（推奨）。
- **Bugcrowd プログラム向け**：Bugcrowd VRT カテゴリ名の記載（[https://bugcrowd.com/vulnerability-rating-taxonomy](https://bugcrowd.com/vulnerability-rating-taxonomy)）。Triager の重複検出と P レベル割当が速くなる。
- ベンダー Advisory、CVE、過去の類似報告へのリンク（あれば）。

### 典型 NG 例
- 「CWE-XXX」とプレースホルダのまま。
- Stored XSS なのに無関係な CWE-200 と書いている。
- 存在しない CWE 番号（例：`CWE-9999`）— AI 生成の典型的兆候。

### スコアリング目安
- 3: 適切な CWE + OWASP + 関連リファレンス
- 2: 適切な CWE のみ
- 1: CWE があるが不適切
- 0: CWE / リファレンスなし、または存在しない CWE

---

## 10. Recommended Fix（修正提案）(5 点)

### 合格基準
- **対象アプリのスタックに即した**具体的な修正案（一般論ではなく）。
- 入力バリデーション、出力エンコーディング、認可チェック、CSRF トークン等、対症ではなく根本対策。
- 言語/フレームワーク固有の安全な API への置き換え提案：
  - **HTML サニタイズ**：DOMPurify (JS), Bleach (Python), Ammonia (Rust)
  - **HTTP セキュリティヘッダ**：Helmet.js (Node), secure_headers (Ruby)
  - **CSRF**：Gorilla CSRF (Go), Django built-in
  - **暗号**：Google Tink, Themis
  - **XML**：defusedxml (Python)
  - **SSRF**：ssrf_filter (Ruby), ssrf-req-filter (Node)
- 副作用の警告（例：「すべての既存リクエストに影響するので段階的展開を推奨」）。

### 典型 NG 例
- 「ユーザー入力をサニタイズしてください」(一般論)
- 「ホワイトリストを実装してください」(具体性ゼロ)
- 「最新版にアップデートしてください」(根本原因の理解が示されない)

### スコアリング目安
- 5: スタック固有 + 根本対策 + 副作用考慮 + 推奨ライブラリ
- 4: 根本対策が示され具体性あり
- 2: 一般論だけ
- 0: Recommended Fix なし

---

## 11. Supporting Material（スクショ・動画）(6 点)

### 合格基準
- スクショ/動画が「証拠」として機能する（URL バー、レスポンス、関連 UI が写る）。
- 機微情報（実在する個人情報、自分以外の本物のセッション、社内情報）が**マスキング**されている。
- 動画は短く（30 秒〜2 分目安）、要点だけ。
- 画像にハイライトや矢印で注目箇所が示されている。
- ファイル名が説明的（`xss-poc.mp4`、`idor-response.png` 等）。
- HTTP リクエスト/レスポンスは**スクショではなくテキスト**で貼る（観点 5 と整合）。

### 典型 NG 例
- スクショに本物の他人の PII が写ったまま（規約違反リスク）。
- 10 分の無編集動画。
- 「ビジュアルがない」（複雑な脆弱性ほど致命的）。
- 重要な HTTP リクエストがスクショだけで提供（テキスト形式が無い）。

### スコアリング目安
- 6: 必要箇所すべてに視覚資料 + マスキング + 短い動画 + 説明
- 5: 視覚資料あり、機微情報も問題なし
- 3: 視覚資料はあるが説明なし or 不要に長い
- 2: スクショ 1 枚だけで証拠として弱い
- 0: 視覚資料なし

---

## 12. トーン・体裁・Markdown (10 点)

### 合格基準
- プロフェッショナルなトーン（侮辱、煽り、emoji 連打 NG）。
- Markdown が適切に使われている：
  - コードブロック（`` ``` ``）でリクエスト/レスポンス/ペイロードを分離
  - 太字（`**`）で重要箇所を強調
  - 引用（`>`）で参考文献の引用
  - 見出し階層が論理的
- 誤字脱字、文法ミスがない。
- **報酬要求や脅迫がない（Beg Bounty NG）**：「報酬をくれないと公開する」「Severity を High にしてくれ」等の要求は厳禁。
- セクション順序が標準テンプレート（`report-template.md`）に準拠。
- レポート言語がプログラムの想定言語（通常英語）に合っている。

### 典型 NG 例
- 「Pay me $$$ for this critical bug」のような直接的な報酬要求。
- 「Severity を High に上げないと公開するぞ」（脅迫＝アカウント停止リスク、Disclosure Threat）。
- すべて大文字で書く、感嘆符の連打。
- コードがインライン引用なしで地の文に混ざっている。
- 日本語のレポートをグローバル H1 プログラムに提出。
- 「48 時間以上溜めてから出す」と発言（Intigriti 公式禁止：脆弱性の意図的蓄積）。
- 同じレポートを複数プラットフォームに同時提出（重複扱い + Reputation 低下）。

### スコアリング目安
- 10: プロ + Markdown + 誤字なし + Beg Bounty なし + 適切な順序 + 想定言語
- 8: 軽微な体裁ミス（少しの誤字 or 軽い順序のズレ）
- 5: 体裁が一貫しない（見出し階層の乱れ、コードブロック未使用、想定言語との不一致）
- 2: 読みにくい、誤字多数、Markdown が崩れている
- 0: Beg Bounty / 脅迫 / Disclosure Threat を含む（即時 Reputation 棄損リスク）

**観点 13 との切り分け**：
- 観点 12 = 形式（Markdown / トーン / 順序 / 言語 / Beg Bounty）
- 観点 13 = 対象アプリ非依存性・テンプレ性（スキャナ垂れ流し・AI ハルシネーション・汎用テンプレ Impact）
- 「英語が崩れている」は観点 12、「過度に流暢で対象非依存」は観点 13

---

## 13. 品質汚染チェック（スキャナ垂れ流し / AI 生成丸投げ / 汎用テンプレ）(10 点)

この観点は **「対象アプリ非依存性」** を測る。発生源は問わない（スキャナ・AI・他人のテンプレ・自分の使い回し）。Bugcrowd 公式指摘：**70% 以上のハンターが Impact を汎用テンプレでコピペしている** — それを観点 13 で潰す。

### 合格基準（汚染なし = 高得点）
- 自動スキャナ（Nuclei / Nessus / Burp Scanner / Acunetix 等）の出力をそのまま貼っていない。
- スキャナ警告に対して**手動で再現検証**が行われている。
- AI 生成文をそのまま提出していない（人間による精査の痕跡がある）。
- 他人/過去レポートのテンプレを対象非依存のまま貼っていない（汎用 Impact 文 NG）。
- CWE 番号、CVE 番号、ライブラリ名、関数名が**実在する**もの。
- 攻撃シナリオが論理的に成立する（AI ハルシネーションでない）。
- 対象アプリ固有のコンテキスト（実エンドポイント、実機能名、実バージョン、実データ種別等）が複数箇所に含まれている。

### HackerOne 公式ポリシー（2025-26）
HackerOne Code of Conduct は AI 利用そのものは禁止していないが、「ハルシネーション、不正確な技術内容、低努力ノイズ」をスパム判定 → エンフォースメント対象としている。2025 年から **Hai Triage**（半自動トリアージ）が導入され、表面的なヒューリスティックで低品質レポートが事前フィルタされる。Nextcloud は AI 生成レポート急増を理由に 2026 年 4 月にバグバウンティプログラムを終了している。

### 「スキャナ垂れ流し」の検出サイン
- バージョン情報や HTTP ヘッダだけを根拠に「脆弱性」と主張
- 「The server returns Server: Apache/2.4.1 which has known vulnerabilities」だけで PoC なし
- スキャナ特有の出力形式（`[INFO]`、`[CRITICAL]`、CVE-ID 列挙のみ）
- 手動再現の証拠（リクエスト/レスポンス/影響）がない

### 「AI 生成丸投げ」の検出サイン
- 過度に流暢で対象非依存の汎用文（「This vulnerability poses a significant risk to...」を多用）
- 存在しない CWE / CVE 番号
- 存在しないパラメータ・ファイルパスを「攻撃可能箇所」として記述
- 攻撃シナリオの辻褄が合わない（自己 XSS なのに「他ユーザーが影響を受ける」等）
- 同じ意味の文を別の表現で 2-3 回繰り返す（AI の冗長性パターン）
- 関係のない一般論を脆弱性レポートに混入（「セキュリティはチームの責任です」等）
- 不自然な箇条書きの連続（中身が薄い）
- 対象アプリ名の言及が一切なく、テンプレ的な攻撃説明のみ

### スコアリング目安
- 10: 汚染なし、人間が手動検証した痕跡が明確、対象アプリ固有のコンテキスト多数
- 8: 軽度の汎用文が散見されるが手動検証は行われている
- 5: 中程度の汚染（スキャナ出力の言及あり、または AI 文体の混在）
- 2: 重度の汚染（手動検証なしのスキャナ警告、AI 生成丸投げ感）
- 0: 完全な汚染（提出するだけで Reputation 棄損リスク確定）

### スコアが 5 以下の場合の対処
**他観点が満点でも `Do not submit` を推奨する**。観点 13 で 5 以下が出たら、TL;DR の「品質汚染」フィールドを `重度` または `軽度` にし、修正後に再レビューするよう促す。汚染レポートは：
- HackerOne / Bugcrowd / Intigriti / YesWeHack で**スパム判定** → Reputation 棄損
- 繰り返しでアカウント停止
- プログラム招待が来なくなる

検出時の対処は `quality-pollution.md` を参照。

---

## スコアリング総括

各観点を上記基準で採点し、合計を 100 点満点で算出する。レディネス判定は SKILL.md の表に従う。

**注意**：観点ごとに小数点は使わず整数で採点する。最大点ジャストか満点の半分か、明確な根拠がない限り中間値で逃げない。これは「具体的に何が足りないのか」をユーザーに伝えるための規律。

**観点 13 のオーバーライド規則**：観点 13 が 5 以下の場合、合計スコアの算出はそのまま行うが、レディネス判定は `Do not submit` を上書き表示する。「品質汚染」フィールドの値（重度/軽度/なし）を TL;DR 冒頭に明示する。
