# 患者DB CSVインポート仕様書

作成日: 2026-04-21  
対象システム: patient-db (wiseinc-pharmacy.github.io/patient-db/)  
Firestoreプロジェクト: wise-patientdb

---

## 1. 対応CSVフォーマット

### 1-1. 患者マスターCSV

任意の列順に対応（ヘッダー自動マッピング方式）。

| 受理カラム名（例） | Firestoreフィールド | 型 | 説明 |
|---|---|---|---|
| 患者ID / ID | patientId | string | レセコン患者番号。照合キー① |
| 氏名 / 名前 / 漢字 / 患者名 | name | string | 氏名（漢字）。照合キー②（必須） |
| カナ / フリガナ | kana | string | 氏名カナ。半角カナ→全角変換 |
| 生年月日 / 誕生日 | birthday | string | YYYY-MM-DD 形式 |
| 性別 | gender | string | "男" / "女" |
| 郵便番号 / 〒 | zip | string | 000-0000 形式 |
| 住所 | address | string | |
| 電話番号 / 電話 / TEL | phone | string | |
| 担当店舗 / 店舗 | store | string | 柏/足立/ながれやま/アイ調剤/我孫子/さくら台 |
| 施設 / 分類 / 施設名 | facility | string | 訪問先施設名 |
| 備考 / メモ | note | string | |

**照合ロジック（既存患者との突合）:**
1. `patientId` が存在する → patientId 完全一致で照合
2. patientId が空 → `name` + `kana` 完全一致で照合
3. 一致なし → 新規追加

---

### 1-2. 請求CSV（レセプト集計）

固定カラム位置方式（2行ヘッダー構造）。

#### ファイル構造
```
行0: カラムヘッダー
行1: 単位行（円・点等の表示用）← スキップ
行2〜: データ行
```

#### カラムマッピング（0インデックス）

| 列 | CSVヘッダー | Firestoreフィールド | 型 | 保存 |
|---|---|---|---|---|
| 0 | 患者名 | name | string | ✅ |
| 1 | (空) | - | - | - |
| 2 | 給 | copayRatio | integer | ✅ 負担割合（0/7/8/9） |
| 3 | 保険種 | insuranceType | string | ✅ 例: 国老+介, 公+介 |
| 4 | 回数 | visits | integer | ✅ 訪問調剤回数 |
| 5 | 定負担 | copay | integer | ✅ 患者定額負担（円） |
| 6 | (介護負担) | careCopay | integer | ✅ 介護保険自己負担（円） |
| 7 | 選定療養 | selectTherapy | integer | ✅ 選定療養費（円） |
| 8 | 自費薬 | selfPayDrug | integer | ✅ 自費薬品費（円） |
| 9 | 請求額 | billedAmount | integer | ✅ 患者請求額（円） |
| 10 | 入金額 | paidAmount | integer | ✅ 入金済み額（円） |
| 11 | 現未収 | unpaidAmount | integer | ✅ 未収金額（円） |
| 12 | 保険計 | insuranceTotal | integer | ✅ 保険請求計（円） |
| 13 | 基本点 | basePoints | integer | ✅ 基本調剤料（点） |
| 14 | 時加算 | timePoints | integer | ✅ 時間外加算（点） |
| 15 | 調剤点 | dispensingPoints | integer | ✅ 調剤技術料（点） |
| 16 | 調加算 | dispensingAddPoints | integer | ✅ 調剤加算（点） |
| 17 | 指導点 | guidancePoints | integer | ✅ 薬剤服用歴管理指導料（点） |
| 18 | 内服薬 | oralMedPoints | integer | ✅ 内服薬剤料（点） |
| 19 | 内服外 | externalMedPoints | integer | ✅ 内服以外薬剤料（点） |
| 20 | 材料点 | materialPoints | integer | ✅ 特定保険医療材料料（点） |
| 21 | 患者ID | csvPatientId | string | ✅ レセコン患者ID（空欄あり） |
| 22 | 分類 | facility | string | ✅ 施設名・チーム名 |
| 23 | (カナ) | kana | string | ✅ 患者カナ読み（半角→全角変換） |
| 24 | 選定療養 | selectTherapyAmount | integer | ✅ 選定療養金額（円） |
| 25 | 内訳 | selectTherapyTax | integer | ✅ 選定療養税10%（円） |

#### Firestoreドキュメント構造（billing コレクション）

