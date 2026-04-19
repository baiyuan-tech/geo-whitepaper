---
title: "付録D — 図目録と制作ガイドライン"
description: "日本語版における全 Mermaid ダイアグラム・表・図解のマスターインデックス、配色・フォント・ツール推奨を含む制作スタイルガイド。"
appendix: D
part: 6
lang: ja
authors:
  - name: Vincent Lin
    affiliation: 百元科技 (Baiyuan Technology)
license: CC-BY-NC-4.0
last_updated: 2026-04-18
---

# 付録D — 図目録と制作ガイドライン

日本語版における全図をここにインデックス化する。`.md` ファイルに直接埋込まれた Mermaid ダイアグラムは GitHub 上でネイティブ描画され、その他の静的画像（ある場合）は `assets/figures/` に配置される。

## D.1 完全目録

| ID | 種類 | 主題 | 章 |
|----|------|---------|---------|
| 図 1-1 | xychart-beta | 2023〜2025年の情報チャネル移行 | [第1章](./ch01-geo-era.md) |
| 図 1-2 | 表 | SEO vs GEO 中核的差異 | [第1章 §1.3](./ch01-geo-era.md) |
| 図 2-1 | flowchart | 6段階クローズドループシステム | [第2章 §2.1](./ch02-system-overview.md) |
| 図 2-2 | flowchart | ブランドガバナンス全体サイクル | [第2章 §2.3](./ch02-system-overview.md) |
| 図 2-3 | flowchart | マルチテナント RLS + アプリ層二重保険 | [第2章 §2.5](./ch02-system-overview.md) |
| 図 3-1 | graph | 7次元レーダー隣接関係 | [第3章 §3.2](./ch03-scoring-algorithm.md) |
| 図 4-1 | xychart-beta | プラットフォーム障害時の3戦略 | [第4章 §4.3](./ch04-stale-carry-forward.md) |
| 図 4-2 | stateDiagram | プラットフォームデータ状態機械（Fresh/Stale/Expired/Reset） | [第4章 §4.3](./ch04-stale-carry-forward.md) |
| 図 4-3 | ASCII | フロントエンド isStale バッジ | [第4章 §4.6](./ch04-stale-carry-forward.md) |
| 図 5-1 | flowchart | modelRouter 2経路アーキテクチャ | [第5章 §5.2](./ch05-multi-provider-routing.md) |
| 図 5-2 | flowchart | 切替判断ツリー | [第5章 §5.5](./ch05-multi-provider-routing.md) |
| 図 6-1 | flowchart | 同一ブランドの2つのビュー | [第6章 §6.1](./ch06-axp-shadow-doc.md) |
| 図 6-2 | flowchart | AXP 3層構造 | [第6章 §6.2](./ch06-axp-shadow-doc.md) |
| 図 6-3 | flowchart | Worker ルーティング判断フロー | [第6章 §6.3](./ch06-axp-shadow-doc.md) |
| 図 6-4 | flowchart | 25種 AIボット UA の4グループ分類 | [第6章 §6.4](./ch06-axp-shadow-doc.md) |
| 図 6-5 | flowchart | SaaS自社ブランドのパスカテゴリ判断ツリー | [第6章 §6.5](./ch06-axp-shadow-doc.md) |
| 図 6-6 | コードブロック | JSON-LD フラット vs ネスト配列 | [第6章 §6.7](./ch06-axp-shadow-doc.md) |
| 図 7-1 | 表 | 25業種分類表 | [第7章 §7.2](./ch07-schema-org.md) |
| 図 7-2 | flowchart | 3層エンティティ・ナレッジグラフ | [第7章 §7.3](./ch07-schema-org.md) |
| 図 7-3 | flowchart | 実店舗 vs オンラインの重み差異 | [第7章 §7.4](./ch07-schema-org.md) |
| 図 7-4 | flowchart | ウィザード + 編集の2エントリ | [第7章 §7.6](./ch07-schema-org.md) |
| 図 7-5 | flowchart | GBP URL パーサ4分岐判断ツリー | [第7章 §7.7](./ch07-schema-org.md) |
| 図 8-1 | flowchart | 一方向データフロー：GBP → Schema → AXP → AI | [第8章 §8.1](./ch08-gbp-integration.md) |
| 図 8-2 | flowchart | 2つのホスティングモデル（Manager / OAuth） | [第8章 §8.3](./ch08-gbp-integration.md) |
| 図 8-3 | 表 | 同期頻度マトリクス | [第8章 §8.5](./ch08-gbp-integration.md) |
| 図 8-4 | flowchart | GBP統合4フェーズ・ロードマップ | [第8章 §8.7](./ch08-gbp-integration.md) |
| 図 9-1 | flowchart | 6段階クローズドループ | [第9章 §9.1](./ch09-closed-loop.md) |
| 図 9-2 | flowchart | ハルシネーション5分類体系 | [第9章 §9.2](./ch09-closed-loop.md) |
| 図 9-3 | flowchart | 3階層知識源のNLI統合 | [第9章 §9.3](./ch09-closed-loop.md) |
| 図 9-4 | flowchart | 中央共有RAGアーキテクチャ | [第9章 §9.4](./ch09-closed-loop.md) |
| 図 9-5 | flowchart | LLM Wiki 文書ライフサイクル | [第9章 §9.5](./ch09-closed-loop.md) |
| 図 9-6 | flowchart | ClaimReview 3経路注入 | [第9章 §9.6](./ch09-closed-loop.md) |
| 図 9-7 | flowchart | 2層スキャンの役割分担 | [第9章 §9.7](./ch09-closed-loop.md) |
| 図 9-8 | xychart-beta | ハルシネーション残存率収束曲線 | [第9章 §9.8](./ch09-closed-loop.md) |
| 図 10-1 | flowchart | 通常スキャン vs ベースライン | [第10章 §10.1](./ch10-phase-baseline.md) |
| 図 10-2 | flowchart | Phase 1/2/3 データ構造 | [第10章 §10.2](./ch10-phase-baseline.md) |
| 図 10-3 | flowchart | 4軸変化観察マトリクス | [第10章 §10.4](./ch10-phase-baseline.md) |
| 図 11-1 | graph | 5ブランド7次元レーダー（Wk 1 vs Wk 6） | [第11章 §11.2](./ch11-case-studies.md) |
| 図 11-2 | flowchart | プラットフォームカバレッジの非対称性（英語 vs ローカル） | [第11章 §11.3](./ch11-case-studies.md) |
| 図 11-3 | xychart-beta | 完成度 × シテーション率差分 | [第11章 §11.4](./ch11-case-studies.md) |
| 図 11-4 | xychart-beta | AXP前後のAIボットトラフィック | [第11章 §11.5](./ch11-case-studies.md) |
| 図 11-5 | pie | 顧客側落とし穴分布 | [第11章 §11.6](./ch11-case-studies.md) |
| 図 12-1 | 表 | 現行カバレッジマトリクス | [第12章 §12.1](./ch12-limitations.md) |
| 図 12-2 | flowchart | 今後の課題依存（短/中/長期） | [第12章 §12.4](./ch12-limitations.md) |

