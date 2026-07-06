---
title: "第 15 章 — rag-backend-v2 の堅牢化:6 層の LLM ハルシネーション失敗モードに対する防御設計"
description: "中央 RAG マイクロサービス(rag-backend-v2)のエンジニアリング堅牢化史:Wiki Cascade 4 層 cascade、6 種の wikiCompiler ハルシネーション失敗モードの修法(cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens)から Queue retention 自動 cleanup まで、LLM-driven システムの robustness 設計実践を完全に提示する。"
chapter: 15
part: 5
word_count: 7100
lang: ja
authors:
  - name: Vincent Lin
    affiliation: Baiyuan Technology
    email: services@baiyuan.io
license: CC-BY-NC-4.0
keywords:
  - rag-backend-v2
  - LLM Wiki Compiler
  - Wiki Cascade
  - LLM Hallucination Hardening
  - Docker DNS
  - LLM Timeout
  - AbortSignal
  - Multilingual Prompts
  - BullMQ Retention
  - PostgreSQL pgvector
last_updated: 2026-05-03
canonical: https://baiyuan.io/whitepaper/ja/ch15-rag-backend-v2-hardening
---

# 第 15 章 — rag-backend-v2 の堅牢化:6 層の LLM ハルシネーション失敗モードに対する防御設計

> LLM を信頼できない世界では、エンジニアリングの職責は LLM を完璧に強いることではなく、あらゆる失敗の形態に出口を用意しておくことである。

## 目次 {.unnumbered}

