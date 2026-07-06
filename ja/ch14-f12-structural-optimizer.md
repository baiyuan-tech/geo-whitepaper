---
title: "第 14 章 — F12 三層構造オプティマイザ:ルールベース V1 から双エンジン v3.1 へ"
description: "F12(TLSO, Three-Layer Structural Optimizer)のアーキテクチャ進化史:V1 のルールベーススコアリング + LLM rewrite、V3.1 の AutoGEO + E-GEO 双エンジン並行 + Hybrid 統合、さらに Plan Gate / Multi-Layer Cache / Tier Priority / Scale Trigger などの 1 万テナント基盤。"
chapter: 14
part: 5
word_count: 7200
lang: ja
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - F12
  - TLSO
  - Three-Layer Structural Optimizer
  - AutoGEO
  - E-GEO
  - Hybrid Engine
  - Adaptive Router
  - LLM Gateway
  - Plan Feature Gate
  - Multi-Layer Cache
  - Tenant Isolation
  - Scale-up Trigger
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/ja/ch14-f12-structural-optimizer
---

# 第 14 章 — F12 三層構造オプティマイザ:ルールベース V1 から双エンジン v3.1 へ

> 「AI 引用率の最適化」を、勘に頼った prompt 調整から、測定可能・再現可能・スケール可能なエンジニアリングシステムへ変える。

## 目次 {.unnumbered}

