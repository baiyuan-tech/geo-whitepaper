<!--
Qiita 投稿設定
-------------
Title: Stale Carry-Forward：外部API障害でダッシュボードを0%にしないためのデザインパターン
Tags: Architecture, API, Resilience, DesignPattern, SaaS
Private: false

投稿URL（投稿後に記入）: https://qiita.com/...
-->

# Stale Carry-Forward：外部API障害でダッシュボードを0%にしないためのデザインパターン

## TL;DR

- 外部 AI プロバイダに依存する SaaS では、**部分的障害**（レート制限、リージョン障害、クオータ枯渇）が必ず発生する。
- 素朴な対応（失敗をゼロ扱い／失敗プロバイダを分母から除外）は、**パイプライン健全性** と **ブランド状態** を混同する。ダッシュボードが毎日 80% → 0% → 75% のように乱高下する。
- 解：**Stale Carry-Forward** — 100% 失敗を検知したら最後の成功値を `isStale` フラグ付きで前送し、UI で「⚠ 14 時間前のデータ」を明示する。
- このパターンは「高頻度サンプリング、不安定ソース」型のあらゆる系（IoT、金融レート、ソーシャルモニタリング）に一般化できる。

---

## 1. 問題：なぜ失敗を 0 扱いしてはいけないか

15 の AI プラットフォーム（ChatGPT、Claude、Gemini、Perplexity、DeepSeek、Kimi…）に対して 4 時間毎にスキャンする構造を想定する。あるプラットフォームで以下が起きたとする：

- **レート制限**：429 連発で全クエリ失敗
- **リージョン障害**：プロバイダ側の数時間停止
- **クオータ枯渇**：月次配分消費
- **モデルバージョン切替中**：一時的に API 応答が不安定

素朴な実装はこうする：

```typescript
// アンチパターン
async function computeScore(brand) {
  const results = await Promise.allSettled(
    platforms.map(p => scanPlatform(brand, p))
  );
  const scores = results.map(r =>
    r.status === 'fulfilled' ? r.value.score : 0
  );
  return average(scores);
}
```

結果：1 プラットフォームが 4 時間落ちただけで、そのブランドのダッシュボードスコアが **85 → 57** に急落する。翌朝復旧すると 57 → 85。顧客は「何をすれば改善するのか」を読み取れない。

---

## 2. 誤った代替案：失敗プロバイダを分母から外す

次に思いつくのは「失敗を分母から除外する」：

```typescript
const scores = results
  .filter(r => r.status === 'fulfilled')
  .map(r => r.value.score);
return average(scores);
```

これは一見スマートだが、**別の問題を生む**：

- 先週は 15 プラットフォーム平均、今週は 14 プラットフォーム平均 → **比較不能**
- 偶然、失敗が続いたプロバイダがそのブランドで強かった場合、平均が嵩上げされる
- 顧客が「なぜスコアが上がったのか」を追跡しようとしても、カウントされたプロバイダ集合が時間で変わるため再現できない

**パイプライン健全性**（ツール側の問題）と **ブランド状態**（顧客側の実体）が混同されている。

---

## 3. Stale Carry-Forward のデザイン

### 3.1 基本ルール

- プラットフォーム毎のスキャンが **100% 失敗**したら、当該プラットフォームのスコアは DB を遡って **最後の成功値** を取る（最大 200 行、概ね 30 日相当）
- 持越値には `isStale = true` を付与
- UI はバッジ（黄色 ⚠）と tooltip（「14 時間前のデータ」）で明示
- 下流分析（平均計算、トレンド線）は持越値を使い続ける（= 時系列の連続性を壊さない）

### 3.2 状態機械

プラットフォームデータは 4 状態を持つ：

```text
Fresh（直近成功）
  ↓ 失敗検知
Stale（持越中、isStale=true）
  ↓ 72h 経過
Expired（データ古すぎ、UI で「期限切れ、現採点に含めず」）
  ↓ 復旧成功
Reset → Fresh
```

### 3.3 擬似コード