**合計**：13章、44図（Mermaid優位、ASCIIアート・表・コードブロックを少量含む）。

---

## D.2 制作スタイルガイド

### 1. 統一スタイル

以下のパレットを使用する（`%%{init: {'theme':'base'}}%%` + 個別 `style` オーバーライドで適用）：

| 用途 | Hex |
|-----|------|
| プライマリ（ノード境界、強調） | `#1e40af`（深青） |
| アクセント（強調、矢印） | `#ea580c`（明橙） |
| 警告（エラー、失敗経路） | `#dc2626`（赤） |
| 成功（修復、収束） | `#16a34a`（緑） |
| ニュートラル（背景、補助） | `#64748b`（スレート） |

Mermaid `style` 構文で特定ノードに適用できる：

```mermaid
flowchart LR
    A[node]
    style A fill:#1e40af,color:#fff
```

### 2. Mermaid を静的画像より優先

- **技術フロー**（flowchart / stateDiagram / sequenceDiagram）— 全て Mermaid
- **データ関係**（graph、ER）— Mermaid
- **トレンドチャート**（xychart-beta）— Mermaid
- **円グラフ**（pie）— Mermaid

理由：

- バージョン管理下で差分可能、読者がコピー・改変可能
- 外部画像ホスト依存無し（ホストは失敗しうる）
- AIクローラが Mermaid ソースコードを読取可能、図自体が構造化データ

### 3. 静的画像（SVG/PNG）を使うべき場合

以下の場合のみ `assets/figures/` 配下に配置：

- Mermaid で表現できない複雑な視覚（深くネストされたレイアウト、イラスト）
- プロダクトスクリーンショット（UIフレーム、顧客パネル例）
- Recharts 対話性が必要なチャート（PDFでは静的PNGになる）

全静的画像は：

- ソースファイルを保持（`.fig`、`.drawio`、Figma リンク）
- デフォルトはSVG（スケーラブル、軽量）
- 命名：`fig-<章>-<番号>-<slug>.svg`、例：`fig-11-03-completion-citation.svg`

### 4. alt テキスト

全静的画像に記述的 alt テキストを付与：

```markdown
![図 X-Y 主題：具体的内容<br>*図 X-Y：詳細キャプション*](assets/figures/fig-XX-YY-slug.svg)
```

alt テキストは画像を見られない読者に図の要点を伝達すべきである（視覚障害ユーザとAIクローラの両方に対応）。

### 5. 多言語版は独立制作

- 中国語版の図は英語版へ**翻訳しない**、英語慣習で再描画する
- 日本語版は日本語で独立描画、日本語フォント・慣習・フレーズを使用
- データ単位・日付形式・文字方向はロケール毎に調整
- 全言語版は図番号を共有（図 1-1 中国語 / 図 1-1 英語 / 図 1-1 日本語の並列）

### 6. プロダクトスクリーンショットの匿名化

UIスクリーンショットは：

- 顧客ブランド名を伏字化（*「Brand A」*、*「Example Brand」* を使用）
- 実数値をぼかし or 置換（例：`$12,345` → `$XX,XXX`）
- 内部URLパスを伏字化（構造のみ示し、具体的ドメインは示さない）
- メールアドレスをマスク（`user@***.com`）

### 7. Pandoc PDF ビルド時の Mermaid

Mermaid は GitHub Markdown、GitLab、Obsidian、VS Code Preview 上でネイティブ描画される。**Pandoc PDF 生成**には前処理ステップが必要：

- CI が `mermaid-cli` でコードブロックを SVG に事前描画
- または Pandoc フィルタ `pandoc-mermaid` で自動処理

本リポジトリの `build-pdf.yml` は v1.0-draft で用いられたプレースホルダ置換手法を含む（[`.github/workflows/build-pdf.yml`](../.github/workflows/build-pdf.yml)参照）。

---

## D.3 制作ステータス

| ステータス | 図数 | 比率 |
|--------|-------------:|------:|
| ✅ `.md` 内完成 | 44 | 100% |
| 🟡 プロダクトスクショ要取得 | 0 | 0% |
| ⚪ 日本語版再描画 | 0（日本語版全編公開済） | — |

---

**ナビゲーション**：[← 付録C：参考文献](./appendix-c-references.md) · [📖 目次](../README.md)
