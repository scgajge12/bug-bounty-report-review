# 標準レポートテンプレート（プラットフォーム共通・英語版）

HackerOne / Bugcrowd / Intigriti / YesWeHack に共通する標準的なレポート構造。レビュー対象のレポートがこの構造から大きく逸脱している場合、観点 11（Supporting Material）/ 12（トーン・体裁）の評価に反映する。

セクション順は **Summary → Estimated Severity → Environment / Test Accounts → Steps to Reproduce → Proof of Concept → Impact → Supporting Material → Recommended Fix → References**。トリアージャーが上から順に読んで、(1) 何の脆弱性か即座に把握、(2) Severity を確認、(3) テスト環境を準備、(4) 再現手順を実行、(5) PoC で証拠確認、(6) Impact でビジネス文脈理解、(7) 視覚資料で確証、(8) 修正方針を開発者に渡す、という業務フローを最短化する設計。

レポートを 0 から書く / 全面書き直しを助言する場面では、このテンプレートをそのまま渡せる形で参照してもらう。

---

## テンプレート（コピペ可）

````markdown
# [Vulnerability Type] in [Location/Parameter] leading to [Impact]

## Summary
- **Vulnerability Type**: e.g. Stored Cross-Site Scripting (CWE-79)
- **Asset / Target**: e.g. https://app.example.com (in-scope per program brief)

A 3-5 line plain-English summary: where the vulnerability lives, what
causes it, and what the worst-case business consequence is.

## Estimated Severity
- **Severity**: e.g. High
- **CVSS Vector**: `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N` (8.1)
- **Justification**: One paragraph explaining the metric choices the
  triager might disagree with (typically `S`, `PR`, and `UI`).

## Environment / Test Accounts
- **URL**: `https://app.example.com/api/v2/users/{id}/email`
- **Browser**: Chrome 130.0 on macOS 14.6 (Burp Suite v2024.x)
- **Attacker Account**: `attacker@evil.com / Test123!` (user_id 99001)
- **Victim Account**:   `victim@example.com / Test456!` (user_id 99002)
- **Date Tested**: YYYY-MM-DD HH:MM UTC

## Steps to Reproduce

**Prerequisites**
- (List anything the triager must set up before step 1)

**Reproduction**
1. ...
2. ...
3. ...

(Each step is one action. URLs, payloads, and HTTP requests go inside
fenced code blocks. No prose paragraphs inside this section.)

## Proof of Concept (PoC)

### HTTP Request / Response
```http
PATCH /api/v2/users/99002/email HTTP/1.1
Host: api.example.com
Cookie: session=<attacker_session>
Content-Type: application/json

{"email":"pwned@evil.com"}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"id":99002,"email":"pwned@evil.com","updated_at":"2026-05-05T..."}
```

### Script PoC (optional but recommended for complex flows)
See `poc-scripts.md` for JavaScript / Python / cURL templates.

```python
# poc.py
import requests

ATTACKER_COOKIE = "session=<attacker_session>"
VICTIM_USER_ID  = 99002
NEW_EMAIL       = "pwned@evil.com"
TARGET          = f"https://api.example.com/v2/users/{VICTIM_USER_ID}/email"

resp = requests.patch(
    TARGET,
    headers={"Cookie": ATTACKER_COOKIE, "Content-Type": "application/json"},
    json={"email": NEW_EMAIL},
    timeout=10,
)

if resp.status_code == 200 and NEW_EMAIL in resp.text:
    print("[+] IDOR confirmed: victim email was changed")
else:
    print(f"[-] Not vulnerable (status={resp.status_code})")
```

## Impact

### Attack Scenario (Attacker → Victim → System)

**1. Attacker preparation**
- Attacker registers any normal account on `app.example.com`.
- Attacker enumerates user IDs via `GET /api/v2/users` (sequential IDs).

**2. Victim involvement**
- No victim action is required; the vulnerability is exploited entirely
  server-side from the attacker's session.

**3. System outcome**
- Email of any chosen user is overwritten to an attacker-controlled
  address. Subsequent password reset → full account takeover.