```json
{
  "name": "山田 太郎",
  "kana": "ヤマダ タロウ",
  "facility": "グランシア",
  "insuranceType": "国老+介",
  "copayRatio": 9,
  "visits": 2,
  "copay": 2508,
  "careCopay": 1368,
  "selectTherapy": 0,
  "selfPayDrug": 0,
  "billedAmount": 2508,
  "paidAmount": 2508,
  "unpaidAmount": 0,
  "insuranceTotal": 25080,
  "basePoints": 340,
  "timePoints": 0,
  "dispensingPoints": 154,
  "dispensingAddPoints": 0,
  "guidancePoints": 1772,
  "oralMedPoints": 210,
  "externalMedPoints": 32,
  "materialPoints": 0,
  "csvPatientId": "",
  "selectTherapyAmount": 0,
  "selectTherapyTax": 0,
  "period": "2026-03",
  "store": "柏",
  "patientRef": "FIRESTORE_DOC_ID_OR_NULL",
  "importedAt": "serverTimestamp",
  "importedBy": "user@example.com"
}
```

**ドキュメントID生成ルール:**
```
{period}_{store}_{name} の特殊文字を _ に置換
例: 2026-03_柏_山田　太郎
```
→ 同じ年月・店舗・患者名の場合は **上書き（upsert）**

---

## 2. バリデーションルール

### 共通

| チェック項目 | ルール | エラー時の動作 |
|---|---|---|
| ファイル選択 | 必須 | 処理中断・エラー表示 |
| ファイル形式 | .csv / .txt | ファイル選択ダイアログで制限 |
| 文字コード | UTF-8 / Shift-JIS 自動検出 | BOM優先、次にマルチバイトパターン判定 |
| 最小行数 | 2行以上（患者マスター） / 3行以上（請求CSV） | 処理中断・エラー表示 |

### 患者マスターCSV固有

| フィールド | ルール |
|---|---|
| name | 空の場合スキップ（`patientId` もない場合） |
| kana | 半角カナ → 全角カナ変換（濁点・半濁点の合字処理含む） |
| name | 全角・半角スペースを全角スペース1つに正規化、前後トリム |

### 請求CSVファイル固有

| フィールド | ルール |
|---|---|
| name (col 0) | 空・"合計"・"小計" の行はスキップ |
| 年月 (UI入力) | 必須。YYYY-MM 形式 |
| 店舗 (UI入力) | 必須。6店舗から選択 |
| 数値フィールド | `parseInt()` 変換。NaN → 0 で補完 |
| kana (col 23) | 半角カナ → 全角カナ変換 |

---

## 3. エラーハンドリング

| エラー種別 | 発生条件 | 対応 |
|---|---|---|
| ファイル未選択 | importFile / billingFile が空 | エラーメッセージ表示、処理中断 |
| 年月未選択 | billingPeriod が空（請求CSVのみ） | エラーメッセージ表示、処理中断 |
| データ不足 | CSVが2/3行未満 | エラーメッセージ表示、処理中断 |
| Firestore書き込みエラー | ネットワーク・権限エラー等 | `try-catch` でメッセージ表示 |
| バッチサイズ超過 | 1バッチ400件超え | バッチ分割コミット（★後述の修正で対応） |

### バッチ分割の正しい実装

```javascript
// ❌ 旧実装（バグあり）: batchをリセットしないため400件超でエラー
const batch = db.batch();
...
if (batchCount >= 400) {
  await batch.commit();  // コミット後も同じbatchに追加し続けてしまう
  batchCount = 0;
}

// ✅ 修正後: コミットのたびにbatchを再生成
let batch = db.batch();
let batchCount = 0;
...
if (batchCount >= 400) {
  await batch.commit();
  batch = db.batch();   // ← 再生成
  batchCount = 0;
}
```

---

## 4. 患者マスター照合ロジック（請求CSVインポート時）

```
請求CSV 1行につき:
  1. col[21] (csvPatientId) が空でない
       → patients.find(p => p.patientId === csvPatientId)
  2. 上記で未照合 かつ name が存在
       → patients.find(p => p.name === name || (kana && p.kana === kana))
  3. 照合成功 → billing.patientRef = matched.id（FirestoreドキュメントID）
  4. 照合失敗 → patientRef = null
                 patients コレクションに最小情報で新規追加
                 （name, kana, facility, store のみ）
```

---

## 5. 既存GASツールとの互換性メモ

（GASツール群のスプレッドシートURL共有後に更新予定）

- **ロボットペイメント請求データ**: 請求CSVと同一フォーマットと推定。患者IDの有無を確認要
- **処方箋作成ツール**: 患者マスターの `patientId` をキーに患者情報を参照すると想定
- **共通キー**: `patientId`（レセコン患者番号）を全ツール間の共通IDとして使用

---

## 6. 今後の拡張ポイント

| 拡張内容 | 優先度 | メモ |
|---|---|---|
| 生年月日・性別・電話番号の請求CSVからの補完 | 中 | 現状CSVには含まれていない |
| 請求データの月次集計レポート | 高 | 未収金額・請求額のサマリー表示 |
| 店舗別CSVバッチ取込（複数ファイル一括） | 低 | 現状は1ファイルずつ |
| レセコン直接連携 | 低 | CSV経由の手動運用を継続 |