- [14.1 なぜ F12 が必要か](#141-なぜ-f12-が必要か)
- [14.2 V1:三層分析 + LLM Optimizer](#142-v1三層分析--llm-optimizer)
- [14.3 V3.1:双エンジン並行 + 統合戦略](#143-v31双エンジン並行--統合戦略)
- [14.4 1 万テナント基盤](#144-1-万テナント基盤)
- [14.5 デプロイ、クォータ、課金の 3 点](#145-デプロイクォータ課金の-3-点)
- [14.6 観察された制限と未解決問題](#146-観察された制限と未解決問題)
- [14.7 エンジニアリングの教訓](#147-エンジニアリングの教訓)

---

## 14.1 なぜ F12 が必要か

### 14.1.1 Ch 3 スコアリングが残した問題

Ch 3 の七次元スコアリングは「ブランド引用率はいくつか」に答えるが、**「次の記事をどう書けば引用率が上がるのか」は教えてくれない**。商業顧客は実際にスコアを手にしてこう問う:

> 「AI 引用率が 32 点だと分かった。次の blog をどう直せば 60 まで上がる?」
>
> 「この fact-check は、なぜ ChatGPT は引用するのに Gemini は引用しないのか?」
>
> 「同じ内容を書き方を変えると、引用率はどれだけ変わるのか?」

スコアリングシステムは「測定」しか担わず、「処方」は担わない。処方を marketing のコピーライターや外注に丸投げするのは、SaaS の核心価値を人間の経験に漏らすのと同じ——そして人間の経験は 1 万テナント scale で一つ一つ手調整することは不可能である。

### 14.1.2 初期の手動 prompt 調整における 3 つの失敗モード

V1 以前(2026 年 1〜2 月)、プラットフォームには「LLM コンテンツ書き換え」の internal tool があり、エンジニアが ad-hoc に OpenAI playground で顧客コンテンツを回していた。3 ヶ月で頓挫した。3 つの失敗:

1. **コスト暴走**:単一顧客のコンテンツは平均 4〜6 回のリランで満足に至る。単位 cost ≈ $0.50/篇、30 顧客 × 22 page_type = 660 回で $330。月 2 回の完全再生成で $660、retry を数えていない。顧客の月額は $99 から——粗利はマイナス。
2. **再現不可**:同じ prompt でも翌日は違う結果(GPT-4 のデフォルト temperature は 1)。顧客は「なぜ前回はこう書いて、今回はこうなのか?」と問うが、エンジニアは答えられない。
3. **品質劣化**:LLM「最適化」後、顧客は自社コンテンツを認識できなくなる。ある匿名事例:保険営業員の「貯蓄保険の商品紹介」が GPT-4 によって「投資アドバイス」に書き換えられ(prompt が informativeness を強調していたため)、金管会の規定に違反し、顧客は危うく通報されかけた。

F12 という名前は Web DevTools の F12 キーに由来する——押すとブラウザの「構造インスペクタ」が開く。この比喩を LLM 引用率最適化に持ち込んだ:**あらゆるコンテンツは「F12 一発」で三層の構造スコアを見られ、必要に応じて書き換えられるべきである。**

### 14.1.3 F12 と Ch 3 スコアリングの違い

| | Ch 3 スコアリング | F12 |
|---|---|---|
| 入力 | ブランド次元 + AI プラットフォーム次元 | 一段のコンテンツ(URL / テキスト / Markdown) |
| 出力 | 0–100 点 + シグナル分項 | 三層スコア + 最適化後コンテンツ |
| 用途 | 「自ブランドは健全か」 | 「この記事をどう直せば引用されるか」 |
| 対象 | brand-level | content-level |
| 頻度 | 毎日 / 週次スキャン | 記事書き込みごとにトリガー |
| コスト | scan platform call(LLM 1 回/クエリ) | analyzer + optimizer(規則 + 必要に応じて LLM) |

**F12 の核心的な約束は、同一のコンテンツを与えたとき、システムが複数のバリアントを生成し、LLM が最も引用しそうなものを選び出せることである。**

### 14.1.4 既存 SEO ツールとの違い

SurferSEO / Clearscope などの SEO ツールの核心は依然として「keyword density / TF-IDF」であり、目標は Google 検索順位である。F12 の目標は LLM 引用であり、両者のシグナルは重ならない:

- SEO ツールは keyword の出現回数を見る;F12 は「atomic fact 密度」(数字、日付、ソースリンク)を見る
- SEO ツールは backlink quality を見る;F12 は「実体マークアップ」(Schema.org Person/Organization/Service が完全か)を見る
- SEO ツールは content length を見る;F12 は「段落が LLM に直接摘録されうるか」(TL;DR、lead-in、明確な H1)を見る

両者は対立しない——良い SEO コンテンツはたいてい良い GEO コンテンツでもあるが、逆は成り立たない。F12 が補うのは「LLM-specific」なシグナルであり、これは従来の SEO ツールが全く見られないものである。

---

## 14.2 V1:三層分析 + LLM Optimizer

### 14.2.1 三層構造の設計

| 層 | 粒度 | 検査内容 |
|---|---|---|
| **Macro** | 文書レベル | ≥1 篇の fact-check / FAQ / comparison があるか?明確な H1 があるか?entity を宣言しているか? |
| **Meso** | 段落レベル | 段落長の分布、TL;DR が前にあるか、LLM の摘録を誘う lead-in 文があるか |
| **Micro** | 文レベル | 数字 / 日付 / ソースリンク密度、実体(entity)マークアップ、引用可能な atomic fact があるか |

各層は独立して採点(0–100)、最終的に `overall = 0.4 × macro + 0.35 × meso + 0.25 × micro`(重みは `scoring_configs.f12_thresholds_<brand_type>` SSOT に格納、admin がコード変更なしに調整可能)。

**なぜ三層か**:LLM が「何を引用するか決める」とき、経験的に見るのは 3 つの尺度である:

- **文書レベル**:この記事は「信頼できる」コンテンツか(fact-check があり、entity が明確か)?
- **段落レベル**:直接抜き出せる「答えの段落」があるか(LLM は TL;DR モードを好む)?
- **文レベル**:単一の文が「引用可能な事実」か(数字、日付、明確な主語がある)?

いずれかの層が欠けると、LLM 引用確率は 30〜50% 落ちる。2026 Q1 に 3 ヶ月の内部実験を行い、三層がそれぞれ独立して有効であることを確認した——macro を外すと引用率は 31% 落ち、meso を外すと 47% 落ち、micro を外すと 38% 落ちた。

### 14.2.2 なぜ V1 は LLM を使わない(規則式のみ)か

V1 のアナライザは**純粋に規則式**である:正規表現 + AST 解析 + 統計密度、LLM を使わない。毎秒 30 篇以上を分析でき、batch の全プラットフォームスキャンに適する。

設計理由:

1. **コスト**:30 brand × 22 page = 660 篇/週スキャン、LLM 分析は 1 篇 $0.02 = $13.2/週、規則式はほぼ 0
2. **再現可能**:規則式は同一コンテンツに対して永遠に同点、LLM は ±5 点の揺れがある
3. **監査可能**:顧客が「なぜ macro が 60 点か」と問えば、システムは「fact-check なし / H1 が長すぎ / entity なし」という具体的 issue を列挙できる;LLM は「権威が足りない感じ」としか答えられない
4. **速度**:規則式は 30 篇以上/秒、LLM は 0.5 篇/秒、60 倍の差

具体的な規則の例(抜粋):

```yaml
macro:
  - has_fact_check: 1 if brand has any fact-check page
  - has_clear_h1: 1 if first <h1> within 100 chars and ≥10 chars
  - has_entity_declaration: 1 if Schema.org Person/Org/Service in JSON-LD

meso:
  - avg_paragraph_chars: weighted by closeness to ideal range
  - has_tldr_in_first_300: 1 if "TL;DR" / "簡言之" / "重點" within first 300 chars
  - lead_in_score: count of paragraphs starting with question/statement hook

micro:
  - number_density: count(<digit>+) / total_chars
  - date_density: count(YYYY-MM-DD | YYYY 年 X 月 | etc) / total_chars
  - source_link_density: count(<a href>) / paragraph_count
  - entity_mention_count: count of Schema.org-marked entities in text
```

各規則の閾値(`paragraph_ideal_min`、`number_density_target` など)はすべて `scoring_configs` SSOT にあり、admin はコードを変えずに調整できる。踏んだ落とし穴:閾値をコードにハードコードすると、変更のたびに deploy + 再起動が必要で、顧客は新しいスコアを見るまで 30 分待つ;SSOT に変えてからは即時反映になった。

### 14.2.3 LLM Optimizer の設計

スコアが < 70(`scoring_configs.f12_score_thresholds.low_score`)の場合、**自動的にキューに入り** LLM Optimizer を通る:

```text
[低スコアページ] → axp_pages.content_md を取得 → LLM へ投入
       → 与える:原文 + 三層分析 issues + 参考 templates
       → 要求:スコアを ≥ 20 上げるよう書き換えるが、cosine 類似度 ≥ 0.90 で脱線しない
       → axp_page_history に書き込む(双方向 rollback 可)
       → analyzer を再実行し、新スコア ≥ 元スコア + 15 を確認
       → axp_pages に書き込み needs_recompile をマーク
```

LLM prompt の構造(簡略版):

```text
你是 GEO 內容優化專家。下面是一篇分數 X 的內容,issue 為:[列表]
請改寫,目標:
1. 修補列出的 issue
2. 保持原意(別偏題,別加客戶沒說的事實)
3. 段落結構偏好 TL;DR + 條列 + 來源連結

可參考的 5 個 template 風格:[列表]

原文:
[content_md]

改寫後輸出 JSON:{ rewritten: "...", reasoning: "..." }
```

`min_similarity ≥ 0.90` は spec の定錨点である(0.85 から 0.90 へ引き上げ、migration 186)。もとの 0.85 は一見妥当に思えたが、実測で 2 つの落とし穴を踏んだ:

1. **保険→投資アドバイスの脱線バグ**:ある保険営業員の axp_pages がスコア低で、Optimizer が「informativeness を上げる」ために「貯蓄保険、月払い 5000、20 年期、保証利率 1.8%」を「投資理財アドバイス:貯蓄保険を ETF 投資に切り替えることをお勧めします」に書き換えた。cosine 0.86 で 0.85 の門を通ったが、内容はすでに違法だった。
2. **個人 IP→企業紹介のドリフトバグ**:ある personal_ip brand の「個人の専門性」が Optimizer によって「当社は以下のサービスを提供します」に書き換えられ、語調が完全に合わなくなった。cosine 0.88 だが brand_type が個人から企業へ変わってしまった。

0.90 に上げた後、前者は 0.83 に下がってブロックされ、後者も 0.89 に下がってブロックされた。代償は一部の妥当な書き換えもブロックされること(約 12% の false negative)だが、「違法コンテンツが本番に出る」リスクに比べれば、この寛容度の狭さは許容できる。

### 14.2.4 双方向可逆(rollback)の設計

`axp_page_history` テーブルは Optimizer による各書き換え前の snapshot を保存し、admin UI からワンクリックで rollback できる。設計動機:

V1 の初期には history テーブルがなく、あるとき cron rerun のバグで 30 brand の fact-check ページがすべて書き換えられた。調べると、RAG 失効で空の ground_truth を受け取った Optimizer が空疎な内容を生成していた。当時は rollback できず、force-refresh RAG + pipeline 再実行しかなく、30 brand × 22 page = 660 LLM call をすべて回し直し、5 時間後にようやく復旧した。

`axp_page_history` を追加してからは、同種の事象は 30 秒で rollback できる。代償は DB が年間 ~40 GB 増えること(`content_md` 平均 8 KB × 30 brand × 22 page × 365 日 × 1.5 のリラン率)だが、一度の障害で 5 時間の downtime に比べれば、それだけの価値がある。

### 14.2.5 axpPageWriter hook の全カバレッジ

`axp_pages` に書き込むあらゆる経路(9 個の generators + 4 legacy + admin 手動編集)は**すべて F12 analyze を hook** し、新しいコンテンツが即座に採点され、閾値を下回れば即座に optimizer にキューされることを保証する。カバレッジ検証:

- ✅ `axpPageWriter.service.js#upsertAxpPage`(主経路)
- ✅ `hybridCoordinator.service.js`(6 核心類)
- ✅ `factCheckGenerator.js#refreshFactCheckPage`(直 SQL、hook を追加)
- ✅ `axp.controller.js#updatePageContent`(admin 編集)

漏れは weekly backfill cron(`0 2 * * 0`)がセーフティネットとして拾う。

**なぜ hook 全カバレッジが難しいか**:9 個の generators は同じチームが同じ sprint で書いたものではない——`overviewGenerator` は V1、`comparisonGenerator` は V2、`factCheckGenerator` は V2.5、`featuresGenerator` は V3 が書いた。各 generator の「axp_pages への書き込み」ロジックは少しずつ異なる(service を使うもの、直 SQL を使うもの、service を迂回して直接 INSERT するもの)。

コードベース全体を 1 週間 grep して「axp_pages への書き込み」全経路を洗い出した:

```bash
grep -rn "INSERT INTO axp_pages\|UPDATE axp_pages\|axp_pages.*INSERT\|axp_pages.*UPDATE" backend/src/
```

13 個の入口を見つけ、一つずつ F12 hook を補った。最後に weekly backfill をセーフティネットとして追加——将来 hook 漏れの新 generator が加わっても、backfill が日曜 02:00 に全プラットフォームを再分析し、漏れを補う。実測では weekly backfill は一度 ~12 分(30 brand × 22 page = 660 篇、約 1 篇/秒)。

### 14.2.6 5 つの cron の時系列設計

| Cron | スケジュール | 用途 |
|------|------|------|
| `f12-weekly-backfill` | `0 2 * * 0` | 全 brand × 22 page F12 再分析 + brand_faq 健全性チェック |
| `f12-trends-refresh` | `30 2 * * *` | REFRESH MATERIALIZED VIEW `structural_score_trends` |
| `f12-low-score-optimizer` | `0 4 * * *` | LLM rewrite 低スコアページ、双方向可逆 |
| `f12-immediate-optimize` | priority queue | axpPageWriter hook トリガー、score < 70 で即キュー |
| `f12-retention-cleanup` | `50 3 * * *` | 30 日で古い optimization runs を掃除(後に v3.24 retention に吸収) |

時間帯選定のロジック:

- **02:00 UTC = 台湾 10:00**:プラットフォームトラフィックが最低の時間帯(台湾顧客は出社したばかりでまだ大量操作していない)。weekly backfill は 12 分で影響は無視できる。
- **02:30 UTC**:trends MV refresh は ~30 秒、backfill の後に配置する必要がある(後者はデータを書き、前者は読む)。
- **03:50 UTC**:retention cleanup は trends の後(後者は履歴データに依存)。
- **04:00 UTC = 台湾 12:00**:LLM Optimizer が全プラットフォームで走る。この時間帯を選んだのは、Anthropic / OpenAI の RPM ピークが通常 UTC 04:00 には過ぎているため(米西海岸の未明、誰も prompt を書いていない)。実測では同じ prompt が UTC 04:00 と UTC 14:00 で、後者は平均 1.8 倍遅い(rate limit の待ち行列)。

priority queue は即時トリガーで cron 表には入らない——axpPageWriter hook が score < 70 の新コンテンツを検知すると即座に enqueue し、worker が 1〜2 分以内に処理を完了する。

---

## 14.3 V3.1:双エンジン並行 + 統合戦略

V1 が 4 ヶ月走った後、2026 年に発表された 2 篇の arxiv 論文から「もう 1 桁上げる」手がかりを見つけた。

### 14.3.1 2 つの学術的源流

**AutoGEO**(arXiv:2510.11438, github.com/cxcscmu/AutoGEO):

CMU チームが発表。核心手法は**何千もの「LLM に引用された vs されなかった」コンテンツの対比から、汎化可能な書き換え規則を自動 mining する**ことである。実験範囲:

- 25 条の rule、2 つの dataset に分かれる:
  - Researchy-GEO(学術コンテンツ)+ Gemini:15 条
  - E-commerce(EC コンテンツ)+ Gemini:10 条
- 各 rule の形式は LLM-readable instruction(例:「冒頭にこの文が論じる主題を明確に宣言する一文を加える」)
- 実験 LLM は主に Gemini-2.0-flash + GPT-4o + Claude-3.5

長所:**実際の対比実験から derived された**、直感ではない
短所:rule 集合が相対的に狭い(25 条のみ)、かつ大部分が Gemini 向け

**E-GEO**(arXiv:2511.20867, github.com/psbagga17/E-GEO):

独立研究者が発表。核心手法は**異なる prompt スタイル(authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling)から生成する**ことで、各スタイルが異なる LLM 上でどれだけ引用率が違うかを実測する。

- 15 個の template(= 15 個の prompt スタイル)
- 各 template は一段の「書き換え instruction」(例:authoritative = 「権威ある語調で書き換え、ソース、データ、引用を加える」)
- 実験範囲は GPT-4o / Claude-3.5 / Gemini-2.0 / DeepSeek-V3 を横断

長所:**template が多様**で、異なる use case に適合
短所:**template ごとの uplift 数値が示されていない**(全体的な実験 macro 結論のみ)

### 14.3.2 なぜ両方を port するのか、1 つを選ばないのか

2 篇の方法論は補完的である:

- AutoGEO の rule は**「客観的な対比実験から derived」**であり、batch 自動化に適する(F12 の大部分の case)
- E-GEO の template は**「人手で定義した prompt スタイル」**であり、「特定の語調が必要な」case に適する(例:顧客が whitepaper に authoritative tone を求める)

両者を**字面通り**(line-by-line)にシステムへ port し、**いかなる paper_table_ref / expected_uplift も捏造しない**(原論文は逐条の mapping を示しておらず、書けばエンジニアリング憲法 #1「データを模擬しない」に違反する)。

> ⚠️ 重要:V3.1 の `autogeo_rules` テーブルの初版 seed は捏造された paper_table_ref(`'Table 3'` など)+ 捏造された expected_uplift(`1.32-1.92`)を含み、エンジニアリング憲法 #1 の trigger に阻止されて、実際の arxiv ソースコードから再 port した。後に DB レベルの placeholder guard trigger を追加し永久に再発を防いだ。14.3.5 を参照。

### 14.3.3 三エンジンアーキテクチャ

```text
                 ┌──────────────────┐
                 │ adaptiveRouter   │ ← 依 brand tier / page_type / use case 路由
                 └─────────┬────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
  ┌──────────┐       ┌──────────┐       ┌──────────┐
  │ AutoGEO  │       │ E-GEO    │       │ Hybrid   │ ← parallel call,挑 improvement 高的
  │ engine   │       │ engine   │       │ engine   │
  └──────────┘       └──────────┘       └──────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
                    ┌─────────────┐
                    │ ME custom   │ ← personal_ip 專屬,從 personal_profiles 守門
                    │ engine      │
                    └─────────────┘
```

4 つの engine はすべて**標準化された EngineResult** schema を吐く:

```typescript
interface EngineResult {
  engine: 'autogeo' | 'egeo' | 'hybrid' | 'me_custom';
  rule_or_template_id: string;         // arxiv source 或 internal id
  source: string;                       // 'arXiv:2510.11438' 等
  original_content: string;
  rewritten_content: string;
  diff: { additions, deletions, similarity };
  scores: { before, after, delta };
  llm_call: { provider, model, tokens, cost };
  accepted: boolean;                    // optimizer 自決(score 提升 ≥ 15 + similarity ≥ 0.90)
}
```

`executeRoute` が統一して audit + persist + cache する。Hybrid engine の「並行」実装:

```javascript
const [autogeoResult, egeoResult] = await Promise.allSettled([
  Promise.race([autogeoEngine.optimize(input), timeout(30_000)]),
  Promise.race([egeoEngine.optimize(input), timeout(30_000)]),
]);

const candidates = [autogeoResult, egeoResult]
  .filter(r => r.status === 'fulfilled' && r.value.accepted)
  .map(r => r.value);

if (candidates.length === 0) {
  return { engine: 'hybrid', accepted: false, reason: 'all_engines_failed' };
}

return candidates.reduce((best, c) =>
  c.scores.delta > best.scores.delta ? c : best
);
```

`Promise.allSettled` は単一 engine の失敗が他方に影響しないことを保証する;30 秒 timeout は LLM のハングを防ぐ(プラットフォーム憲法「LLM call は必ず ≤60s timeout を持つ」に対応)。

### 14.3.4 ルーティング決定

`adaptiveRouter.decide({ brandId, contentType, ... })` は以下の優先順位で `{ path, template, target_engine }` を返す:

1. **plan gate**:starter は一律 E-GEO `authoritative`(コスト最低);Pro でようやく sync API を使える;Enterprise+ でようやく Hybrid と AutoGEO を使える
2. **page_type → template**:`f12_page_type_to_template` SSOT(22+1 page_type → 5 template category)
3. **content_type 特化**:whitepaper / case_study / fact_* は強制で E-GEO `authoritative`;FAQ は `FAQ`;product は `clickable`
4. **brand_type fallback**:personal_ip は `me_custom`(この 1 条だけが baiyuan internal template)

完全な decision tree:

```text
adaptiveRouter.decide(input)
├── if brand.tier === 'starter':
│     return { path: 'egeo', template: 'authoritative' }
├── if input.contentType in ['whitepaper', 'case_study', 'fact_check']:
│     return { path: 'egeo', template: 'authoritative' }   ← 強制 authoritative
├── if input.contentType === 'faq':
│     return { path: 'egeo', template: 'FAQ' }
├── if brand.brand_type === 'personal_ip':
│     return { path: 'me_custom' }                         ← ME 守門
├── if brand.tier in ['enterprise', 'group']:
│     return { path: 'hybrid' }                            ← 雙引擎並行
└── default (Pro tier general content):
      template = mapPageTypeToTemplate(input.page_type);
      return { path: 'egeo', template };
```

**なぜ plan gate が router + executeRoute の 2 層なのか**:router 側で決定時にブロックするが、executeRoute 側は defensive な第 2 層である——router のバグや SSOT config の誤設定があっても、executeRoute が再度 brand.tier と path を照合し、合致しなければ直接 reject する。一度踏んだ:あるとき router config を変更した際 starter の path を変え忘れ、starter brand が 3 回 hybrid call を回してしまった(3× LLM cost)が、executeRoute の第 2 層が止められなかった。defensive plan_gate を追加してからは同種の事象は起こりえなくなった。

### 14.3.5 placeholder guard trigger の物語

V3.1 の `autogeo_rules` テーブルの初版 seed(migration 189)は重大事象を起こした——**捏造された paper_table_ref**。

当時の spec には「各 rule は paper Table 3 の第 N 行に対応」と書かれており、エンジニアは直感で `'Table 3'` / `'Table 4'` を入れたが、実際に arXiv:2510.11438 論文をめくると「rule X が Table Y の第 Z 行に対応」という mapping は存在しなかった。同時に `expected_uplift` 欄にも `1.32-1.92` の範囲を入れたが、論文実験は全体的な実験 uplift しか示しておらず、逐条ではなかった。

エンジニアリング憲法 #1「データを模擬しない」の trigger が阻止した:

```sql
CREATE OR REPLACE FUNCTION autogeo_rules_placeholder_guard()
RETURNS trigger AS $$
BEGIN
  IF NEW.paper_table_ref IS NOT NULL
     AND NEW.source NOT LIKE 'arXiv_%'
     AND NEW.source NOT LIKE 'arXiv:%' THEN
    RAISE EXCEPTION 'paper_table_ref must be NULL or source must be arXiv (憲法 #1)';
  END IF;

  -- 6 patterns 偵測 placeholder
  IF NEW.rule_description ~ '(待填入|To be filled|Actual Data Pending|TODO:|FIXME:|XXX)' THEN
    RAISE EXCEPTION 'placeholder pattern detected in rule_description (憲法 #1)';
  END IF;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_autogeo_rules_placeholder_guard
BEFORE INSERT OR UPDATE ON autogeo_rules
FOR EACH ROW EXECUTE FUNCTION autogeo_rules_placeholder_guard();
```

migration 191 は `TRUNCATE ... RESTART IDENTITY CASCADE` で捏造 seed を消し、migration 192 で trigger を追加し、実際の arxiv ソースコードから再 port した。`autogeo_rules` テーブルの 25 条はすべて `source = 'arXiv:2510.11438'` で、以下の実データを含む:

- `target_engine ∈ {gemini, gpt, claude}`
- `target_domain ∈ {researchy_geo, geo_bench, ecommerce}`
- `paper_table_ref` はすべて NULL(arxiv は逐条の mapping を示さない、憲法 #1 は捏造しない)
- `expected_uplift` はすべて NULL(全体的な実験 uplift であり逐条ではない — V2 の実際の ML でようやく埋める)

`egeo_templates` の 15 条はすべて `source = 'arXiv:2511.20867'` で、template 名は論文の表 1 と完全に一致(authoritative / technical / unique / fluent / clickable / diverse / quality / competitive / trick / format / FAQ / advertisement / language / minimalist / storytelling)。

百原内部の 1 条 `me_personal_ip` は明示的に `source = 'baiyuan_internal_v3_1'` とし、arxiv の命名に混ぜない。

**この trigger は「後知恵」である**——一度捏造されて初めて取り付けたものだ。しかし取り付けた後は、将来のいかなる admin 編集 / 新 generator / cron 障害も入り込めない。1 万テナント scale ではこれが必要なハードウェア層の保障である。

---

## 14.4 1 万テナント基盤

V3.1 は 9 個の batch で規模化基盤を敷いた(v3.21 → v3.27 の 8 日間)。各 batch は 1 つのボトルネックを解決し、9 個が合わさってシステムを「30 brand を回せる」から「設計上 10000 brand を回せる」へ進化させた。

### 14.4.1 クォータと課金(spec §4.3 + §4.4)

```text
TenantQuotaService(monthly_optimizations / max_content_size_kb)
  starter:  100 ops, 50 KB
  pro:      1000, 200
  enterprise: 10000, 1000
  group:    ∞, 5000

BillingTracker(每次 LLM call 記 cost)
  9 model × per-token pricing(claude_haiku $0.0003 / opus $0.015 / 4o-mini $0.0002 ...)
  in/out 倍率 4×
```

`f12_quota_usage` テーブルの monthly counter UPSERT(`tenant_id+brand_id+action+period` が一意)、`f12_billing_records` は年間 12M 行の単一テーブルで十分(1 万 × 100 opt × 12 月)。`quota_enforcement_enabled` flag はデフォルト `false`(観察期は usage を記録するだけでブロックしない)、`billing_recording_enabled` はデフォルト `true`(常に記録)。

「観察期」の設計:リリース直後は顧客に「今月 X / 上限 Y を使用」と見せるがブロックしない。分布が妥当(starter 顧客が月 5000 回使う outlier が出ない)であることを ~2 週間観察してから enforcement に切り替える。実測の観察期データ:

- starter は平均月 18 ops(中央値 5)、95th percentile 47——100 の上限を大きく下回る
- pro は平均月 234 ops、95th = 612——1000 を下回る
- enterprise は平均 1820 ops、95th = 4500——10000 を下回る

enforcement に切り替えた後、ブロックされた顧客は 0 個で、クォータ設計が妥当であることを示す。

### 14.4.2 Plan Feature Gate(spec §9)

`f12_plan_features` SSOT は 4 tier × 6 flag(`scoring_configs` テーブル内の JSONB):

```yaml
egeo_template_count:    starter=0    pro=5     enterprise=15  group=15
autogeo_rules:          'none'       'none'    'full'         'full'
engine_specialization:  false        false     true           true
hybrid_engine:          false        false     true           true
sync_api:               false        true      true           true
```

非 Enterprise が `whitepaper` / `case_study` / `fact-*` を走らせる場合は強制で E-GEO `authoritative` に切り替わる(AutoGEO は使わない);Enterprise+ のみ Hybrid + 三エンジン特化を使える。`adaptiveRouter` 内の plan gate + `executeRoute` の defensive plan_gate の 2 層で阻止する。

設計動機:starter / pro 顧客は月額が低く($99 / $499)、最も高価なエンジン(Hybrid は一度に 2 つの LLM を呼ぶ)を使わせられない、さもなくば粗利がマイナスになる。enterprise+ は月額 $2000+ で Hybrid のコストを負担できる。

`egeo_template_count` の設計:starter 0(デフォルトの authoritative のみ)、pro 5(authoritative / FAQ / clickable / fluent / diverse)、enterprise+ は全 15。顧客は「より多くの template を有効にするには Enterprise へアップグレード」という upsell を見る。

### 14.4.3 Multi-Layer Cache(spec §5.2)

```text
L1 Redis (5 min TTL)        ← 同 content + decision 重複 query
L2 PG f12_result_cache (7d) ← 跨 process / 跨 instance
L3 S3 hook (TBD Phase 3)    ← cross-region replication 需要時
```

cache key = `sha256(content + decision.path + template + target_engine)`、**tenant_id を含まずクロステナントで共有**(spec が明文で許可、内容が同じ + 規則が同じ → 結果は同じはずで、共有すれば LLM cost を N 倍削減できる)。

`accepted=true` の結果のみ cache する(cache poisoning を避ける)。`f12_cache_metrics` は hourly に L1/L2/miss を集計 → admin が 70% hit rate 目標を監視。

**クロステナント共有のプライバシー境界**:一見 sketchy に見える(A 顧客のコンテンツ cache を B 顧客が取れるのか?)が、実際に分析すると:

- cache key は**コンテンツ全段**の sha256 を含み、content が完全一致した場合のみヒットする
- 顧客 A のコンテンツは 99.9% 顧客 B と完全一致しない(content は KB 規模、sha256 は等価)
- ヒットする唯一のシナリオ:同一の generic content(FAQ template「GEO とは何か?」など)が複数顧客に使われる——この case での共有は妥当(顧客の私有情報がない)

実測で 1 ヶ月の cache hit rate は 38%(L1+L2)、70% 目標を下回る。原因:`content` が brand-specific 情報(brand name、entity など)を帯びるため、ロジックが同じでも内容がすでに unique 化している。Phase 3 でやりたいのは「**cache key の正規化**」——brand-specific token を `<<BRAND_NAME>>` placeholder に置換してから hash すれば、理論上は類似 brand が cache を共有できる。これは V2 の作業である。

### 14.4.4 LLM Gateway(spec §5.3)

per-model のプラットフォーム層 RPM 制限(in-memory sliding window 60s):

```yaml
claude_haiku:  10000 / min
sonnet:         2000
opus:            500
gpt-4o:         5000
gpt-4o-mini:   10000
gemini-flash:  20000
qwen_direct:   30000
deepseek_v4:    8000
```

5 個の F12 LLM call site はすべて `gatewayAiCall` を通る:autogeo / egeo / meCustom / V1 optimizer / hybrid は前 2 者を経由して連なる。`opts.billing` が提供されれば自動的に record する(後方互換)。

**なぜプラットフォーム層の RPM 制限が必要か**:Anthropic / OpenAI の API key は per-organization で、1 万テナントが 1 本の key を共有し、単一顧客の burst が全プラットフォームの quota を使い切る。Gateway は LLM call の前に現在の RPM をチェックし、超えたらキューまたは reject する。

実測シナリオ:あるとき admin UI のバグで 1 つの brand が 100 回 hybrid optimize を繰り返しトリガーし、各回 2 つの LLM call、200 個の call が同時に Claude Sonnet を打った。Gateway がなければ直接 Anthropic の 30 RPM 制限を超え、全プラットフォームの全 brand がブロックされる;Gateway があれば我々の側でブロックされ、その brand だけが影響を受け、他は正常に継続した。

**制限**:in-memory bucket は per-process で、単一 backend container なら OK;multi instance では Redis sliding-window が必要(hook は残してあるが未実装)。1 万テナント scale では backend が少なくとも 5〜10 個の instance を持つ見込みで、Phase 3 では必ず Redis-based に切り替える必要がある。

### 14.4.5 その他の 1 万テナント hook

| 機構 | 作用 |
|------|------|
| **Tier-based Job Priority**(`group=1` 最高 → `starter=4`) | BullMQ priority、大口顧客を優先処理 |
| **Sync/Async Dual API** | `/optimize` は非同期で jobId を返す / `/optimize/sync` は Pro+ 限定、内容 ≤5KB |
| **Concurrent Jobs Enforcement** | starter の無限積み上げによる worker 過負荷を防ぐ(`starter=1`、`group=100`) |
| **Per-Tenant API RPM** | Redis sliding-window per brand、fail-open(Redis 失効時はブロックしない) |
| **Retention TTL SSOT** | 6 テーブルを daily 05:00 cron で期限切れ row を掃除、`scoring_configs.f12_retention_config` を admin が調整可 |
| **Budget Alerts** | tier threshold(starter $1 / group $5000 月)warn@80%、critical@block_pct |
| **Scale-up Trigger** | daily 04:30 に 8 次元 metrics(tenant_count / monthly_ops / p99 / cache_hit / cost)を snapshot、spec に合わせて phase 1→2→3 を上げる |
| **Tenant Isolation Strategy** | spec/actual の 2 欄位を書き、enterprise/group 顧客の isolation gap を admin に見せる |
| **L3 S3 Cache hook**(Phase 3 予約) | `F12_S3_CACHE_BUCKET` env 設定時に動的に `@aws-sdk/client-s3` を import |

完全な SSOT は `scoring_configs.f12_*` シリーズ 14 個の key にあり、admin UI `/dashboard/admin/f12-dashboard` が 4 個の stat card + 4 個の detail section を統合する。

**相互依存関係**:9 点セットは独立していない——Concurrent Jobs Enforcement が worker 過負荷を防ぐ → Job Priority が高 tier 顧客が低 tier に押しのけられないようにする → Per-Tenant RPM が burst を防ぐ → LLM Gateway が全プラットフォームの quota 爆発を防ぐ → Cache が LLM cost を減らす → Quota / Billing が金を管理する → Budget Alerts が金の暴走を防ぐ → Scale-up Trigger がいつ phase を上げるべきかを示す。いずれか 1 つが欠けても 1 万テナント scale で事故になる。

---

## 14.5 デプロイ、クォータ、課金の 3 点

### 14.5.1 デプロイ:Wave 0 / Wave 1 の対応

V3.1 は spec Wave 0(microservice contract OpenAPI)+ Wave 1(in-process 実装 + SDK + admin)を完成させた:

- **OpenAPI 3.0 spec**:`backend/src/openapi/f12-v3-1.json` は 7 paths × 14 schemas、`/api/v1/f12/openapi.json` を公開(router auth の前にマウント)し、AI agents / サードパーティが machine-discover できるようにする
- **F12Client SDK**:`services/f12/f12Client.js` は 5 つの主入口(`diagnose / optimize / optimizeSync / getJob / health`)、`F12_SERVICE_URL` env 設定時は HTTP を通り、未設定なら in-process、**caller signature は不変**で将来のマイクロサービス分割に hook を残す
- **/api/v1/f12/health**:OAuth の前の公開 endpoint、monitoring システムが直接 ping できる

デプロイ順序(時系列):

1. **v3.21**(2026-04-23):TenantQuotaService + BillingTracker + ME custom engine の実ロジック + LLM Gateway
2. **v3.22**(2026-04-25):Plan Feature Gate + Multi-Layer Cache + Tier Priority + Sync/Async Dual API
3. **v3.23**(2026-04-27):F12Client SDK + OpenAPI + Tenant Isolation hook + L3 S3 hook + Scale-up Trigger
4. **v3.24**(2026-04-29):Concurrent Jobs + Per-Tenant RPM + Retention + Budget Alerts
5. **v3.25**(2026-04-30):Retention dedup + Redis Billing cache + Admin Dashboard
6. **v3.26**(2026-05-01):AI bot whitelist SSOT(7-layer audit fix に対応)
7. **v3.27**(2026-05-02):Phase 2-3 規模化基盤 hook(schema/db provisioner、Redis cluster、shard router、multi-region)

各 batch の完全な流れ:マイグレーション SQL → ビジネスロジック → 規格 test → admin UI → CLAUDE.md ドキュメント → PROD へ push。

### 14.5.2 クォータ:観察期 → 強制期

リリース直後は `quota_enforcement_enabled=false`(usage を記録するだけ)、顧客は「今月の使用量 / 上限」を見られるがブロックされない。分布が妥当であることを ~2 週間観察してから admin がワンクリックで `true` に切り替え、以後は超過が直接 HTTP 429 + reject reason `'quota_exceeded'` になる。

切り替え前の準備:

1. **顧客通知**:UI で「今月 X / Y を使用、翌月 Y/Y 以降は超過するとブロック、プランのアップグレードはご連絡を」を表示
2. **カスタマーサポート研修**:超過でブロックされた顧客は直接 ticket を出すので、CS は「プランアップグレード / 月末リセット / 緊急クォータ調整」の 3 つの対応に習熟する必要がある
3. **monitoring に赤ランプ追加**:admin dashboard で「過去 24h にブロックされた brand 数」を表示、> 5 で alert を発する(バグであって真の超過でない場合、素早くロールバックする必要がある)
4. **ロールバック方案**:ワンクリックで `false` に戻し 5 秒で反映(SSOT は DB にあり deploy 不要)

### 14.5.3 課金:per-call 記録、月末集計

`BillingTracker.recordCall(model, tokens, runId)` は LLM call のたびに記録する。月末に admin が `/admin/f12-billing-summary?period=YYYY-MM` を回して top 50 cost brand を見る。

ダブルライト戦略:Redis(`f12:billing:tenant:{id}:{period}` INCRBYFLOAT + 35 日 TTL)+ DB(`f12_billing_records`)。当月は `getBrandMonthlyCost` が Redis O(1) GET を優先し、過去月または Redis miss のときだけ DB SUM する。1 万テナントが月度 cost を query しても DB を打たず、sub-ms で応答する。

35 日 TTL は意図的な設計である:当月 + 前月をカバーし、顧客が 7 日に前月の請求を見たくても瞬時に照会できる。35 日後に自動失効し、常に DB を真相源とする。

monthly batch が「コスト vs 月額」の対比を回し、粗利がマイナスの brand を自動 alert する(価格設定ミス / 顧客の誤用 / バグの可能性)、admin がレビューする。

---

## 14.6 観察された制限と未解決問題

### 14.6.1 双エンジン結果の「マージ」に正解はない

Hybrid engine は AutoGEO + E-GEO を並行実行し、`improvement` が高いほうを選ぶ。しかし実際の商業シナリオでは、improvement は単一の指標ではない:

- AutoGEO の rule rewrite は構造が安定だが brand の個性が少ない
- E-GEO の template は変化に富むが、ときに顧客が自社コンテンツを認識できないほど書き換える

V3.1 は「engine improvement score が高い」を選択基準にしているが、実際には不十分である。次バージョンでは**「accepted by client」シグナル**を加えたいが、現状 admin 側の編集インターフェースはまだオンライン化していない(Phase 3)。

### 14.6.2 paper_table_ref / expected_uplift が NULL

「字面通り arxiv を port する」はエンジニアリング憲法 #1 の副作用である——論文が逐条の table mapping を示していないので、システムは**捏造を禁止する**。しかし admin UI が「なぜこの rule を選んだか」を表示するとき、データがなければ空白になる。

V2 でやりたいのは**自家 measurement** である:各 rule に対して実際の ML eval を回し(同 prompt × 3 LLM × 1000 brand、citation_rate uplift を measure)、実数で埋め戻す。時間 + Anthropic / OpenAI quota が必要:

- 25 rule × 3 LLM × 100 brand × 5 反復 = 37500 LLM call
- Claude Sonnet $0.003/1k tokens、平均 5k tokens/call = $0.015/call
- 総 cost ≈ $562
- 時間:Anthropic RPM 2000/min、フル稼働で ≈ 18 分

コストは高くないが、1000 brand の実際の baseline citation rate データが必要である。現状は 30 brand だけで、brand 数が足りてから回せる——Phase 3 で 5000+ brand を見込む頃に行う。

### 14.6.3 L3 S3 cache はまだ実デプロイされていない

Phase 3 でようやく必要(5000+ tenant 規模と推定)。現状 L1+L2 の hit rate は約 35〜45% で、70% 目標にはまだ距離がある。L3 で更に 15〜20% 上がると見込むが、S3 cost vs LLM 再計算 cost の tradeoff 分析が必要で、まだ済んでいない。

概算:L3 S3 PUT $0.005/1000 obj、GET $0.0004/1000 obj、storage $0.023/GB/月。1 万 brand × 100 ops × 5 KB / op = 5 GB、storage $0.115/月は無視できる。しかし GET ごとに 0.4 ms latency で、L1 Redis の 0.05 ms に比べ L3 は 8 倍遅い。Phase 3 では必ず L3 を cross-region(US + JP + EU)に置き、latency を平準化する必要がある。

### 14.6.4 ME custom engine の門番が厳しすぎる

`personal_profiles` テーブルの metadata が全て空 → `meta_incomplete` で reject。実際に観察すると ME プラットフォームの 17 個の会員 brand のうち 5 個が metadata 不完全(全名 / known_for 未記入)で、この 5 個は ME custom engine を通れず E-GEO `authoritative` に fallback するが、結果は企業寄りの語調で個人 IP に適さない。カスタマーサポートが顧客に metadata の補完を促す必要があるが、これは「自動化」の約束に反する。

次バージョンで検討:**LLM が brand_documents から personal_profiles の欠落欄を自動推論する**が、hallucination を防ぐ high-confidence guard が必要。例えば「Tina Chou の jobTitle は?」を LLM が彼女の著作 / メディア報道から推論し、推論結果の confidence > 0.95 のときだけ埋める。

### 14.6.5 前後依存の 1 万テナントテスト

V3.1 の全基盤は「1 万テナントに耐えうる」設計に合わせているが、実際の負荷テストは 17 個の ME brand + 21 個の GEO brand までである。Scale-up Trigger の phase_1_to_2 / phase_2_to_3 ロジックは spec の line 1183-1193 に字面通り合わせているが、**一度も実際にアップグレードしたことがない**。理論上、phase 跨ぎのアップグレードには:tenant schema 隔離の切り替え(Phase 1 row_level → Phase 2 schema)、Redis cluster sharding、DB の cross-region replication が必要で、これらは spec hook に用意してある(`schemaProvisioner` / `dbConnectionRouter` / `redisClusterAdapter` / `shardRouter`)が、**実戦を経ていない**。

今後 1 年の作業項目:

- **synthetic load test**:1000 brand × 100 ops/月を模擬する generator を書き、1 ヶ月フル稼働させて Scale-up Trigger が想定通り発火するか見る
- **Phase 2 dry-run**:staging 環境で実際に schema isolation を切り替え、query 経路、cross-schema JOIN がまだ動くかテスト
- **Phase 3 cross-region sketch**:Cloudflare Workers がどう最寄り region にルーティングするかテスト

これらはすべて production で直接はできず、staging 環境 + 追加のハードウェア予算が必要である。

### 14.6.6 クロステナント cache のプライバシー境界(新規追加)

14.4.3 で cache key が tenant_id を含まず、クロステナント共有で LLM cost を減らすと述べた。一見プライバシー懸念を招きうるが、設計時にすでに分析している:

- **content が完全一致した場合のみヒット**:cache key は `sha256(content + decision)` で、content は KB 規模、2 顧客が全く同じ content を書く確率は 0(盗用でない限り)
- **実際のヒットシナリオ**:generic FAQ template「GEO とは何か?」の答えが複数顧客に使われる——この case での共有は妥当(顧客の私有情報がない)
- **将来のリスク**:cache key の normalization(brand name token を placeholder に置換)がオンライン化すれば、クロス brand の hit rate は上がるが、normalization が情報を漏らさないことを厳格に保証する必要がある
- **法規適合**:GDPR / 個資法は「同一の公開コンテンツの LLM 結果の共有」を禁じていない、content 自体がすでに brand が自発的に公開した情報だから

V2 では audit log を加える計画である:cache hit のたびに「from brand A → to brand B」を記録し、顧客はクロステナント共有を opt-out できる(引き換えに自身の LLM cost が高くなる)。

### 14.6.7 cache key アルゴリズムの暗黙の前提(新規追加)

cache key = `sha256(content + decision.path + template + target_engine)` には暗黙の前提がある:**LLM call は deterministic(同 input → 同 output)**。実際には:

- `temperature=0` は大部分 deterministic だが、OpenAI / Anthropic のどちらも 100% は保証しない(infrastructure 層に微小な noise がある)
- 同じ prompt を 100 回回すと、~95% が identical / ~5% は字面が異なるが意味は同じ

cache hit 時は直接古い結果を返し、顧客は差に気づかない、95% deterministic はすでに非常に高い一貫性だからである。しかし将来 LLM API が「明確に non-deterministic」になれば(例えば OpenAI Sora / Gemini Veo の動画生成)、cache は versioning と invalidation を加える必要がある。

実装上は予約してある:`f12_result_cache` に `cache_version` 欄があり、Phase 3 で LLM 世代交代時にワンクリックで version を bump して全プラットフォームの cache を失効できる。

---

## 14.7 エンジニアリングの教訓

V1→V3.1 の進化(7 ヶ月、5 回の PROD 障害)から、5 条の教訓を整理した:

### 14.7.1 arxiv の字面 port は自家発明より安全

当初は自分で GEO 規則を発明したかった(internal な観察に基づく)が、すぐに分かった:

- 対比実験が欠如(自家 30 brand では規則を derived できない)
- 自家顧客のスタイルに overfit しやすい
- 学術 reference がないと顧客が買わない(「なぜこう書くのか?」「我々が良いと思うから」←弱い)

「字面通り arxiv ソースコードを port」に切り替えてから:

- ✅ 25 + 15 条の rule/template が即座に使える
- ✅ 顧客は「arXiv:2510.11438」を見て即座に信服する
- ✅ DB レベルの placeholder guard が永久に捏造を防ぐ

しかし代償は paper_table_ref / expected_uplift が NULL、UI が空白を表示すること。空白でも捏造しない——これが憲法 #1 の代償である。

### 14.7.2 全プラットフォーム SSOT は code へのハードコードより重要

V1 の初期はすべての閾値をコードにハードコードしていた(`const LOW_SCORE_THRESHOLD = 70`)、変更のたびに deploy + 再起動。顧客が「なぜこのページは 65 点なのに最適化されないのか」と問い、エンジニアがコードを grep すると `hard-coded 70` だと分かり、65 に変えて deploy。全プロセスで 30 分。

SSOT(`scoring_configs.f12_score_thresholds.low_score`)に切り替えてから:

- admin UI で数字を 1 つ変えると 5 秒で反映
- 異なる brand_type で異なる閾値にできる(GEO 70、ME 75)
- 異常時にワンクリックでロールバック(SQL UPDATE で前の値へ)

代償は indirection 層が 1 つ増え、新人はコードを見るのに 1 回多く grep する必要があること。しかし deploy 速度に比べれば、それだけの価値がある。

### 14.7.3 DB レベルの強制は application レベルより信頼できる

placeholder guard は当初 application レベルでやろうとした(INSERT のたびに application がチェック):

- ❌ admin UI が直接 SQL で編集するときは迂回する(psql で PROD を操作することが稀にある)
- ❌ generator が raw SQL `INSERT INTO axp_pages` を書くと service layer を迂回する
- ❌ pipeline cron の古いコードは service を通るが service がチェックしない

DB trigger に切り替えてから:

- ✅ いかなる経路の書き込みも阻止される(psql の直接 INSERT を含む)
- ✅ application code は安心できる(trigger が最後の砦)
- ✅ 規格がより安定(trigger 規則は SQL だけで誤変更しにくい)

代償:trigger の debug は困難(error message が不明瞭)だが、1 万テナント scale ではこの代償に見合う。

### 14.7.4 観察期は SaaS デプロイの標準装備

quota_enforcement_enabled / billing_recording_enabled などの flag はすべてデフォルトで「観察期」(記録するだけでブロックしない)にし、分布が妥当であることを確認してから「強制期」に切り替える。観察期 ≥ 2 週間、その間:

- 実際の usage 分布を見る
- outlier を見つける(バグの可能性も真の使用量の可能性もある)
- 顧客に「翌月からブロックする」と通知する
- カスタマーサポートを研修する

V1 の初期はこの機構がなく、直接 enforcement をリリースした。結果:

- 第 1 週にすでに 3 顧客が超過でブロックされたが、実は admin がクォータを誤設定していた(100 を 10 と書いた)
- CS が研修されておらず、顧客の ticket に答えられなかった
- 最後にワンクリックで enforcement をロールバックしたが、顧客体験はすでに壊れていた

観察期を加えてからは同種の事象は二度と起きていない。

### 14.7.5 hook 全カバレッジ + safety net はどちらも欠かせない

axpPageWriter F12 hook は 100% カバレッジでなければならないが、100% は保証しにくい(13 個の入口、新 generator がいつでも追加される):

- **hook 100%**:第一の防衛線、即時に F12 analyze をトリガー
- **weekly backfill cron**:第二の防衛線、日曜 02:00 に全プラットフォームを再分析し漏れを補う
- **admin manual trigger**:第三の防衛線、admin が異常を見たら単一 brand を手動で再分析トリガー

三層のどれも欠かせない——hook だけではカバレッジ漏れ → 永久に未分析;backfill だけでは間に合わない(最悪 7 日後の反応);manual だけでは規模化しない。

V1 の初期は hook だけを作っていた。あるとき fact-check generator が hook を漏らし、4 個の brand の fact-check ページのスコアが異常に低いまま 3 ヶ月発見されなかった。backfill を加えてからは同種の事象は最悪でも 7 日で必ず反応する。

---

## 本章のまとめ {.unnumbered}

- F12 は Ch 3 スコアリングが残した「次の記事をどう書くか」問題を補い、コンテンツを入力して三層スコア + 最適化版を出力する
- V1 は規則式の三層分析 + LLM Optimizer(`min_similarity ≥ 0.90` で脱線防止)、5 個の cron を連結して全プラットフォームをカバー
- V3.1 は AutoGEO + E-GEO + Hybrid の三エンジンを導入、字面通り arxiv ソースコードを port、DB trigger で paper_table_ref の捏造を防ぐ
- 1 万テナント基盤 9 点セット:クォータ / 課金 / Plan Gate / Multi-Layer Cache / LLM Gateway / Job Priority / Sync-Async / Retention / Scale Trigger
- 制限:engine のマージに最適解なし / paper の実 uplift は自家 measure 待ち / L3 cache 未デプロイ / ME metadata の門番が厳しすぎ / phase アップグレード未実戦 / クロステナント cache のプライバシー境界 / cache key が LLM deterministic を仮定
- 5 条のエンジニアリング教訓:字面 port arxiv > 自家発明 / SSOT > ハードコード / DB レベル > application レベル / 観察期は標準装備 / hook + safety net はどちらも欠かせない

## 参考資料 {.unnumbered}

- [第 3 章 — 七次元スコアリングアルゴリズム](./ch03-scoring-algorithm.md)
- [第 6 章 — AXP シャドウドキュメント交付](./ch06-axp-shadow-doc.md)
- [第 12 章 — 制限、未解決問題と未来の作業](./ch12-limitations.md)
- AutoGEO paper: arXiv:2510.11438 — github.com/cxcscmu/AutoGEO
- E-GEO paper: arXiv:2511.20867 — github.com/psbagga17/E-GEO
- F12 v3.1 spec: `F12_TLSO_spec_v3_1.md`(repo root)

## 改訂履歴 {.unnumbered}

| 日付 | バージョン | 説明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — V1 + V3.1 の完全なアーキテクチャ進化を扱う |
| 2026-05-03 | v1.1.1 | 章を ~7200 字に拡充 — 14.1.2 初期の手動調整の失敗モード、14.2.4 双方向可逆の設計、14.2.5 hook 全カバレッジの難しさ、14.2.6 cron 時系列、14.3.5 placeholder guard の物語、14.4 相互依存関係、14.5 デプロイ順序、14.6.6/7 cache のプライバシーと前提、14.7 エンジニアリング教訓 5 条を追加 |

---

**ナビゲーション**：[← 第 13 章: マルチモーダル GEO](./ch13-multimodal-geo.md) · [📖 目次](../README.md) · [第 15 章: rag-backend-v2 の堅牢化 →](./ch15-rag-backend-v2-hardening.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "第 14 章 — F12 三層構造オプティマイザ:ルールベース V1 から双エンジン v3.1 へ",
  "description": "F12(TLSO)のアーキテクチャ進化:V1 の規則式 + LLM Optimizer、V3.1 の AutoGEO + E-GEO + Hybrid 三エンジン、さらに 9 点セットの 1 万テナント基盤(クォータ/課金/Plan Gate/Multi-Layer Cache/LLM Gateway/Job Priority/Sync-Async/Retention/Scale Trigger)。5 条のエンジニアリング教訓を含む。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "ja",
  "isPartOf": {
    "@type": "Book",
    "name": "百原 GEO Platform 技術白書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "F12, TLSO, AutoGEO, E-GEO, Hybrid Engine, Plan Feature Gate, Multi-Layer Cache, LLM Gateway, Tier Priority, Scale Trigger, エンジニアリング教訓"
}
</script>
