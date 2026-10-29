# TheLook eCommerce データスキーマ・クレンジング・集計仕様書（Technical Specification）

- 文書種別: データエンジニアリング技術仕様書（Technical Data & Aggregation Specification）
- 対象データセット: `<PROJECT_ID>.thelook_soap`（BigQuery `US` マルチリージョン）
- 対象オントロジーグラフ: `<PROJECT_ID>.thelook_soap.TheLookEcommerceGraph`
- 対応する業務規定書類（Google Docs）:
  1. `FIN-POL-2024-009`：TheLook eCommerce 財務会計ポリシー規定書
  2. `CRM-MD-2025-014`：TheLook eCommerce CRM・MD 業務運用ポリシー規定書

---

## 1. 概要と目的

本技術仕様書は、財務経理本部が策定した『財務会計ポリシー規定書（`FIN-POL-2024-009`）』および CRM・MD 統括部が策定した『CRM・MD 業務運用ポリシー規定書（`CRM-MD-2025-014`）』を、BigQuery データセット `thelook_soap` 上の SQL クエリ、Property Graph（`TheLookEcommerceGraph`）、および Semantic Measures として正確に実装するための技術仕様を定義する。

Knowledge Studio の Enrichment エージェントおよび Conversational Analytics in BigQuery は、Google ドキュメント上の業務ポリシーと本技術仕様書を照合し、スキーマ定義、結合パス、データクレンジング処理、および確定計算式を生成する。

---

## 2. 物理データモデルと全 6 テーブル・56 カラム仕様

### 2.1 テーブル一覧と主キー・外部キー関係

| テーブル名 | 論理エンティティ名 (Graph Label) | 主キー (`PRIMARY KEY`) | 外部キー (`FOREIGN KEY`) と参照先 | テーブルの技術的役割 |
| :--- | :--- | :--- | :--- | :--- |
| `distribution_centers` | `LogisticsCenter` | `id` (`INT64`) | なし | 物流・配送センターマスタ（全 10 拠点）。商品の出荷元および物理在庫の保管拠点を管理する。 |
| `users` | `Customer` | `id` (`INT64`) | なし | 登録顧客マスタ。顧客の属性、居住国（`country`）、登録日時、流入チャネルを管理する。 |
| `products` | `MerchandiseProduct` | `id` (`INT64`) | `distribution_center_id` -> `distribution_centers(id)` | 取扱商品カタログマスタ。商品カテゴリ（`category`）、部門（`department`）、標準仕入原価（`cost`）、カタログ定価（`retail_price`）を管理する。 |
| `orders` | `PurchaseOrder` | `order_id` (`INT64`) | `user_id` -> `users(id)` | 注文ヘッダトランザクション。1 注文につき 1 行生成され、注文全体のステータスと購入アイテム点数を保持する。 |
| `order_items` | `OrderLineItem` | `id` (`INT64`) | - `order_id` -> `orders(order_id)`<br/>- `user_id` -> `users(id)`<br/>- `product_id` -> `products(id)`<br/>- `inventory_item_id` -> `inventory_items(id)` | 注文明細トランザクション。個別アイテムごとの実販売価格（`sale_price`）、明細ステータス（`status`）、注文日時（`created_at`）を保持する売上・利益集計のメインファクトテーブル。 |
| `inventory_items` | `WarehouseInventoryUnit` | `id` (`INT64`) | - `product_id` -> `products(id)`<br/>- `product_distribution_center_id` -> `distribution_centers(id)` | 倉庫内の物理在庫ユニットテーブル。入庫日時（`created_at`）、販売出庫日時（`sold_at`）、原価（`cost`）、商品定価スナップショット（`product_retail_price`）を管理する。 |

### 2.2 カラム定義詳細

#### (1) `distribution_centers`
- `id` (`INT64`): 物流センターの一意識別子（主キー）。`products.distribution_center_id` および `inventory_items.product_distribution_center_id` から参照される。
- `name` (`STRING`): 物流センターの拠点名称（例: `'Chicago IL'`, `'Memphis TN'`, `'Houston TX'` 等）。
- `latitude` (`FLOAT64`): 物流センター所在地の緯度。
- `longitude` (`FLOAT64`): 物流センター所在地の経度。

