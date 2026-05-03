---
title: "LinkedIn 投稿 — Vincent Lin 個人アカウント (ja)"
audience: "日本のエンジニア、SaaS 企業、AI 研究者"
recommended_post_time: "2026-05-05 21:00 GMT+8 (48h after zh-TW)"
char_count: ~1700
license: CC-BY-NC-4.0
---

# ja personal post

## 見出し (LinkedIn 非表示、SEO 用)

百原 GEO Platform ホワイトペーパー v1.1.1 — 16 章 5 万字、Zenodo DOI 公開

## 本文 (コピペ用)

---

過去 2 年間の開発成果をホワイトペーパーにまとめて Zenodo に公開しました:

📖 *Baiyuan GEO Platform: A Whitepaper on Building a SaaS for Generative Engine Optimization*

DOI: https://doi.org/10.5281/zenodo.19994035
GitHub: https://github.com/baiyuan-tech/geo-whitepaper

マーケティング資料ではありません。16 章 5 万字の、実際の実装記録と踏んだ地雷の正直な記録です。

【中身の要点】

✅ **AI 引用率 7 次元評価アルゴリズム** — ChatGPT / Claude / Gemini が自社ブランドを正確に言及しているかを定量化する仕組み

✅ **AXP シャドウ文書配信** — 顧客サイトに SEO がなくても、Cloudflare Workers を顧客ドメイン上で動かし、AI ボット限定で Schema.org / llms.txt / sitemap を配信

✅ **F12 三層構造オプティマイザー** — V1 ルールベース + V3.1 デュアルエンジン (AutoGEO [arXiv:2510.11438] + E-GEO [arXiv:2511.20867]) を逐行ポート、DB レベルの placeholder guard で偽の paper_table_ref 混入を防止

✅ **RAG マイクロサービスにおける LLM ハルシネーション 6 層防御** — 本番で実際に踏んだバグ:
   • LLM が部分 UUID を吐いて KB compile 全体が中断
   • Docker DNS デュアルフォールバック (内部 DNS のみだと Google OAuth がプラットフォーム全体で機能停止)
   • 推論モデルが `maxTokens` を推論で使い切り、本文が空になる現象

✅ **Schema.org 三層エンティティ知識グラフ** + 閉ループのハルシネーション検知・自動修復

✅ **プラットフォーム SSOT チェーン** — 1 万テナント SaaS で、ページ間・テーブル間・クローラー間のデータをどう単一事実源で統一するか

【正直に書いた、まだ解けていないこと】

各章 12 / 14–16 の末尾に「Open Problems」節を設けています:

⚠️ L3 S3 キャッシュレイヤー未デプロイ
⚠️ Phase 1→2→3 エスカレーションロジックは仕様準拠だが本番負荷では未検証
⚠️ Wiki フォールバック記事の品質が end-to-end 評価未実施
⚠️ ME カスタムエンジンのメタデータゲーティングが厳しすぎる (個人 IP 17 ブランド中 5 ブランドが企業トーンへフォールバック)

成功事例集ではありません。傷口の見える、エンジニアリングの現場日誌です。

【どんな方が読むと有益か】

- マルチテナント SaaS を構築中のエンジニア (アーキテクチャパターンをそのまま流用可能)
- 生成 AI の引用メカニズムを研究する研究者 (引用論文は実在の arxiv のみ、DB トリガーが偽装メトリクスを排除)
- AI マーケター / ブランド担当者 (非エンジニア向け Ch 1 + Ch 11 に 5 ブランドの実データ)

ライセンスは CC BY-NC 4.0。引用・翻案・翻訳自由。GitHub Issues で議論歓迎。各章末に Schema.org JSON-LD を埋め込み済み。

この領域で仕事をしている方、ぜひ意見交換しましょう。

<!-- markdownlint-disable-next-line MD018 -->
#GenerativeEngineOptimization #GEO #AISearch #SaaS #LLM #OpenScience #Zenodo #SchemaOrg #Multitenant #PostgreSQL #BullMQ #Cloudflare #生成AI
