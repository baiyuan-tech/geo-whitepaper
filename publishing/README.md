# publishing/ — 日本語圏向け投稿素材

本ディレクトリは、日本語ホワイトペーパー（`../ja/`）を **note.com** と **Qiita** に投稿するための記事草稿を収録する。

## ディレクトリ構成

```text
publishing/
├── note/       ← note.com 向け記事（CMO/CDO/事業責任者向け、物語的）
│   ├── 01-brand-disappearing.md
│   └── 02-japan-b2b-first-mover.md
└── qiita/      ← Qiita 向け記事（エンジニア向け、実装重視）
    ├── 01-geo-overview.md
    ├── 02-axp-cloudflare-worker.md
    └── 03-stale-carry-forward.md
```

## 投稿手順

### Qiita

1. <https://qiita.com/drafts/new> で新規投稿画面を開く
2. 記事ファイル冒頭のコメント（`Title`、`Tags`、`Private`）を確認
3. コメント行（`<!-- ... -->`）を除いた本文をコピー＆ペースト
4. タイトル欄にタイトルを、タグ欄にタグ（カンマ区切り）を入力
5. プレビューで Mermaid ダイアグラムとコードブロックを確認（Qiita はデフォルトで Mermaid 対応）
6. 公開

### note.com

1. <https://note.com/new> で新規投稿画面を開く
2. 記事ファイル冒頭のコメント（`Title`、`Tags`、`Cover`）を確認
3. コメント行を除いた本文をコピー＆ペースト
4. カバー画像を設定（コメント内の指示参照）
5. タグを追加（note.com はハッシュタグ形式）
6. 本文末尾の「導入相談・パートナーシップ」CTA を確認
7. 公開

## 投稿順序の推奨

**Week 1**：

- [qiita/01-geo-overview.md](./qiita/01-geo-overview.md) ← 入り口記事、GEO 分野の全体像
- [note/01-brand-disappearing.md](./note/01-brand-disappearing.md) ← ビジネス側の導入

**Week 2**：

- [qiita/02-axp-cloudflare-worker.md](./qiita/02-axp-cloudflare-worker.md) ← AXP 実装詳解（技術深掘り）

**Week 3**：

- [qiita/03-stale-carry-forward.md](./qiita/03-stale-carry-forward.md) ← パターン的な一般化（他分野エンジニアにも届く）

**Week 4**：

- [note/02-japan-b2b-first-mover.md](./note/02-japan-b2b-first-mover.md) ← 日本市場向けの決定版記事

## 投稿 URL トラッキング

投稿後、各ファイル冒頭の `投稿URL:` 欄を更新する：

```diff
  投稿URL（投稿後に記入）: https://qiita.com/...
+ 投稿URL（投稿後に記入）: https://qiita.com/vincentlin/items/xxxxx
```

## 共通事項

- 全記事の末尾に **ホワイトペーパー本体へのリンク** + **プロダクト URL**（`geo.baiyuan.io`）の CTA を配置済み
- **CC BY-NC 4.0** ライセンスであることを明記
- **文体**：である体（Qiita）／です・ます体の要所混在は note.com で調整可
- **ハッシュタグ**：Qiita は半角カンマ区切り、note は `#GEO` `#生成AI` `#B2Bマーケティング` 等

## 追加記事の案（将来）

- Qiita：「NLI + ChainPoll で AI ハルシネーションを検知する設計」（第 9 章ベース）
- Qiita：「Schema.org の 25 業種分類と @id 三層相互リンク」（第 7 章ベース）
- note.com：「B2B SaaS 企業の GEO 導入ロードマップ」（第 12 章ベース）
- note.com：「高 ACV 業界における Content Depth の測定と改善」（第 11 章ベース）

## 問い合わせ

- 誤記・質問：[GitHub Issues](https://github.com/baiyuan-tech/geo-whitepaper/issues)
- 導入相談：<services@baiyuan.io>

---

*本ディレクトリは百元科技（Baiyuan Technology）のホワイトペーパーを日本語圏で流通させるための素材群である。再配布・改変は CC BY-NC 4.0 ライセンスに従う。*