#### (2) `users`
- `id` (`INT64`): 顧客の一意識別子（主キー）。`orders.user_id` および `order_items.user_id` から参照される。
- `first_name` (`STRING`): 顧客の名。
- `last_name` (`STRING`): 顧客の姓。
- `email` (`STRING`): 顧客のメールアドレス。
- `age` (`INT64`): 顧客の年齢。
- `gender` (`STRING`): 顧客の性別（`'M'`, `'F'`）。
- `state` (`STRING`): 居住州・都道府県。
- `street_address` (`STRING`): 番地・ストリートアドレス。
- `postal_code` (`STRING`): 郵便番号。
- `city` (`STRING`): 居住都市。
- `country` (`STRING`): 顧客の居住国名。ヒストリカルデータに `'Brasil'`（正しくは `'Brazil'`）および `'España'`（正しくは `'Spain'`）の表記揺れが含まれるため、第 3 章の正規化仕様を適用すること。
- `latitude` (`FLOAT64`): 顧客居住地の緯度。
- `longitude` (`FLOAT64`): 顧客居住地の経度。
- `traffic_source` (`STRING`): 獲得流入元チャネル（`'Search'`, `'Organic'`, `'Facebook'`, `'Email'`, `'Display'`）。
- `created_at` (`TIMESTAMP`): 顧客アカウント登録日時（UTC）。

#### (3) `products`
- `id` (`INT64`): 商品の一意識別子（主キー）。
- `cost` (`FLOAT64`): 商品の 1 単位あたり標準仕入原価（USD）。売上総利益および監査調整後純営業利益の算出における原価基準カラム。
- `category` (`STRING`): 商品カテゴリ（例: `'Outerwear & Coats'`, `'Jeans'`, `'Sweaters'`, `'Tops & Tees'` 等）。財務規定 `FIN-POL-2024-009` の監査返品引当率 $\alpha$ の判定キー。
- `name` (`STRING`): 商品表示名。
- `brand` (`STRING`): ブランド名。
- `retail_price` (`FLOAT64`): 商品のカタログ標準小売価格・定価（USD）。
- `department` (`STRING`): 商品の対象部門（`'Men'`, `'Women'`）。
- `sku` (`STRING`): SKU コード。
- `distribution_center_id` (`INT64`): 管轄物流センター ID（`distribution_centers.id` への外部キー）。

#### (4) `orders`
- `order_id` (`INT64`): 注文ヘッダの一意識別子（主キー）。
- `user_id` (`INT64`): 注文顧客 ID（`users.id` への外部キー）。
- `status` (`STRING`): 注文全体のステータス（`'Complete'`, `'Shipped'`, `'Processing'`, `'Cancelled'`, `'Returned'`）。
- `gender` (`STRING`): 注文時点の顧客性別。
- `created_at` (`TIMESTAMP`): 注文作成日時（UTC）。
- `returned_at` (`TIMESTAMP`): 返品日時（未返品は `NULL`）。
- `shipped_at` (`TIMESTAMP`): 出荷日時（未出荷は `NULL`）。
- `delivered_at` (`TIMESTAMP`): 配達完了日時（未配達は `NULL`）。
- `num_of_item` (`INT64`): 注文に含まれる明細アイテム点数。

#### (5) `order_items`
- `id` (`INT64`): 注文明細の一意識別子（主キー）。
- `order_id` (`INT64`): 親注文 ID（`orders.order_id` への外部キー）。有効注文数のカウントには `COUNT(DISTINCT order_id)` を使用する。
- `user_id` (`INT64`): 購入顧客 ID（`users.id` への外部キー）。
- `product_id` (`INT64`): 購入商品 ID（`products.id` への外部キー）。
- `inventory_item_id` (`INT64`): 引当在庫ユニット ID（`inventory_items.id` への外部キー）。
- `status` (`STRING`): 明細単位の取引ステータス（`'Complete'`, `'Shipped'`, `'Processing'`, `'Cancelled'`, `'Returned'`）。財務規定 `FIN-POL-2024-009` に基づき、有効取引の集計では必ず `status = 'Complete'` でフィルタする。
- `created_at` (`TIMESTAMP`): 注文明細の作成・購入日時（UTC）。期間集計時は `DATE(created_at)` で日付変換する。
- `shipped_at` (`TIMESTAMP`): 出荷日時（UTC）。
- `delivered_at` (`TIMESTAMP`): 配達完了日時（UTC）。
- `returned_at` (`TIMESTAMP`): 返品日時（UTC、未返品は `NULL`）。
- `sale_price` (`FLOAT64`): 顧客への実販売価格（USD）。カタログ売上総額（Gross Revenue）の集計対象カラム。