### Business Impact
- All ~1.2M registered users (per the public stats page) are affected.
- Administrative accounts are identified by ID range 1-100, allowing
  targeted compromise of internal admin panels.
- Subsequent access on takeover: stored payment methods (PAN last-4),
  order history (PII), and admin-only features.
- Regulatory exposure: GDPR notification obligation for EU customers.

## Supporting Material
- `idor-200-response.png` — successful PATCH response with victim's data
- `idor-takeover.mp4` — 45-second screencast of full ATO flow
  (PII masked, only 2 test accounts used)

## Recommended Fix

**Root cause**: The `users#update` controller resolves
`User.find(params[:id])` without an authorization check.

**Fix**:
1. Enforce `authorize @user, :update?` (Pundit) before the email update
   in `app/controllers/users_controller.rb`.
2. Restrict admin overrides to a separate explicit endpoint requiring
   an admin role check.

**Defense-in-depth**:
- Migrate to UUIDs for user IDs to limit enumeration risk.
- Add an audit log entry for any email change request.

## References
- CWE-639: https://cwe.mitre.org/data/definitions/639.html
- OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization
- Program brief: (link to relevant scope/policy)
````

---

## セクション別の注意点

### Title
- 推奨フォーマット：`[VulnType] in [Location/Parameter] leading to [Impact]`
- 100〜140 文字以内
- 感情・絵文字・大文字連打 NG

### Summary
- 3〜5 行で「どこに」「どんな脆弱性が」「最終的に何が」を示す
- 技術的な詳細は Steps / PoC に分ける

### Estimated Severity
- ベクトル + スコア + Severity ラベル + **根拠**の 4 点セット
- Hacker101 推奨：「なぜこのスコアになるか」を Impact 欄で補足

### Environment / Test Accounts
- IDOR / 認可検証では必ず 2 アカウント明記
- ブラウザ / OS / ツールバージョンを記録（環境依存バグの場合に必須）
- テスト用アカウントの作成方法（公開サインアップ / 招待制）も書くと親切

### Steps to Reproduce
- **「対象システムを見たことがない人」が再現できる**粒度
- 1 ステップ 1 アクション、散文 NG
- HTTP リクエストは ``` http ``` ブロックで分離
- 短縮版（Triager 用）と完全版（顧客用）の両方があるとベスト

### Proof of Concept
- HTTP リクエスト/レスポンスは**テキスト**で（スクショ NG）
- 不要なヘッダ（トラッキング Cookie 等）は削除
- 複雑な手順は PoC スクリプトを併載（`poc-scripts.md` 参照）

### Impact
- **Attacker → Victim → System の 3 段**で攻撃シナリオを書く
- ビジネス的損害（CIA 影響、ユーザー数、規制対応）
- 対象アプリ固有のコンテキストを必ず含める（テンプレ NG）

### Supporting Material
- ファイル名は説明的（`xss-poc.mp4`、`idor-response.png`）
- 動画は 30 秒〜2 分、機微情報マスキング必須
- HTTP は PoC セクションでテキスト化、ここはスクショ/動画のみ

### Recommended Fix
- スタック固有の修正案
- 推奨ライブラリ（DOMPurify / Helmet.js / defusedxml 等）
- 副作用の警告

### References
- CWE-ID（実在するもの）
- OWASP Top 10 該当カテゴリ
- 関連 CVE / ベンダー Advisory
- プログラム規約・スコープへのリンク（必要なら）

---

## カスタマイズ余地

プログラムによっては以下のセクションを追加 or 削除する：
- **Disclosure Plan** : 公開時期の希望（HackerOne の Disclosure Request 用）
- **Mitigation Status** : 自分で確認した暫定回避策
- **Related Reports** : 既知の類似脆弱性へのリンク（重複でないことの説明）

YesWeHack は「KYC 認証後に Profile > My YesWeHack tools > Report templates でカスタムテンプレートを保存できる」と公式案内している。よく報告するハンターはプラットフォーム機能を活用すること。