- [15.1 なぜ RAG は独立マイクロサービス + 自家 Wiki が必要か](#151-なぜ-rag-は独立マイクロサービス--自家-wiki-が必要か)
- [15.2 Wiki Cascade:4 層カスケード削除](#152-wiki-cascade4-層カスケード削除)
- [15.3 6 種の wikiCompiler ハルシネーション失敗 + 対応 patch](#153-6-種の-wikicompiler-ハルシネーション失敗--対応-patch)
- [15.4 Worker robustness:silent hang から自動復旧へ](#154-worker-robustnesssilent-hang-から自動復旧へ)
- [15.5 1 万テナントのデプロイパターン](#155-1-万テナントのデプロイパターン)
- [15.6 観察と未解決問題](#156-観察と未解決問題)
- [15.7 エンジニアリングの教訓](#157-エンジニアリングの教訓)

---

## 15.1 なぜ RAG は独立マイクロサービス + 自家 Wiki が必要か

### 15.1.1 Karpathy の LLM Wiki 概念の紹介

Andrej Karpathy は 2024 年にある概念を公開共有した:従来の RAG は chunk を直接 LLM に投入するには「断片的」すぎて構造がない;むしろ先に LLM で全ソース文書を**wiki にコンパイル**し、各 wiki は明確な slug / page_type / TLDR / tags を持ち、query 時にはまず wiki を見て(速い、安い、説明可能)、wiki に答えがなければ chunk-level RAG に fallback する。

我々はこの概念を cs-frontend(企業 RAG 知識ベース製品)+ GEO Platform 共用の rag-backend-v2 マイクロサービスに**実際に実装**し、宣伝概念として使うだけでなく、実際に本番で走らせている:

- 顧客がソース文書(PDF / Markdown / URL crawl)をアップロード → `tenant_documents` テーブルへ
- LLM compiler がすべての ready-to-compile な doc を wiki pages にコンパイル → `wiki_pages` テーブルへ
- ユーザーの query → wiki_query_route LLM がまず「この query はどの page_type に属するか」を決定 → 対応する wiki を直接返す
- wiki がミス → L2 chunk-level RAG(pgvector + BM25 + RRF)に fallback

この L1+L2 hybrid が生む効益:同じ query「百原 GEO はどうやってブランドの AI 引用率を上げるのか」:

- L2 only:5 個の chunks を検索 → LLM rewrite → 12 秒、LLM cost ≈ $0.008
- L1+L2:wiki が直接返す → 1.5 秒、LLM cost ≈ $0.0012(85% 削減)

しかし L1 の成立には 1 つの前提がある:**Wiki のコンパイルが安定していなければならない**。実際には、4 月初旬から 5 月初旬の 1 ヶ月で、6 種の LLM ハルシネーション形態の落とし穴を踏み、そのどれもが wiki コンパイルを部分的または全面的に失敗させた。本章はこの 6 層の堅牢化を記録する。

### 15.1.2 なぜ RAG は独立マイクロサービスなのか

GEO 主 backend と `rag-backend-v2` は同一の PostgreSQL instance 内にあるが、**2 つの DB は完全に分離している**:

| | `geo_db` | `cs_rag_db` |
|---|---|---|
| サービス | GEO Platform 主 backend | rag-backend-v2 マイクロサービス |
| 用途 | brands / users / axp_pages / ground_truths | tenant_documents / chunks / wiki_pages |
| 役割 | ビジネスロジック + マルチテナントデータ | ベクトル検索 + LLM Wiki コンパイル |
| クロス DB query | `csRagQuery(text, params)` 独立 pool | — |

分割の理由は 3 つ:

1. **データ規模が異なる**:RAG の `chunks` テーブル(ベクトル embedding)は顧客量とともに爆発し、単一テーブル 100M row も珍しくなく、ビジネステーブルと混ぜると主 DB の性能を損なう
2. **更新ペースが異なる**:RAG 文書は「アップロードしたら動かさない」、主 DB のビジネスデータは「秒単位で更新」
3. **降格境界が明確**:RAG がダウンしても主プラットフォームはまだ動く(AI 回答品質が下がるだけ);主 DB がダウンすると全部壊れる — RAG に独立 surface を与えることで fault isolation が可能になる

### 15.1.3 クロス DB query のコスト

`backend/src/config/database.js` は 2 つの独立 PG pool を開く:

```javascript
const geoPool = new Pool({ connectionString: DATABASE_URL });
const csRagPool = new Pool({
  connectionString: DATABASE_URL.replace(/\/geo_db$/, '/cs_rag_db'),
});

async function csRagQuery(text, params) {
  try {
    return await csRagPool.query(text, params);
  } catch (err) {
    console.error('[csRagQuery] failed:', err.message);
    return { rows: [] };  // fail-soft 不擋主流程
  }
}
```

UAT には cs_rag_db がない(`DATABASE_URL` は single-DB instance に対応)→ csRagQuery は直接 fail するが、try/catch が受け止めて空配列を返し、主流程はブロックされない。PROD は両 DB がそろっており、csRagQuery は正常に動く。

代償:GEO 主 backend が「N 個の Wiki ページ」を表示するために `wiki_link_count` をクロス DB で取るには csRagPool query を回さねばならず、同 DB JOIN に比べ +30ms の遅延。しかし 2 つの DB を統合する運用コストに比べれば、この 30ms は許容できる。

### 15.1.4 V1 と V2 の進化

V1 の RAG は chunks(L2 ベクトル検索)+ pgvector + BM25 + RRF だけだった。L1 layer は 2026 年 4 月に加えた「Karpathy LLM Wiki」概念の実装である。V1→V2 の決定要因:

- L2-only の平均応答は 12 秒で、顧客が遅いと嫌がる(特に cs-frontend の顧客対応シナリオ)
- LLM cost / 月 / brand は約 $80 で、粗利がマイナス
- 顧客が「なぜこの回答か」と問うが、L2 chunks は断片的すぎてソースを説明できない

V2 で wiki を加えた後、L1 hit のときは 1.5 秒で返し + 「ソース」として wiki page_id を指せるようになり、顧客満足度が上がった。しかし L1 は安定していなければならず、いかなるコンパイル失敗も顧客に「関連データなし」という虚像を見せてしまう。

---

## 15.2 Wiki Cascade:4 層カスケード削除

### 15.2.1 なぜ 4 層 cascade が必要か

顧客がフロントエンドで「文書を削除」を押すと、連鎖的に掃除する必要がある:

```text
brand_documents (geo_db)
        │ via central_doc_id UUID
        ▼
tenant_documents (cs_rag_db)        ← LLM Wiki 編譯來源
        │ via wiki_page_sources junction
        ▼
wiki_page_sources                   ← 多對多關聯表
        │ via SSOT 同步 trigger
        ▼
wiki_pages.source_doc_ids           ← 被引用的 wiki page
```

4 層の全自動 cascade は以下を通じて行われる:

1. `brand_documents.central_doc_id` 欄(migration 170)— アップロード時に中央が返す doc id をこの欄に書き込む
2. DELETE handler:`ragDeleteDocument(tenantId, central_doc_id)` → 中央の `trg_source_soft_delete` cascade trigger が自動的に junction を掃除
3. junction が最後の source を失った wiki_pages は自動的に soft delete
4. `migration 201:sync_wiki_page_sources trigger` — `wiki_pages.source_doc_ids` 配列と `wiki_page_sources` junction の同期を強制する(SSOT)

設計の要:**junction が SSOT、array は冗長な derivation** である。もとは両者を別々に維持しており、よく desync していた(36 個の stale junction entries を観察)、migration 201 の後 trigger が両者の一致を強制し、顧客側の「N 個の Wiki ページ」UI 表示が常に正しくなった。

### 15.2.2 SSOT trigger の SQL 実装

migration 201 の trigger は両者の同期を強制する:

```sql
CREATE OR REPLACE FUNCTION sync_wiki_page_sources()
RETURNS TRIGGER AS $$
BEGIN
  -- INSERT/UPDATE wiki_pages.source_doc_ids → 同步 junction
  IF TG_TABLE_NAME = 'wiki_pages' AND (TG_OP = 'INSERT' OR TG_OP = 'UPDATE') THEN
    -- 刪除 junction 中不在 source_doc_ids 的 row
    DELETE FROM wiki_page_sources
     WHERE wiki_page_id = NEW.id
       AND NOT (source_doc_id = ANY(NEW.source_doc_ids));
    -- 插入 source_doc_ids 中不在 junction 的 row
    INSERT INTO wiki_page_sources(wiki_page_id, source_doc_id)
    SELECT NEW.id, doc_id
      FROM unnest(NEW.source_doc_ids) AS doc_id
     WHERE NOT EXISTS (
       SELECT 1 FROM wiki_page_sources
        WHERE wiki_page_id = NEW.id AND source_doc_id = doc_id
     );
  END IF;

  -- DELETE wiki_page_sources 後,若 wiki_page 失去最後一個 source → 軟刪
  IF TG_TABLE_NAME = 'wiki_page_sources' AND TG_OP = 'DELETE' THEN
    UPDATE wiki_pages
       SET deleted_at = NOW(), source_doc_ids = '{}'
     WHERE id = OLD.wiki_page_id
       AND NOT EXISTS (
         SELECT 1 FROM wiki_page_sources WHERE wiki_page_id = OLD.wiki_page_id
       );
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;
```

この trigger をデプロイする前は、掃除ロジックが 7 ヶ所に散らばっていた(application code 4 ヶ所 + migration script 3 ヶ所)、よく desync した。trigger オンライン後は 1 ヶ所を変えれば 7 ヶ所を触る必要がなくなり、1 万テナント scale に必要である。

### 15.2.3 36 個の stale junction entry のデバッグ

trigger オンライン時に一度きりの reconciliation を行った:

```sql
-- 偵測 stale junction(junction 指向不存在的 wiki_pages 或 tenant_documents)
SELECT wps.*
  FROM wiki_page_sources wps
  LEFT JOIN wiki_pages wp ON wp.id = wps.wiki_page_id
  LEFT JOIN tenant_documents td ON td.id = wps.source_doc_id
 WHERE wp.id IS NULL OR td.id IS NULL;
-- → 36 row
```

一度の DELETE で掃除し、trigger 以降は新規発生しないことを保証する。この 36 個の entry はどこから来たのか?追跡すると:

- 12 個は application code の初期版が junction を INSERT したが array の更新を忘れたもの
- 14 個は cron が wiki を再コンパイルするときの race condition(古い wiki は soft delete だが junction が掃除されない)
- 10 個は migration script が歴史の中で backfill を試みて失敗した残骸

trigger 以降は 3 種の case すべてが行き止まりとなり、以下を保証する:**junction は常に array の内容を reflect する**。

### 15.2.4 UAT vs PROD の挙動差異

UAT には中央 RAG がない(`CENTRAL_RAG_URL` unset)→ Phase 1 path が自動 skip し、ローカルの brand_documents は純粋に pgvector adapter を通り、**主流程に影響しない**(try/catch がブロックしない)。PROD には中央 RAG があり → 完全な 4 層 cascade。これは「降格境界が明確」という設計に合致する。

---

## 15.3 6 種の wikiCompiler ハルシネーション失敗 + 対応 patch

各失敗はすべて実際の PROD 観察から来ており、以下 `Patch N` の番号順で記録する。**この 6 個の patch は 2026 年 4 月末 5 月初旬の 1 週間で連鎖的に落とし穴を踏んで修したものである**、それぞれが血みどろの障害の瞬間に対応する。

### 15.3.1 Patch 1:150K char cutoff bug

**失敗形態**:ある KB の 84 個の docs のうち 33 個しか wiki にコンパイルされず、**51 個が永久 orphan** になった。

**発見過程**:顧客が「84 個の FAQ 文書をアップロードしたのに、wiki のページ数は 33 ページしか表示されない」と反応。最初は LLM コンパイル失敗を疑ったが log は全部緑;次に doc が not ready を疑ったが DB チェックで全部 `status='ready'`。最後に `wikiCompiler.js` を grep して cutoff ロジックを発見した。

**根本原因**:`compileWiki()` function の冒頭に cutoff ロジックがあり、`compileIncremental` から借りてきたが**ロジックが適用されない**:

```js
// Buggy 原版
let totalChars = 0;
const selectedDocs = [];
for (const doc of docs) {
  if (totalChars + (doc.content?.length || 0) > 150000) break;  // 砍超過 150K 後的全部 doc
  selectedDocs.push(doc);
  totalChars += doc.content?.length || 0;
}
```

問題は、main `compileWiki` は doc ごとに**独立した LLM call** を回すので、aggregate token 制限の理由がないことである。しかし `compileIncremental` は複数 doc を 1 つの prompt に詰め込むので prompt token 制限がある。コードを借りたとき使用状況をよく見なかった。

さらに `ORDER BY created_at ASC` が**初期 doc を先に** selectedDocs に入れ、新 doc は常に切られる。1 万テナント scale:84 docs × 4.5K avg = 378K → 33 docs しかコンパイルされない → 51 docs が永久 orphan。

**修法**:cutoff を削除し、すべての ready wiki source を取り込む(per-doc LLM call は sequential で burst しない):

```js
// Patched
const selectedDocs = docs;
const totalChars = docs.reduce((s, d) => s + (d.content?.length || 0), 0);
```

### 15.3.2 Patch 2:parseCompiledArticles partial UUID

**失敗形態**:KB 全体の compile が中断し、後続 docs が全くコンパイルされない。PROD log:

```text
[WIKI] Compile error: invalid input syntax for type uuid: "b7bced27"
```

**発見過程**:Patch 1 deploy 後に compile を回し直すと、13 個目の doc で全体が abort。log は「invalid uuid」を示すが、完全な UUID は `b7bced27-3e4f-4abc-...` で、先頭 8 文字だけ切り取られたように見える。

**根本原因**:LLM(DeepSeek)が `SOURCE_IDS:` header 行で稀に切り取られた UUID(完全な 36 文字でなく 8 文字 prefix)を出力する。元のコードは検証せず、そのまま PG `INSERT INTO wiki_pages(source_doc_ids UUID[])` に渡す → PG RAISE → catch できない → batch 全体の compile transaction が rollback。

要:**単一 LLM の 1 回の小さな誤りが、batch 全体を吹き飛ばす**。1 万テナント scale では、LLM が偶発的に 1 回 hallucinate するだけで KB 全体を回し直すことになる。許容できない。

**修法**:LLM 出力を正規表現で検証する:

```js
const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
const sourceDocIds = sourceIdsRaw
  ? sourceIdsRaw.split(',').map(s => s.trim()).filter(s => UUID_RE.test(s))
  : [];
```

filter 後に空配列なら、後段の「保底」ロジックが依然として `doc.id` の実 UUID を prepend し、LLM が garbage を出しても帰属の正しさに影響しない。

### 15.3.3 Patch 3:fallbackWrapArticle — LLM marker ハルシネーションの防御

**失敗形態**:LLM が稀に prompt で定めた `---ARTICLE---/---CONTENT---` marker を無視して純テキストを吐く。`parseCompiledArticles` が marker を見つけられない → 0 article を返す → その doc が永久 orphan。PROD で `02_醫美業_FAQ_與_Query集.md` のコンパイルで LLM が marker なしの 5670 字を出力するのを観察。

**発見過程**:Patch 2 deploy 後に回し直すと、84 docs すべてが compile 完了したが、`wiki_pages` は 51 個しか追加されなかった。対応する doc id を grep し、一部の doc に対応する wiki ページがないことを確認。LLM 出力 log をチェックすると、一部 doc の LLM が純テキストを直接吐き、`---ARTICLE---` すらなかった。

**根本原因**:LLM の出力 schema は信頼できず、特定の token / marker / JSON shape に依存する parser はすべて偶発的に失敗する。Anthropic / OpenAI / Gemini はどれも稀に「指示を忘れる」、DeepSeek は割合が高め(約 5〜8%)。

**修法**:fallback wrap helper を書き、marker が欠けても内容が ≥50 字あれば保底で single article に包む:

```js
function fallbackWrapArticle(text, doc) {
  const trimmed = (text || '').trim();
  if (trimmed.length < 50) return null;
  return {
    title: doc.title.replace(/\.md$/i, '').slice(0, 100),
    slug: deriveSlug(doc.title) || `fallback-${doc.id.slice(0, 8)}`,
    pageType: heuristicPageType(doc.title),  // faq/product/policy/concept
    confidence: 'low',                        // 標 low 提示 admin 人工檢視
    tags: ['wiki_fallback'],                  // admin 用此 tag 過濾
    content: trimmed,
    sourceDocIds: [doc.id],                   // 與原 doc 雙向綁定
    // ...
  };
}
```

設計の要:**低信頼で包む(tags=wiki_fallback)+ admin 警告のほうがよく、顧客がアップロードしたソース文書を orphan にしてはならない**。`compileWiki` の per-doc loop と `compileIncremental` の batch hook の両経路で適用する。

admin UI は `WHERE 'wiki_fallback' = ANY(tags)` で人手レビュー用にフィルタする。実測では 84 docs KB のうち 14 個が fallback bucket に入り、人手で内容を見るとほとんど OK でただ書式が整っていないだけで、signal はまだ十分だった。

### 15.3.4 Patch 4:Docker DNS の内外双 fallback(三版進化)

**失敗形態**:rag コンテナが `postgres` 内部 hostname を解決できない → `pg.Pool.connect("postgres")` SERVFAIL **silent hang**(throw しない)→ wikiCompiler の最初の `await q(SELECT...)` が永遠に connection を待つ → wiki compile 全体が表面上は正常だが 0 進度。

**発見過程**:Patch 3 deploy 後 PROD で wiki コンパイルが「止まっている」のを観察 — admin UI に `progress: 0/84`、コンテナ CPU 0.01%、DB 0 active query、error log は一切なし。30 分進度なし。SSH でコンテナに入り `nslookup postgres` → SERVFAIL。

**根本原因**:コンテナの `/etc/resolv.conf` が host の `systemd-resolved`(`127.0.0.53`)を指し、Docker 内部 DNS(`127.0.0.11`)ではないという、Lightsail Ubuntu daemon の DNS 設定のエッジケース。**SERVFAIL は NXDOMAIN ではない**、pg client はこれを「一時的失敗、リトライを続ける」と解釈し「ドメインが存在しない、あきらめる」とはせず、無限に待つ。

**修法**(三版進化、前 2 版はどちらも失敗):

| バージョン | 設定 | 結果 |
|---|---|---|
| 初版 | `extra_hosts: ["postgres:172.18.0.10"]` ハードコード IP | 失敗:postgres IP が docker recreate 時に shift する |
| 第二版 | `dns: ["127.0.0.11"]` only | 惨事:`www.googleapis.com` を解決できない → Google OAuth が全プラットフォームで 8 時間壊れる |
| 最終版 | `dns: ["127.0.0.11", "1.1.1.1", "8.8.8.8"]` | 通過:内外の hostname が両方通る |

第二版の惨事は特にひどかった——backend / worker / frontend / rag をすべて `dns: [127.0.0.11]` に変えた後、コンテナ内 Node.js が `www.googleapis.com`(Google OAuth 用)を解決できず、Google ログインが `getaddrinfo EAI_AGAIN` を返し、顧客がまるまる 8 時間ログインできなくなった。緊急ロールバック後に根本原因を追った:Docker embedded DNS(127.0.0.11)は内部 hostname(`postgres` / `redis` / `backend`)しか扱わず、外部 hostname は host の resolv.conf に forward するが、Lightsail Ubuntu の systemd-resolved が誤設定されていて forward が失敗した。

最終版は `1.1.1.1`(Cloudflare)+ `8.8.8.8`(Google)を外部 fallback として加え、内部 hostname は 127.0.0.11 を通り、外部 hostname は fallback を通り、両方が通るようにした。

教訓:**単に Docker embedded DNS を設定するだけでは足りず、外部 fallback を残さねばならない**。`docker-compose.prod.yml` は backend / worker / frontend / rag すべてこう書く必要がある。新コンテナを追加するときは必ず**「dns: は内部 + 外部 fallback を含まねばならない」**をプラットフォーム憲法に書き込んだ。

### 15.3.5 Patch 5:LLM call の 60s timeout

**失敗形態**:wikiCompiler progress が「`AI 編譯 (13/84):官網-FAQ`」で 10 分以上止まる、CPU 0.01%、DB 0 active query。`aiCall` が永遠に返らない DeepSeek の stream を待っている。

**発見過程**:Patch 4 deploy 後 DNS は通ったが、顧客がある回のコンパイルが「13 個目で止まる」と反応。SSH でコンテナに入り `top` で node process が 0% CPU、`netstat` で DeepSeek API への `ESTABLISHED` 接続が 1 つあるが 0 byte traffic が 5 分続いているのを見た。

**根本原因**:OpenAI SDK のデフォルトは `timeout: 600_000ms`(10 分)+ `maxRetries: 2` で、理論上 20 分止まる;Gemini SDK(`@google/generative-ai`)はネイティブの timeout / abort API を提供しない。DeepSeek は稀に `ECONNRESET` / 半開接続 / 低速応答があり、デフォルト timeout では持ちこたえられない。

さらに悪いのは、**streaming response の中断はエラーを投げない**ことである——SDK は「stream はまだ終わっていない、待ち続ける」と考え、永遠に `[DONE]` event を受け取れない。

**修法**(三層 timeout):

```js
// 1) OpenAI client constructor 60s + maxRetries=1
new OpenAI({ apiKey, timeout: 60_000, maxRetries: 1 });

// 2) per-request abort signal(防 streaming 卡 chunk)
client.chat.completions.create(params, { signal: AbortSignal.timeout(60_000) });

// 3) Gemini Promise.race + 60s timeout(SDK 無 abort)
function withTimeout(promise, ms, label) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error(`${label} timeout ${ms}ms`)), ms)
    ),
  ]);
}
withTimeout(genModel.generateContent(...), 60_000, `gemini/${model}`);
```

修正後:単一 doc の失敗は最大 60s で止まり、wikiCompiler の per-doc try/catch が受け止め → Patch 3 の fallback に行くか次の doc をスキップ、**batch 全体 84 docs が単一 LLM のハングに引きずられなくなる**。1 万テナント scale で安全。

この patch は後にプラットフォーム憲法**「LLM call は必ず ≤60s timeout を持つ」**となった——rag-backend-v2 + geo-saas backend + frontend のすべての LLM call site に適用する。この鉄則に違反した結果:単一 LLM の半開接続だけで batch 全体のワークフローを引きずり、1 万 brand が同時に失敗しうる。

### 15.3.6 Patch 6:wiki_query_route maxTokens 200 → 2000

**失敗形態**:Wiki L1 が全プラットフォームでミスし、大半の query が L2 に fallback し、「関連データなし」を返す。

**発見過程**:2026-04-30 に PROD primary LLM を `deepseek-v3-direct` から `deepseek-v4-flash` に切り替えた(reasoning model のほうが賢く、理論上 wiki query のルーティングがより正確)が、翌日顧客が「何を聞いても答えられない」と反応。wiki query log を grep すると全部空。

**根本原因**:reasoning model は `message.content` の他に `reasoning_tokens` を追加で食う(token budget の 70%+ を占めうる)。`wikiCompiler.js` の 2 ヶ所の `wiki_query_route` call が `maxTokens: 200` を使っており、deepseek-v3 には十分だったが、V4 reasoning は**200 token をすべて reasoning に食い、content が常に空文字列** → JSON parse fail → wiki query ルーティング失敗 → fallback L2 も「ヒットなし」を返す。

最も奇妙なのは:**エラーがない**こと、LLM API が 200 OK で `{ content: "", reasoning_tokens: 198 }` を返し、content が空なだけ。元の try/catch では捕まえられない。

**修法**:2 ヶ所の `maxTokens` を 200 → 2000。実測で 2000 token は reasoning + content の両方に十分で、wiki query routing が正常に戻った。

**1 万テナントへの影響**:wiki を有効にし + V4-style reasoning model(DeepSeek V4 / OpenAI o1-o4 / Anthropic の thinking 含む)を使うテナントはすべて、Wiki L1 が全失効し、fallback L2 はかろうじて答えられるがコストが高く品質が悪い。この教訓はプラットフォーム憲法に広げられた:**reasoning model に切り替えたら必ず全 maxTokens 設定をチェックする**、いかなる maxTokens < 1500 も reasoning model 上で content が空でないことを検証すべきである。

### 15.3.7 6 patch 全部の全体効果

完全な 6 patch をデプロイした後の PROD 観察:

| 指標 | Before | After |
|---|---|---|
| プラットフォーム wiki orphan rate | 45%(156/348) | ≤25%(見込み) |
| 5ce78a51 KB(84 docs)compile 完了率 | 39%(33/84) | 100%(84/84) |
| 単一 doc LLM hang が batch を止める確率 | ~20% | <1% |
| Wiki L1 hit rate | 12%(reasoning model 後)| 65% |

orphan rate が 45% から ≤25% に下がったが**まだ 0% に達していない**理由:Patch 3 の fallback は「保底の解決」だが「完璧なコンパイル」ではなく、fallback bucket に入る doc は依然として `confidence='low'` とマークされ、一部の admin の計算基準では partial orphan とみなされる。orphan を完全に消すには LLM が永遠に marker を漏らさない(不可能)か、全 doc が構造化 schema を使う(JSON output mode、一部の LLM が対応)必要があり、これは V2 の作業である。

### 15.3.8 Patch 順序の因果連鎖

振り返ると、6 個の patch の発見順序は非常に巧妙だった——前の patch が直るまで、次の patch のバグは見えなかった:

```text
Patch 1 (cutoff)        → 修了才看到所有 doc 都進編譯
        ↓
Patch 2 (partial UUID)  → 修了才看到後續 doc 都編完
        ↓
Patch 3 (no marker)     → 修了才看到 wiki page 真實生成
        ↓
Patch 4 (DNS)           → 修了才看到生成過程不卡死
        ↓
Patch 5 (LLM timeout)   → 修了才看到單 doc 失敗不拖整批
        ↓
Patch 6 (maxTokens)     → 修了才看到 wiki query 真實命中
```

各 patch は「下流が正常に見えて実は正常でない」層を解く、これは LLM-driven システムの debug の典型的な形態である:**エラーが多層にわたって silent swallow され、玉ねぎを一枚一枚剥くように見えてくる**。

---

## 15.4 Worker robustness:silent hang から自動復旧へ

WikiCompiler の失敗は氷山の一角にすぎない。BullMQ worker 体系全体も同種の罠を踏んだ。

### 15.4.1 Defensive guard pattern

各 cron worker はビジネスロジックの前に必ず entity の存在をチェックする:

```js
async (job) => {
  const { brandId } = job.data;

  // 預檢 — brand 被刪後 cron 還在跑會 throw → BullMQ 累積 failed jobs → admin UI 永遠紅
  const { rows: [b] } = await query('SELECT id FROM brands WHERE id = $1', [brandId]);
  if (!b) {
    console.warn(`[Worker] Brand ${brandId} not found — removing orphan scheduler`);
    const q = new Queue(QUEUE_NAME, { connection });
    await q.removeJobScheduler(`monthly-${brandId}`);  // 自動清孤兒 scheduler
    await q.close();
    return { skipped: true, reason: 'brand_not_found', brandId };
  }

  // ... 業務邏輯
}
```

要点:**チェックが通らない → skipped を return し throw しない**。throw は failed とみなされ、return は completed とみなされ、UI は常に緑。同時についでに `removeJobScheduler` で orphan scheduler も掃除する。

PROD で観察した適用範囲:`monthly.worker.js`(brand が削除された)、`abTesting.worker.js`(experiment が削除された → FK violation)、`closedLoop sentinel`(テーブルが `brand_prompts → multilingual_prompts` にリネームされた)。

### 15.4.2 Queue retention worker

`backend/src/workers/queueRetention.worker.js` は毎日 02:30 に走る:

```js
// 24 個 known queue,daily cleanup
for (const name of KNOWN_QUEUES) {
  const q = new Queue(name, { connection });
  await q.clean(7 * 24 * 3600 * 1000, 500, 'failed');     // 7 天 failed
  await q.clean(30 * 24 * 3600 * 1000, 500, 'completed'); // 30 天 completed
  await q.close();
}
```

掃除しない結果:per-brand cron が毎日大量の jobs を生成し、失敗したものを掃除しないと Redis OOM まで積み上がり、かつ admin UI(`/admin/scan-frequencies`)が永遠に赤ランプで累積し新しい状態が見えなくなる。

実測の一度きり cleanup の結果:

```text
[QueueRetention] Cleaned 2026-05-02 02:30
  monthly-report-queue: 7 failed, 312 completed
  ab-testing-queue: 30 failed, 845 completed
  geo-closed-loop-queue: 4 failed, 198 completed
  geo-rlhf-queue: 0 failed, 67 completed
  geo-axp-queue: 2 failed, 630 completed
  ...
  Total: 43 failed + 2052 completed cleaned
```

きれいに掃除した後、admin UI がリアルタイムの状態に戻り、いかなる新しい失敗も即座に見える。1 万テナント scale では daily cleanup が ≥ 1 万件と見込まれ、retention worker は必須である。

### 15.4.3 KNOWN_QUEUES の保守

retention worker の `KNOWN_QUEUES` 配列は全プラットフォームのすべての queue 名を含まねばならず、さもなくば新 queue は永遠に掃除されない:

```js
const KNOWN_QUEUES = [
  'geo-axp-queue',
  'geo-axp-immediate',
  'geo-axp-bootstrap',
  'monthly-report-queue',
  'tier-a-visual-monthly',
  'billing-cycle-queue',
  'payment-dunning-queue',
  'reverse-scan-queue',
  'ab-testing-queue',
  'geo-closed-loop-queue',
  'geo-rlhf-queue',
  'geo-f12-optimize-queue',
  'queue-retention-queue',  // self
  // ... 其他 11 個
];
```

ある queue を書き漏らす → 失敗が永久累積する。漏れを避けるため、admin UI に `/admin/scan-frequencies` ページを加えた — BullMQ から全 queue 名をリストし `KNOWN_QUEUES` array と比べ、差集合を「⚠ retention 未収録」赤ラベルで表示する。新 queue を加えて KNOWN_QUEUES の更新を忘れると → admin が即座に見る。

### 15.4.4 LLM call site timeout の全プラットフォーム整合

wikiCompiler の他に、geo-saas backend も同時に LLM timeout を加えた(同種の silent hang を防ぐ):

- `services/modelRouter.service.js` Anthropic SDK `timeout: 60000, maxRetries: 0`(元のデフォルト 600s)
- `services/modelRouter.service.js` Gemini call に `Promise.race` + 60s timeout を追加
- `services/tier-a/visualEnrichment.service.js` Anthropic Vision API fetch に `signal: AbortSignal.timeout(60_000)` を追加

検証コマンド(commit 前の自己チェック):

```bash
grep -rn "fetch(.*api\.\(deepseek\|openai\|anthropic\|googleapis\|xiaomimimo\)" backend/src/services/ \
  | grep -v "AbortSignal\|signal:"
# 應為空,有 = 違反 LLM timeout 鐵律
```

この grep を pre-commit hook に加えた(計画中)、違反 → commit をブロック。1 万テナント scale はいかなる漏れも許容できない。

---

## 15.5 1 万テナントのデプロイパターン

### 15.5.1 Patches ディレクトリ構造

`rag-backend-v2` は geo-saas git repo にない(独立マイクロサービス)、patch 記録は `docs/rag-backend-v2-patches/` に保持する:

```text
docs/rag-backend-v2-patches/
├── README.md                          ← 部署順序 + verification block
├── wikiCompiler.js                    ← Patch 1, 2, 3, 6 都改這個
├── modelRouter.js                     ← Patch 5 改這個
├── docker-compose.rag.yml             ← Patch 4 DNS 設定
└── verification-block.sh              ← 一鍵驗證 6 patch live
```

RAG コンテナを再構築するたびに 3 つのファイルを一緒に scp しなければならない(1 つ漏らすと patch が回帰する)。

### 15.5.2 デプロイ + 検証 block

```bash
# 1. 從 patches/ 取最新版
scp wikiCompiler.js     ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp modelRouter.js      ubuntu@PROD:/home/ubuntu/rag-backend-v2/src/services/
scp docker-compose.rag.yml  ubuntu@PROD:/home/ubuntu/geo-saas/

# 2. 重 build + recreate
ssh ubuntu@PROD
cd /home/ubuntu/rag-backend-v2
sudo docker build -t geo-saas-prod-rag:latest .

cd /home/ubuntu/geo-saas
sudo docker compose -f docker-compose.rag.yml --project-name geo-saas-prod up -d --force-recreate rag

# 3. 驗證 6 個 patch 全 live
sudo docker exec geo-saas-prod-rag-1 sh -c '
  grep -c "selectedDocs = docs" /app/src/services/wikiCompiler.js     # patch 1: 1
  grep -c UUID_RE /app/src/services/wikiCompiler.js                   # patch 2: 2
  grep -c fallbackWrapArticle /app/src/services/wikiCompiler.js       # patch 3: 3
  grep -c postgres /etc/hosts                                         # patch 4 prep: 1+
  grep -c LLM_CALL_TIMEOUT_MS /app/src/services/modelRouter.js        # patch 5: 8
  grep -c "maxTokens: 2000" /app/src/services/wikiCompiler.js         # patch 6: ≥3
'
```

期待される出力は `1 / 2 / 3 / 1 / 8 / ≥3`。いずれか合致しなければ patch 回帰であり、即座に再 scp + rebuild する。PROD コンテナを再構築するたびにこの verification block を必ず持ち込む、さもなくば patch が黙って失われ同じ轍を踏む。

### 15.5.3 1 万テナント readiness の検証

patch のデプロイ以外に、1 万テナント readiness にはさらに以下が必要:

- **L1 hit rate monitoring**:`f12_cache_metrics` を毎時集計、目標 ≥65%。50% 未満 → admin alert
- **per-tenant LLM cost cap**:`f12_billing_records` を月度集計、starter が $1 超 / pro が $50 超 → email warn
- **fallback bucket size**:`SELECT COUNT(*) FROM wiki_pages WHERE 'wiki_fallback' = ANY(tags)` > 100 / brand → admin review
- **orphan tenant_documents**:`SELECT COUNT(*) FROM tenant_documents td WHERE NOT EXISTS (SELECT 1 FROM wiki_page_sources WHERE source_doc_id=td.id)` > 10% → compile を再トリガー

この 4 指標はすべて `/admin/rag-health` admin dashboard に入り、daily cron が算出後に alert を書く。

### 15.5.4 RAG コンテナ再構築の歴史

2026-04-23 から 2026-05-02 までのわずか 9 日で、rag コンテナは 11 回 recreate された:

| 日付 | 原因 | Patch |
|---|---|---|
| 04-23 | DeepSeek SDK のアップグレード | — |
| 04-25 | Wiki Cascade trigger の追加 | migration 170 |
| 04-27 | sync_wiki_page_sources trigger の追加 | migration 201 |
| 04-29 | Patch 1 (cutoff) | wikiCompiler.js |
| 04-30 | Patch 2 (UUID) | wikiCompiler.js |
| 04-30 | V4 reasoning への切り替え(Patch 6 の前) | platform_llm_config |
| 05-01 | Patch 3 (no marker) | wikiCompiler.js |
| 05-01 | Patch 4 v1 (extra_hosts IP) | docker-compose.rag.yml |
| 05-01 | Patch 4 v2 (dns 127.0.0.11 only) | docker-compose.rag.yml |
| 05-01 | Patch 4 v3 (dns + fallback) | docker-compose.rag.yml + .prod.yml |
| 05-02 | Patch 5 + 6 | modelRouter.js + wikiCompiler.js |

各再構築は ~3 分の downtime(build & recreate 中)、11 回で累計 ~33 分。次に大改修するときは blue-green deployment(双コンテナ交替)で downtime を減らすことを勧めるが、現状 quota が足りず 2 コンテナを同時に走らせられない。

---

## 15.6 観察と未解決問題

### 15.6.1 LLM が信頼できない本質

6 層の patch は一見よくある失敗モードをカバーしたように見えるが、LLM の進化速度は patch の速度より速い:

- model の切り替え(deepseek-v3 → v4-flash)は新しい hallucination 形態をもたらす(`maxTokens` が reasoning に足りない)
- 異なる prompt 形式は異なる model で異なる失敗モードを持つ(GPT-4 は marker を漏らさないが prefix「Here's the response:」を吐く)
- streaming vs non-streaming の挙動差(streaming は中断後に再送しない可能性)

これは LLM-driven システムの robustness 設計が**常に LLM が新しい方法で失敗すると仮定しなければならない**ことを意味し、新 model を 1 つ加えるごとに hallucination test を 1 巡回し直す必要がある。

### 15.6.2 fallback wrap article の品質

Patch 3 の低信頼 fallback は顧客の doc を救ったが、`confidence='low'` + `tags=['wiki_fallback']` の wiki page が L1 retrieval のとき実際どれほどの品質かは、現状**end-to-end evaluation をしていない**。理論上は:

- wiki_query_route の「あなたはどのカテゴリを聞いているか」LLM ルーティングに参加させない(ユーザーを騙さないため)
- admin UI に「⚠ 要審視」マークを表示
- N 個の失敗を累計したらシステムが admin を alert

3 点目はまだやっていない。

さらに、fallback 内容の「ソース紐付け」は存在するが(`sourceDocIds`)、「LLM の再構成品質」は検証されていない——LLM が顧客の無関係な 2 つの doc の内容を混ぜている可能性がある。これは人手または LLM-as-judge のサンプルレビューが必要で、V2 の作業である。

### 15.6.3 多 LLM ルーティング vs 単一 LLM 堅牢化

理論上、同じ query を 2 つの異なる LLM(DeepSeek + Gemini)で同時に回し、marker / UUID / 内容を交差検証すれば、大半の hallucination 問題を解ける。しかしコストが倍になり遅延も倍増する、それに見合うか?現状データが足りず決めきれない、V2 の評価に残す。

具体的な定量化:単一 wiki compile call は約 $0.0003(DeepSeek V4 flash)、双 LLM で $0.0006 + max latency 1.5s → 3s。84 docs KB compile $0.025 → $0.05。1 万テナント月 100 KB recompile = $50/月、許容できる。しかしまず A/B test で双 LLM が本当に hallucination を減らすか検証する必要がある(2 つの LLM が稀に同じ誤りを犯す可能性がある)。

### 15.6.4 Wiki Cascade の逆操作

削除は cascade が自動処理するが、**復元**(undo soft-delete)はやっていない。顧客が誤って文書を削除 → wiki_pages が soft delete → 30 日後に自動 hard delete。顧客が 7 日後に戻したいと思っても、現状はカスタマーサポートが手動で SQL UPDATE する。よくある要望だがエンジニアリングが手を付けていない。

完全な実装には以下が必要:

- Soft delete 期間中に顧客へ「30 日以内は復元可」UI を出す
- 復元時に逆向き cascade:wiki_page 復元 → junction 復元 → tenant_document 復元 → brand_document 復元
- 4 層それぞれに `deleted_at IS NOT NULL` チェック + restore 関数
- admin の強制 hard delete フロー(GDPR 適合、顧客が完全削除を要求)

エンジニアリング量は約 1〜2 週間、V2 で手を付ける。

### 15.6.5 Wiki コンパイル失敗の復旧可能性

V1 のコンパイル失敗 → DB に `last_compile_error` を書き、UI に「コンパイル失敗」を表示。しかし**リトライ機構がない** — admin が手動でボタンを押す必要がある。1 万テナント readiness では:

- 失敗を自動リトライ 1 回(LLM timeout 類の一時的エラーなら)
- N 回連続失敗 → admin alert、黙って死なせない
- failure の分類:transient(timeout / 5xx)→ 自動リトライ;permanent(invalid input / over quota)→ リトライしない

これも V2 の作業である。

### 15.6.6 RAG マイクロサービス間の idempotent guarantee

`ragDeleteDocument` 失敗後に `brand_documents` はすでに soft delete だが `tenant_documents` は掃除されず、orphan tenant_document(対応する brand_document がない)を生む。現状は daily cron が orphan を捕まえて再度掃除するが、この cron は実行漏れや失敗の可能性がある。

よりクリーンな設計は outbox pattern である:

```sql
-- brand_documents.delete 時寫 outbox row
INSERT INTO outbox_events(event_type, payload, status)
VALUES ('rag_delete_document', '{"central_doc_id": "..."}', 'pending');

-- 獨立 worker 讀 outbox,call ragDeleteDocument,成功才標 'done'
-- 失敗自動 retry,長期失敗 alert
```

しかし outbox を適用するには cascade flow 全体を再構築する必要があり、V2 の作業量が大きく、現状は daily cron で当面十分である。

---

## 15.7 エンジニアリングの教訓

6 個の patch + Worker robustness の進化(2 週間、5 回の PROD 障害)から、5 条の教訓を整理した:

### 15.7.1 LLM の失敗は常に vocal より silent が多い

6 個の patch はどれも LLM が直接 error を throw したものではない。すべて「一見成功だが実際は空 / 切断 / 停止」だった:

- Patch 1:cutoff が黙って 60% の doc を切る、警告なし
- Patch 2:LLM が partial UUID を吐き、PG が throw するが transaction が飲み込む
- Patch 3:LLM が marker を漏らし、parser が 0 article を返し throw しない
- Patch 4:DNS SERVFAIL、pg-pool が永遠に待ち throw しない
- Patch 5:streaming 中断、SDK が永遠に [DONE] を待ち throw しない
- Patch 6:reasoning maxTokens 不足、API 200 OK で空 content を返し throw しない

**LLM-driven システムの debug の第一規則:「error が見えない」を信頼できず、「output が本当に期待した shape か」を検証しなければならない**。各 LLM call の後に schema validate(UUID_RE / marker / minLength)を加えるのが基本動作である。

### 15.7.2 silent hang は分散システムの見えない殺し屋

DNS SERVFAIL / streaming 中断 / reasoning content 空——3 種の状況すべてがシステムを「正常に動いているように見える」が 0 進度にする。従来の「失敗は throw する」という仮定が破綻する。

この種の失敗を防御するエンジニアリングパターン:

- **timeout 必須**:あらゆる外部 call(LLM / DB / 内部 service)は timeout を持つべき、手動リトライしても永遠に待つよりまし
- **進度ハートビート**:long-running task は 10 秒ごとに heartbeat を書き、heartbeat がなければ停止とみなす
- **silent は throw より危険**:try/catch を書くときは「成功経路でも output を検証する」ことを忘れずに、error を catch するだけでなく

### 15.7.3 三層 fallback は一層 retry より信頼できる

Patch 3 の fallback 設計は「**LLM 失敗 → 緩い解析 → まだ失敗 → doc title で保底包装**」の三層で、いずれかの層が成功すれば保底になる。一層 retry(失敗したら 3 回リトライ、まだ失敗なら throw)に比べ、三層 fallback は常に出口がある:

- 第一層の理想:LLM が完璧に出力、直接 parse
- 第二層の緩さ:LLM が marker を漏らす / prefix を加える、緩い regex が拾う
- 第三層の保底:全部失敗、doc title + content を single article に包む

代償:fallback bucket の wiki page 品質が低く、admin が見る必要がある。しかし「LLM 失敗 → doc が orphan」に比べれば、それだけの価値がある。

1 万テナント scale は「永久 orphan」を許容できず、顧客がアップロードしたいかなる内容も**出口を持たねばならない**。

### 15.7.4 Patch 順序が「下流が正常に見える」原理を明かす

6 個の patch の発見順序を振り返る——それぞれ直すと次を「暴露」する:

```text
看到 33/84 完成 → 修 cutoff → 看到 全部 doc 進編譯
看到 全部 doc 進編譯 → 看到第 13 個就斷 → 修 UUID → 看到 全部 doc 完成
看到 全部 doc 完成 → 看到 wiki page 數不對 → 修 marker → 看到 wiki 真實生成
看到 wiki 真實生成 → 看到 編譯卡住 → 修 DNS → 看到 編譯運行
看到 編譯運行 → 看到 卡在第 13 個 → 修 LLM timeout → 看到 編譯穩定
看到 編譯穩定 → 看到 wiki query 全 miss → 修 maxTokens → 看到 wiki 真實命中
```

各層は「**前の層の silent failure が次の層を覆い隠していた**」。debug は一層ずつ剥がねばならず、**一度に全部解こうと焦ると根本原因を見誤る**。

### 15.7.5 patches/ ディレクトリはマイクロサービスの SSOT

rag-backend-v2 は geo-saas git repo にないが、patch が入った後に失われやすい。`docs/rag-backend-v2-patches/` を SSOT として設計した:

- PROD への scp のたびにこのディレクトリから取る
- README にデプロイ順序 + verification block を書く
- コンテナ再構築のたびに verification を回して patch が live か確認する

**verification block を書かなければ、次の再構築で持ち込みを忘れて回帰する**。一度踏んだ:あるとき RAG をアップグレード再構築した際、wikiCompiler.js の scp を忘れ、Patch 6 の maxTokens=2000 が 200 に戻り、顧客が「Wiki がまたヒットしない」と反応。`'maxTokens: 2000'` を grep して count=0 で patch が持ち込まれていないと即座に分かった。以後 RAG container を recreate するたびに必ず verification を回し、二度と踏まない。

---

## 本章のまとめ {.unnumbered}

- rag-backend-v2 は独立マイクロサービス + 自家 cs_rag_db + Wiki Cascade 4 層カスケード削除でデータ一貫性を保証する
- 6 種の LLM ハルシネーション失敗モードは 6 層の patch に対応:cutoff / partial UUID / no marker / DNS / LLM timeout / reasoning maxTokens
- Patch 順序は「下流が正常に見える」原理を明かす:各層を直すごとに次の層の silent failure が見える
- Worker robustness 3 点セット:defensive guard(entity 予検)、per-item try/catch、daily retention 自動 cleanup(43 failed + 2052 completed を一度に掃除)
- 1 万テナントのデプロイは必ず patches/ ディレクトリ SSOT を通り、コンテナ再構築のたびに verification block を回す
- 未解決:LLM 進化速度 > patch 速度;fallback 品質未 eval;多 LLM 交差検証は評価待ち;wiki 復元機構が欠如;失敗リトライが欠如;outbox pattern は再構築待ち
- 5 条のエンジニアリング教訓:LLM 失敗は silent > vocal / silent hang は見えない殺し屋 / 三層 fallback > 一層 retry / Patch 順序の原理 / patches/ SSOT は必ず verification を回す

## 参考資料 {.unnumbered}

- [第 5 章 — 多 Provider ルーティング:エラーハンドリング層](./ch05-multi-provider-routing.md)
- [第 6 章 — AXP シャドウドキュメント交付](./ch06-axp-shadow-doc.md)
- [第 14 章 — F12 三層構造オプティマイザ](./ch14-f12-structural-optimizer.md)
- BullMQ Job Cleaning API: <https://docs.bullmq.io/guide/queues/removing-jobs>
- Docker DNS configuration: <https://docs.docker.com/network/#dns-services>
- Karpathy on LLM Wiki concept(2024 公開共有)

## 改訂履歴 {.unnumbered}

| 日付 | バージョン | 説明 |
|------|------|------|
| 2026-05-03 | v1.1 | 新章 — 6 層の LLM ハルシネーション patch + Wiki Cascade + Worker robustness |
| 2026-05-03 | v1.1.1 | 章を ~7100 字に拡充 — 15.1.1/3 Karpathy 概念と V1→V2 進化、15.2.2/3 SSOT trigger SQL と 36 entry デバッグ、15.3 各 patch に発見過程、15.3.8 Patch 順序の因果連鎖、15.4.3 KNOWN_QUEUES 保守、15.5.4 RAG 11 回再構築史、15.6.6 outbox pattern、15.7 エンジニアリング教訓 5 条を追加;章番号 14.x → 15.x の typo を修正 |

---

**ナビゲーション**：[← 第 14 章: F12 三層構造オプティマイザ](./ch14-f12-structural-optimizer.md) · [📖 目次](../README.md) · [第 16 章: プラットフォーム SSOT チェーン →](./ch16-platform-ssot-chain.md)

<!-- AI-friendly structured metadata -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "TechArticle",
  "headline": "第 15 章 — rag-backend-v2 の堅牢化:6 層の LLM ハルシネーション失敗モードに対する防御設計",
  "description": "中央 RAG マイクロサービス(rag-backend-v2)のエンジニアリング堅牢化史:Wiki Cascade 4 層カスケード、6 種の wikiCompiler ハルシネーション patch、Worker defensive guard と Queue retention、LLM-driven システムの robustness 設計実践を完全に提示する。5 条のエンジニアリング教訓を含む。",
  "author": {"@type": "Person", "name": "Vincent Lin", "affiliation": "Baiyuan Technology"},
  "datePublished": "2026-05-03",
  "inLanguage": "ja",
  "isPartOf": {
    "@type": "Book",
    "name": "百原 GEO Platform 技術白書",
    "url": "https://github.com/baiyuan-tech/geo-whitepaper"
  },
  "keywords": "rag-backend-v2, LLM Wiki Compiler, Wiki Cascade, LLM Hallucination Hardening, Docker DNS, LLM Timeout, BullMQ Retention, エンジニアリング教訓"
}
</script>
