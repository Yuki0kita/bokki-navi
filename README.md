# バイアグラなび - デプロイ・運用ガイド

## ディレクトリ構成

```
viagranavi/
├── index.html              # TOP
├── compare.html            # 薬比較表
├── article-compare.html    # 薬の選び方
├── article-danger.html     # 飲み合わせ
├── article-basic.html      # 基礎知識
├── article-side.html       # 副作用
├── article-sns.html        # SNSの声まとめ（NEW）
├── shindan.html            # 診断
├── compliance.html         # 薬機法・広告表示（NEW）
├── privacy.html            # プライバシーポリシー
├── about.html              # 運営者情報（MA売却・A8.net審査必須）
├── contact.html            # お問い合わせ（信頼性・法的要件）
├── tokusho.html            # 特定商取引法（A8.net規約・法的必須）
├── robots.txt              # クローラー制御
├── sitemap.xml             # サイトマップ
├── nginx.conf              # Nginx本番設定
├── _headers                # Cloudflare Pages ヘッダー
└── _redirects              # Cloudflare Pages リダイレクト
```

## デプロイ手順

### A. Cloudflare Pages（推奨）

1. GitHubリポジトリにプッシュ
2. Cloudflare Dashboard → Pages → Create a project
3. ビルドコマンド: なし（静的HTML）
4. `_headers` と `_redirects` が自動適用される
5. カスタムドメインを設定

```bash
# GitHub連携例
git init
git add .
git commit -m "initial: バイアグラなび v1"
git remote add origin https://github.com/yourname/viagranavi.git
git push -u origin main
```

### B. Nginx + VPS（Ubuntu 22.04 LTS）

```bash
# 1. Nginxインストール
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx

# 2. ファイル配置
sudo mkdir -p /var/www/viagranavi
sudo cp -r ./*.html ./*.txt ./*.xml /var/www/viagranavi/
sudo chown -R www-data:www-data /var/www/viagranavi

# 3. Nginx設定適用
sudo cp nginx.conf /etc/nginx/sites-available/viagranavi.conf
sudo ln -s /etc/nginx/sites-available/viagranavi.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# 4. SSL取得（Let's Encrypt）
sudo certbot --nginx -d example.com -d www.example.com

# 5. 自動更新設定
sudo systemctl enable certbot.timer
```

## アフィリエイトリンク差し替え手順

AFF_CARDSのhref="#rebakuri"等を実際のA8.netリンクに差し替えます。

```bash
# Pythonで一括置換
python3 << 'EOF'
import os, re

links = {
    '#rebakuri': 'https://px.a8.net/svt/ejp?a8mat=YOUR_REBAKURI_LINK',
    '#east-ed':  'https://px.a8.net/svt/ejp?a8mat=YOUR_EAST_ED_LINK',
    '#east-aga': 'https://px.a8.net/svt/ejp?a8mat=YOUR_EAST_AGA_LINK',
}

for fname in [f for f in os.listdir('.') if f.endswith('.html')]:
    with open(fname) as f:
        content = f.read()
    for placeholder, real_link in links.items():
        content = content.replace(f'href="{placeholder}"', f'href="{real_link}"')
        content = content.replace(f"href='{placeholder}'", f"href='{real_link}'")
    with open(fname, 'w') as f:
        f.write(content)
    print(f"Updated: {fname}")
EOF
```

## 薬事法コンプライアンス チェックリスト

デプロイ前に以下を確認：

- [ ] 効果の断定表現がないこと（「必ず効く」「完全に治る」等）
- [ ] 根拠のないNo.1・最強表現がないこと
- [ ] アフィリエイト開示が全ページにあること（フッターの免責事項）
- [ ] 体験談に「個人差あり・効果を保証しない」注記があること
- [ ] クリニックカードに「公式サイト掲載情報・処方は医師判断」注記があること
- [ ] compliance.htmlが正しくリンクされていること
- [ ] about.html（運営者名・所在地）を実際の情報に更新
- [ ] tokusho.html（運営者名・所在地・URL）を実際の情報に更新
- [ ] contact.htmlのフォームをFormspree等の実エンドポイントに接続

- [ ] 医師の処方が必要であることが明記されていること

### NGワード自動チェック

```bash
python3 << 'EOF'
import os, re
NG = ['必ず効く','絶対効く','完全に治る','副作用なし','副作用ゼロ']
for f in [x for x in os.listdir('.') if x.endswith('.html')]:
    text = re.sub(r'<[^>]+>',' ', open(f).read())
    for w in NG:
        if w in text:
            print(f"WARNING {f}: '{w}' found")
EOF
```

## パフォーマンス目標

| 指標 | 目標値 | 測定ツール |
|------|--------|-----------|
| LCP | < 2.5s | PageSpeed Insights |
| FID | < 100ms | PageSpeed Insights |
| CLS | < 0.1 | PageSpeed Insights |
| TTFB | < 600ms | WebPageTest |
| Lighthouse スコア | > 90 | Lighthouse |

### 最適化状況

- ✅ Noto Sans JP フォントのプリロード（rel="preload"）
- ✅ フォントの非同期読み込み（onload手法）
- ✅ Lazy load（IntersectionObserver）実装
- ✅ CSS インライン（RTT削減）
- ✅ Gzip圧縮（nginx設定済み）
- ✅ ブラウザキャッシュ設定（HTML: 1h、CSS/JS: 7d）
- 🔲 WebP画像変換（画像追加時に対応）
- 🔲 Service Worker（オフライン対応・任意）

## セキュリティ設定状況

| 設定 | 状態 |
|------|------|
| HTTPS強制 | ✅ 301リダイレクト |
| HSTS | ✅ max-age=63072000 |
| CSP | ✅ 設定済み |
| X-Frame-Options | ✅ DENY |
| X-Content-Type-Options | ✅ nosniff |
| Referrer-Policy | ✅ strict-origin-when-cross-origin |
| Permissions-Policy | ✅ 設定済み |
| レート制限 | ✅ 30r/m |
| .env等ブロック | ✅ 設定済み |

## 更新フロー

1. ローカルでHTMLを編集
2. NGワードチェックを実行
3. コンプライアンスチェックリスト確認
4. Gitにコミット → 自動デプロイ（Cloudflare Pages）またはSCP転送（VPS）
5. PageSpeed Insightsでスコア確認

## ライセンス・注意

本サイトのコンテンツはアフィリエイト広告を含みます。
薬機法・医療広告ガイドラインの最新情報は厚生労働省公式サイトをご確認ください。
https://www.mhlw.go.jp/stf/seisakunitsuite/bunya/kenkou_iryou/iryou/koukoku/