#### (6) `inventory_items`
- `id` (`INT64`): 物理在庫ユニットの一意識別子（主キー）。
- `product_id` (`INT64`): 商品 ID（`products.id` への外部キー）。
- `created_at` (`TIMESTAMP`): 倉庫への入庫日時（UTC）。在庫滞留日数の起算日（`DATE(created_at)`）。
- `sold_at` (`TIMESTAMP`): 販売・出庫日時（UTC）。未販売在庫（Unsold Inventory）は `sold_at IS NULL` で判定する。
- `cost` (`FLOAT64`): 在庫ユニットの仕入原価（USD）。在庫原価総額および不良在庫廃棄・物流手数料（15%）の算出基準カラム。
- `product_category` (`STRING`): 商品カテゴリスナップショット。
- `product_name` (`STRING`): 商品名スナップショット。
- `product_brand` (`STRING`): ブランド名スナップショット。
- `product_retail_price` (`FLOAT64`): 商品のカタログ定価スナップショット（USD）。未販売在庫の定価総額および不良在庫グロス売却回収額（35%）の算出基準カラム。
- `product_department` (`STRING`): 商品部門スナップショット（`'Men'`, `'Women'`）。
- `product_sku` (`STRING`): SKU スナップショット。
- `product_distribution_center_id` (`INT64`): 保管物流センター ID（`distribution_centers.id` への外部キー）。

---

## 3. データクレンジングおよびフィルタリング技術仕様

### 3.1 有効取引フィルタ仕様（`FIN-POL-2024-009` 第2章対応）
- 売上、原価、粗利、純売上高、純営業利益、および VIP 顧客の購買実績を集計するすべてのクエリにおいて、`order_items` テーブルに対して以下の `WHERE` 条件を必須とする。
  ```sql
  WHERE oi.status = 'Complete'
  ```
- 注意事項：`oi.status NOT IN ('Cancelled', 'Returned')` のような否定条件は、未実現収益である `'Processing'` および `'Shipped'` が混入するため使用してはならない。

### 3.2 顧客居住国（`users.country`）の表記揺れ正規化仕様（`CRM-MD-2025-014` 第2.2節対応）
- `users.country` カラムには、ポルトガル語表記の `'Brasil'` と英語表記の `'Brazil'`、およびスペイン語表記の `'España'` と英語表記の `'Spain'` が混在している。
- 国別集計（`GROUP BY`）を行う際は、必ず以下の `CASE` 式によって標準英語国名へ正規化する。
  ```sql
  CASE
    WHEN u.country = 'Brasil' THEN 'Brazil'
    WHEN u.country = 'España' THEN 'Spain'
    ELSE u.country
  END AS country
  ```

### 3.3 未販売在庫フィルタと評価基準日仕様（`CRM-MD-2025-014` 第2.1節・第4章対応）
- 未販売在庫の抽出条件：
  ```sql
  WHERE inv.sold_at IS NULL
  ```
- 評価基準日（As-of Date）は `DATE '2025-01-01'` に固定する。`CURRENT_DATE()` 関数を使用すると実行日によって滞留日数や経過日数が変動するため使用禁止とする。
- 在庫滞留日数（Holding Days）の算出式：
  ```sql
  DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY)
  ```

---

## 4. 業務規定に対応する確定集計ロジックと計算式

### 4.1 財務会計ポリシー（`FIN-POL-2024-009`）の SQL 実装仕様