```typescript
async function getPlatformScore(brandId, platform) {
  const latest = await scanPlatform(brandId, platform);

  if (latest.allQueriesFailed) {
    // Stale Carry-Forward
    const lastSuccess = await db.query(`
      SELECT score, scanned_at FROM scan_results
      WHERE brand_id = $1 AND platform = $2
        AND status = 'success'
      ORDER BY scanned_at DESC
      LIMIT 1
      OFFSET 0
    `, [brandId, platform]);

    if (!lastSuccess) return { score: null, status: 'no-history' };

    const ageHours =
      (Date.now() - lastSuccess.scanned_at) / 3600_000;

    if (ageHours > 72) {
      return { score: null, status: 'expired', lastSeen: lastSuccess.scanned_at };
    }

    return {
      score: lastSuccess.score,
      isStale: true,
      staleAgeHours: ageHours,
      status: 'stale',
    };
  }

  return { score: latest.score, isStale: false, status: 'fresh' };
}
```

### 3.4 UI バッジ設計

フロント側では platform score カードに表示：

```text
┌─ ChatGPT ────────────────┐
│ 72 ⚠                     │
│ Citation Rate: 68%       │
│ [⚠ 14時間前のデータ]       │  ← hover tooltip で詳細
└──────────────────────────┘
```

72 時間超過で点滅バッジ、7 日超過で `Expired` として除外 + email 通知。

---

## 4. 落とし穴と設計判断

### 4.1 ベースラインテストは Stale Carry-Forward を適用しない

縦断的な真の変化を追跡する Phase ベースラインテストでは、いかなるキャリーフォワードも目的を汚染する。ベースラインテストがプラットフォーム失敗に遭えば、その Phase 結果は直接 `status = incomplete` マークし、回復後に再実行する。

### 4.2 キャッシュとの区別

Redis キャッシュによる重複 API コスト削減とは目的が違う：

| | Redis キャッシュ | Stale Carry-Forward |
|-|----|----|
| 目的 | コスト削減 | 信号連続性の維持 |
| TTL | 数分〜数時間 | 最大 72 時間 |
| フラグ表示 | しない | `isStale` で明示 |
| 発動条件 | 同一クエリ | プラットフォーム 100% 失敗 |

### 4.3 「失敗」の定義が肝

- ネットワークエラー：失敗
- 429（レート制限）：失敗（ただし次回再試行）
- 200 だが空回答：**判定がプラットフォーム依存**。Claude は拒否応答を返しやすく、ChatGPT は「分からない」と答える。各プロバイダの典型的「無言」パターンを識別する辞書が要る。
- 部分失敗（20 クエリ中 3 失敗）：**失敗扱いしない**（17/20 で集計継続）

---

## 5. パターンとしての一般化

Stale Carry-Forward は「高頻度サンプリング、不安定ソース」に広く通用する：

- **IoT センサー**：センサー断線時に最終値保持 + 断線マーク
- **金融レート取得**：API 障害時に前日終値保持 + "PREV" タグ
- **ソーシャルモニタリング**：Twitter/X API 変更時に前週値保持
- **サーバメトリクス**：Pull-based 監視で一時断時の空白を埋める

いずれも **「データの欠落」を「ゼロ」と扱わない** 原理が共通する。欠落は欠落としてマークし、下流は意図的に受け入れる。

---

## 6. 哲学：「分からない」を素直に表示する

SaaS ダッシュボード設計の古典的な罠：**「空白恐怖症」** — UI に数字が表示されていないと顧客が不安になる、という仮定。

実際は逆で、**正確でない数字を出す方が信頼を毀損する**。「⚠ 14 時間前のデータ」は不便な事実を正直に伝える。これは顧客を信頼した設計であり、長期的には ROI がある。

---

## 参考

- **完全版ホワイトペーパー 第 4 章（Stale Carry-Forward 詳解）**：<https://github.com/baiyuan-tech/geo-whitepaper/blob/main/ja/ch04-stale-carry-forward.md>
- リポジトリ：<https://github.com/baiyuan-tech/geo-whitepaper>（CC BY-NC 4.0）
- 関連パターン：Circuit Breaker、Bulkhead、Timeout（Michael Nygard, *Release It!*）

---

*本稿は百元科技（Baiyuan Technology）のホワイトペーパー第 4 章の抜粋である。SaaS プロダクト：<https://geo.baiyuan.io>*