1. **有効注文数（Valid Orders）**: `COUNT(DISTINCT oi.order_id)`
2. **有効販売アイテム数（Valid Line Items）**: `COUNT(oi.id)`
3. **カタログ売上総額（Gross Revenue）**: `ROUND(SUM(oi.sale_price), 2)`
4. **総原価（Total Cost）**: `ROUND(SUM(p.cost), 2)`（`order_items.product_id = products.id` で結合）
5. **売上総利益（粗利 / Gross Margin）**: `ROUND(SUM(oi.sale_price - p.cost), 2)`（または Semantic Measure `AGG(oi.gross_margin_measure)`）
6. **カテゴリ別 監査返品引当率（$\alpha$）と算入係数（$1 - \alpha$）**:
   - `p.category = 'Outerwear & Coats'` -> $\alpha = 0.05$（算入係数 `0.95`）
   - `p.category IN ('Jeans', 'Sweaters')` -> $\alpha = 0.03$（算入係数 `0.97`）
   - 上記以外 (`ELSE`) -> $\alpha = 0.01$（算入係数 `0.99`）
7. **法定準備金率（$\beta$）と算入係数（$1 - \beta$）**:
   - 全カテゴリ一律 $\beta = 0.02$（算入係数 `0.98`）
8. **監査調整後純売上高（Audited Net Revenue）**:
   ```sql
   ROUND(
     SUM(oi.sale_price)
     * CASE
         WHEN p.category = 'Outerwear & Coats' THEN 0.95
         WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97
         ELSE 0.99
       END
     * 0.98,
     2
   ) AS audited_net_revenue
   ```
9. **監査調整後純営業利益（Audited Net Operating Profit）**:
   ```sql
   ROUND(
     (
       SUM(oi.sale_price)
       * CASE
           WHEN p.category = 'Outerwear & Coats' THEN 0.95
           WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97
           ELSE 0.99
         END
       * 0.98
     ) - SUM(p.cost),
     2
   ) AS audited_net_operating_profit
   ```

### 4.2 CRM・MD 業務運用ポリシー（`CRM-MD-2025-014`）の SQL 実装仕様

1. **離反リスク VIP 顧客（At-Risk VIP Customers）の抽出 CTE 仕様**:
   - `oi.status = 'Complete'` でフィルタした上で `oi.user_id` ごとに集計し、`HAVING` 句で以下の 3 条件を課す。
     - `COUNT(DISTINCT oi.order_id) >= 2`
     - `SUM(oi.sale_price) >= 300.0`
     - `MAX(DATE(oi.created_at)) < DATE '2024-01-01'`
   - 顧客ごとの最終購入経過日数：`DATE_DIFF(DATE '2025-01-01', MAX(DATE(oi.created_at)), DAY) AS days_since_last_order`
2. **長期滞留未販売在庫（Deadstock Inventory）の抽出条件**:
   - `inv.sold_at IS NULL AND DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY) >= 180`
3. **不良在庫一掃純回収見込額（Net Clearance Recovery Value）の算定式**:
   - 定価総額の 35%（グロス売却回収額）から原価総額の 15%（廃棄・物流作業手数料）を控除：
   ```sql
   ROUND(
     SUM(inv.product_retail_price) * 0.35 - SUM(inv.cost) * 0.15,
     2
   ) AS net_clearance_recovery_value
   ```

---

## 5. 標準検証タスク（`T1`〜`T5`）の Golden SQL テンプレート

### `T1`：2023年有効取引の基本財務集計
```sql
SELECT
  COUNT(DISTINCT oi.order_id) AS valid_orders,
  COUNT(oi.id) AS valid_items,
  ROUND(SUM(oi.sale_price), 2) AS gross_revenue,
  ROUND(SUM(p.cost), 2) AS total_cost,
  ROUND(SUM(oi.sale_price - p.cost), 2) AS gross_margin
FROM `thelook_soap.order_items` AS oi
JOIN `thelook_soap.products` AS p
  ON oi.product_id = p.id
WHERE
  oi.status = 'Complete'
  AND DATE(oi.created_at) BETWEEN '2023-01-01' AND '2023-12-31';
```

### `T2`：物流センター別の未販売在庫と平均在庫滞留日数（2025-01-01 基準）
```sql
SELECT
  dc.name AS distribution_center,
  COUNT(inv.id) AS unsold_inventory_units,
  ROUND(SUM(inv.product_retail_price), 2) AS unsold_retail_value,
  ROUND(SUM(inv.cost), 2) AS unsold_cost_value,
  ROUND(AVG(DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY)), 2) AS avg_holding_days
FROM `thelook_soap.inventory_items` AS inv
JOIN `thelook_soap.distribution_centers` AS dc
  ON inv.product_distribution_center_id = dc.id
WHERE inv.sold_at IS NULL
GROUP BY dc.name
ORDER BY unsold_inventory_units DESC;
```

### `T3`：財務規定 `FIN-POL-2024-009` に基づくカテゴリ別 監査調整後純売上高・純営業利益（2023年）
```sql
SELECT
  p.category,
  COUNT(oi.id) AS valid_item_count,
  ROUND(SUM(oi.sale_price), 2) AS gross_revenue,
  ROUND(SUM(p.cost), 2) AS total_cost,
  ROUND(
    SUM(oi.sale_price)
    * CASE
        WHEN p.category = 'Outerwear & Coats' THEN 0.95
        WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97
        ELSE 0.99
      END
    * 0.98,
    2
  ) AS audited_net_revenue,
  ROUND(
    (
      SUM(oi.sale_price)
      * CASE
          WHEN p.category = 'Outerwear & Coats' THEN 0.95
          WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97
          ELSE 0.99
        END
      * 0.98
    ) - SUM(p.cost),
    2
  ) AS audited_net_operating_profit
FROM `thelook_soap.order_items` AS oi
JOIN `thelook_soap.products` AS p
  ON oi.product_id = p.id
WHERE
  oi.status = 'Complete'
  AND DATE(oi.created_at) BETWEEN '2023-01-01' AND '2023-12-31'
  AND p.category IN ('Outerwear & Coats', 'Jeans', 'Sweaters')
GROUP BY p.category
ORDER BY audited_net_operating_profit DESC;
```

### `T4`：CRM 規定 `CRM-MD-2025-014` に基づく国別（表記揺れ正規化後）離反リスク VIP 顧客集計
```sql
WITH vip_users AS (
  SELECT
    oi.user_id,
    COUNT(DISTINCT oi.order_id) AS lifetime_orders,
    SUM(oi.sale_price) AS lifetime_spend,
    MAX(DATE(oi.created_at)) AS last_order_date,
    DATE_DIFF(DATE '2025-01-01', MAX(DATE(oi.created_at)), DAY) AS days_since_last_order
  FROM `thelook_soap.order_items` AS oi
  WHERE oi.status = 'Complete'
  GROUP BY oi.user_id
  HAVING
    COUNT(DISTINCT oi.order_id) >= 2
    AND SUM(oi.sale_price) >= 300.0
    AND MAX(DATE(oi.created_at)) < DATE '2024-01-01'
)
SELECT
  CASE
    WHEN u.country = 'Brasil' THEN 'Brazil'
    WHEN u.country = 'España' THEN 'Spain'
    ELSE u.country
  END AS country,
  COUNT(v.user_id) AS at_risk_vip_count,
  ROUND(SUM(v.lifetime_spend), 2) AS total_vip_lifetime_spend,
  ROUND(AVG(v.days_since_last_order), 2) AS avg_days_since_last_order
FROM vip_users AS v
JOIN `thelook_soap.users` AS u
  ON v.user_id = u.id
GROUP BY country
ORDER BY at_risk_vip_count DESC, total_vip_lifetime_spend DESC
LIMIT 10;
```

### `T5`：MD 規定 `CRM-MD-2025-014` に基づく物流センター×部門別 長期滞留未販売在庫と純回収見込額
```sql
SELECT
  dc.name AS distribution_center,
  inv.product_department AS department,
  COUNT(inv.id) AS deadstock_units,
  ROUND(SUM(inv.product_retail_price), 2) AS total_retail_value,
  ROUND(SUM(inv.cost), 2) AS total_cost,
  ROUND(
    SUM(inv.product_retail_price) * 0.35 - SUM(inv.cost) * 0.15,
    2
  ) AS net_clearance_recovery_value
FROM `thelook_soap.inventory_items` AS inv
JOIN `thelook_soap.distribution_centers` AS dc
  ON inv.product_distribution_center_id = dc.id
WHERE
  inv.sold_at IS NULL
  AND DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY) >= 180
GROUP BY distribution_center, department
ORDER BY net_clearance_recovery_value DESC
LIMIT 10;
```
