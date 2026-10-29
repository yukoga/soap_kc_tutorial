# Knowledge Catalog ビジネス用語集（Business Glossary）技術定義書

- 対象システム: Google Cloud Knowledge Catalog / Knowledge Studio
- 対象用語集 ID: `thelook-business-glossary`
- 対象ロケーション: `us-central1`
- 対象 BigQuery データセット: `<PROJECT_ID>.thelook_soap`
- 関連業務規定:
  - 財務会計ポリシー規定書（`FIN-POL-2024-009`）
  - CRM・MD 業務運用ポリシー規定書（`CRM-MD-2025-014`）

---

## 1. 概要

本ドキュメントは、ビジネス部門が策定した業務規定（`FIN-POL-2024-009` および `CRM-MD-2025-014`）に基づき、Google Cloud Knowledge Catalog のビジネス用語集（Business Glossary）へ登録すべき用語体系（カテゴリ、標準用語、技術定義、同義語、および BigQuery 物理カラムへのリンク仕様）を定義する技術仕様書である。

Knowledge Studio の Enrichment タスクは、本ドキュメントおよび登録済みの用語集を参照することで、自然言語の業務概念と BigQuery 上の物理カラム・計算式を一意にマッピングする。

---

## 2. 用語集メタデータとカテゴリ構成

### 2.1 用語集（Glossary）本体
- **Glossary ID**: `thelook-business-glossary`
- **Display Name**: `TheLook eCommerce 標準ビジネス用語集`
- **Location**: `us-central1`（Context Set および Enrichment タスクと同一リージョンに配置すること）
- **Description**: `TheLook eCommerce の財務会計指標（監査調整後純売上高・純営業利益）および CRM・MD 運用指標（離反リスクVIP顧客・長期滞留未販売在庫・平均在庫滞留日数）を定義する全社標準ビジネス用語集。`

### 2.2 カテゴリ（Categories）一覧

| カテゴリ ID (`categoryId`) | 表示名 (`displayName`) | 説明 (`description`) | 準拠する業務規定文書 |
| :--- | :--- | :--- | :--- |
| `finance-accounting-kpis` | `財務・会計KPI` | 財務会計ポリシー `FIN-POL-2024-009` に基づく有効取引、純売上高、純営業利益、および返品引当・法定準備金控除に関する標準定義。 | `FIN-POL-2024-009` |
| `crm-md-operational-metrics` | `CRM・MD運用指標` | CRM・MD 業務運用ポリシー `CRM-MD-2025-014` に基づく優良顧客離反リスク判定、国名表記揺れ正規化、在庫滞留日数、および不良在庫クリアランス回収額に関する標準定義。 | `CRM-MD-2025-014` |

---

## 3. 登録対象ビジネス用語（Terms）の技術定義とカラムマッピング

### 3.1 一覧表

| 項番 | 所属カテゴリ ID | 用語 ID (`termId`) | 表示名 (`displayName`) | 同義語・英語表記 (`synonyms`) | リンク対象 BigQuery カラム (`thelook_soap`) |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | `finance-accounting-kpis` | `valid-completed-order` | `有効取引 (Valid Completed Order)` | `有効な取引`, `完了取引`, `確定注文`, `Valid Order`, `Completed Transaction` | `order_items.status`<br/>`orders.status` |
| 2 | `finance-accounting-kpis` | `audited-net-revenue` | `監査調整後純売上高 (Audited Net Revenue)` | `純売上高`, `監査後純売上`, `引当控除後売上高`, `Audited Net Sales` | `order_items.sale_price`<br/>`products.category` |
| 3 | `finance-accounting-kpis` | `audited-net-operating-profit` | `監査調整後純営業利益 (Audited Net Operating Profit)` | `純営業利益`, `監査後営業利益`, `引当控除後利益`, `Audited Operating Income` | `order_items.sale_price`<br/>`products.cost`<br/>`products.category` |
| 4 | `crm-md-operational-metrics` | `at-risk-vip-customer` | `離反リスクVIP顧客 (At-Risk VIP Customer)` | `休眠リスクVIP顧客`, `離反懸念優良顧客`, `At-Risk VIP Users`, `Churned VIP` | `users.id`<br/>`users.country`<br/>`order_items.order_id`<br/>`order_items.sale_price`<br/>`order_items.created_at` |
| 5 | `crm-md-operational-metrics` | `deadstock-inventory` | `長期滞留未販売在庫 (Deadstock Inventory)` | `不良在庫`, `長期滞留在庫`, `デッドストック`, `Unsold Deadstock`, `Net Clearance Recovery Value` | `inventory_items.id`<br/>`inventory_items.sold_at`<br/>`inventory_items.created_at`<br/>`inventory_items.product_retail_price`<br/>`inventory_items.cost` |
| 6 | `crm-md-operational-metrics` | `avg-inventory-holding-days` | `平均在庫滞留日数 (Average Inventory Holding Days)` | `在庫滞留日数`, `平均保管日数`, `倉庫滞留日数`, `Holding Days`, `Days in Inventory` | `inventory_items.created_at`<br/>`inventory_items.sold_at` |

---

### 3.2 各用語の詳細定義仕様（Description 登録文面）

#### (1) `valid-completed-order`：有効取引 (Valid Completed Order)
- **所属カテゴリ**: `finance-accounting-kpis`
- **定義文**:
  財務会計上の売上高・売上原価・売上総利益・営業利益、および顧客購買実績の集計対象となる確定取引。注文明細テーブルにおいて `order_items.status = 'Complete'`（注文ヘッダ単体集計時は `orders.status = 'Complete'`）を満たすレコードのみを有効取引とする。`'Cancelled'`（キャンセル）、`'Returned'`（返品）、`'Processing'`（処理中）、`'Shipped'`（出荷済・配達未完了）の 4 ステータスは未実現または取消済であるため、有効取引の集計から必ず除外する。期間フィルタは原則として `DATE(order_items.created_at)`（UTC）を基準とする。
- **リンク先物理カラム**:
  - `thelook_soap.order_items.status`
  - `thelook_soap.orders.status`

#### (2) `audited-net-revenue`：監査調整後純売上高 (Audited Net Revenue)
- **所属カテゴリ**: `finance-accounting-kpis`
- **定義文**:
  財務規定 `FIN-POL-2024-009` に基づく監査引当・法定準備金控除後の純売上高（USD）。有効取引（`order_items.status = 'Complete'`）のカタログ売上総額 `SUM(order_items.sale_price)` に対し、第 1 段階として `products.category` 別の監査返品引当率 $\alpha$（`Outerwear & Coats` = 5%［算入係数 `0.95`］、`Jeans` および `Sweaters` = 3%［算入係数 `0.97`］、その他全カテゴリ = 1%［算入係数 `0.99`］）を控除し、第 2 段階として全取引一律の法定準備金率 $\beta = 2\%$（算入係数 `0.98`）を複利方式で乗じ、小数点第 2 位に四捨五入して算出する。
  確定式：`ROUND(SUM(oi.sale_price) * CASE WHEN p.category = 'Outerwear & Coats' THEN 0.95 WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97 ELSE 0.99 END * 0.98, 2)`。
- **リンク先物理カラム**:
  - `thelook_soap.order_items.sale_price`
  - `thelook_soap.products.category`

#### (3) `audited-net-operating-profit`：監査調整後純営業利益 (Audited Net Operating Profit)
- **所属カテゴリ**: `finance-accounting-kpis`
- **定義文**:
  財務規定 `FIN-POL-2024-009` に基づく純営業利益（USD）。監査調整後純売上高（`audited-net-revenue`）から、当該有効取引に対応する商品の標準仕入原価総額 `SUM(products.cost)`（引当率を乗じない確定原価の全額）を差し引き、小数点第 2 位に四捨五入して算出する。
  確定式：`ROUND((SUM(oi.sale_price) * CASE WHEN p.category = 'Outerwear & Coats' THEN 0.95 WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97 ELSE 0.99 END * 0.98) - SUM(p.cost), 2)`。
- **リンク先物理カラム**:
  - `thelook_soap.order_items.sale_price`
  - `thelook_soap.products.cost`
  - `thelook_soap.products.category`

#### (4) `at-risk-vip-customer`：離反リスクVIP顧客 (At-Risk VIP Customer)
- **所属カテゴリ**: `crm-md-operational-metrics`
- **定義文**:
  CRM 規定 `CRM-MD-2025-014` に基づき、評価基準日 `2025-01-01`（`DATE '2025-01-01'`）時点で離反・休眠状態にあると判定される優良顧客（`users.id`）。有効取引（`order_items.status = 'Complete'`）のみを対象に顧客単位で集計した際、(1) 累計有効注文回数 `COUNT(DISTINCT order_id) >= 2`、(2) 累計有効購入金額 `SUM(sale_price) >= 300.00`（USD）、(3) 最終有効購入日 `MAX(DATE(created_at)) < DATE '2024-01-01'`（基準日時点で 366 日以上購入なし）の 3 条件をすべて満たす顧客を指す。国別集計を行う際は、`users.country` の `'Brasil'` を `'Brazil'` に、`'España'` を `'Spain'` に必ず正規化して統合する。
- **リンク先物理カラム**:
  - `thelook_soap.users.id`
  - `thelook_soap.users.country`
  - `thelook_soap.order_items.order_id`
  - `thelook_soap.order_items.sale_price`
  - `thelook_soap.order_items.created_at`

#### (5) `deadstock-inventory`：長期滞留未販売在庫 (Deadstock Inventory)
- **所属カテゴリ**: `crm-md-operational-metrics`
- **定義文**:
  MD 規定 `CRM-MD-2025-014` に基づく不良・長期滞留在庫ユニット。物理在庫テーブル（`inventory_items`）において `sold_at IS NULL`（未販売）であり、かつ入庫日から評価基準日 `2025-01-01` までの滞留日数 `DATE_DIFF(DATE '2025-01-01', DATE(created_at), DAY)` が `180` 日以上のレコードを指す。長期滞留未販売在庫を一括クリアランス売却する際の「不良在庫一掃純回収見込額（Net Clearance Recovery Value）」は、定価総額の 35% から原価総額の 15%（廃棄・物流手数料）を差し引いた `ROUND(SUM(product_retail_price) * 0.35 - SUM(cost) * 0.15, 2)`（USD）で算出する。
- **リンク先物理カラム**:
  - `thelook_soap.inventory_items.id`
  - `thelook_soap.inventory_items.sold_at`
  - `thelook_soap.inventory_items.created_at`
  - `thelook_soap.inventory_items.product_retail_price`
  - `thelook_soap.inventory_items.cost`

#### (6) `avg-inventory-holding-days`：平均在庫滞留日数 (Average Inventory Holding Days)
- **所属カテゴリ**: `crm-md-operational-metrics`
- **定義文**:
  未販売在庫（`inventory_items.sold_at IS NULL`）が入庫日（`DATE(inventory_items.created_at)`）から評価基準日 `2025-01-01`（`DATE '2025-01-01'`）までに倉庫に保管・滞留している平均日数（日）。算出式は `ROUND(AVG(DATE_DIFF(DATE '2025-01-01', DATE(created_at), DAY)), 2)`、または `inventory_items` テーブルに定義された Semantic Measure `AGG(inventory_holding_days_measure)` を用いる。
- **リンク先物理カラム**:
  - `thelook_soap.inventory_items.created_at`
  - `thelook_soap.inventory_items.sold_at`

---

## 4. 機械可読定義（YAML 形式）

```yaml
glossary:
  id: thelook-business-glossary
  display_name: "TheLook eCommerce 標準ビジネス用語集"
  location: us-central1
  categories:
    - id: finance-accounting-kpis
      display_name: "財務・会計KPI"
      source_policy: "FIN-POL-2024-009"
      terms:
        - id: valid-completed-order
          display_name: "有効取引 (Valid Completed Order)"
          sql_filter: "order_items.status = 'Complete'"
          linked_columns:
            - "thelook_soap.order_items.status"
            - "thelook_soap.orders.status"
        - id: audited-net-revenue
          display_name: "監査調整後純売上高 (Audited Net Revenue)"
          sql_expression: "ROUND(SUM(oi.sale_price) * CASE WHEN p.category = 'Outerwear & Coats' THEN 0.95 WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97 ELSE 0.99 END * 0.98, 2)"
          linked_columns:
            - "thelook_soap.order_items.sale_price"
            - "thelook_soap.products.category"
        - id: audited-net-operating-profit
          display_name: "監査調整後純営業利益 (Audited Net Operating Profit)"
          sql_expression: "ROUND((SUM(oi.sale_price) * CASE WHEN p.category = 'Outerwear & Coats' THEN 0.95 WHEN p.category IN ('Jeans', 'Sweaters') THEN 0.97 ELSE 0.99 END * 0.98) - SUM(p.cost), 2)"
          linked_columns:
            - "thelook_soap.order_items.sale_price"
            - "thelook_soap.products.cost"
            - "thelook_soap.products.category"
    - id: crm-md-operational-metrics
      display_name: "CRM・MD運用指標"
      source_policy: "CRM-MD-2025-014"
      terms:
        - id: at-risk-vip-customer
          display_name: "離反リスクVIP顧客 (At-Risk VIP Customer)"
          sql_filter: "oi.status = 'Complete' GROUP BY oi.user_id HAVING COUNT(DISTINCT oi.order_id) >= 2 AND SUM(oi.sale_price) >= 300.0 AND MAX(DATE(oi.created_at)) < DATE '2024-01-01'"
          linked_columns:
            - "thelook_soap.users.id"
            - "thelook_soap.users.country"
            - "thelook_soap.order_items.order_id"
            - "thelook_soap.order_items.sale_price"
            - "thelook_soap.order_items.created_at"
        - id: deadstock-inventory
          display_name: "長期滞留未販売在庫 (Deadstock Inventory)"
          sql_filter: "inv.sold_at IS NULL AND DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY) >= 180"
          sql_expression: "ROUND(SUM(inv.product_retail_price) * 0.35 - SUM(inv.cost) * 0.15, 2)"
          linked_columns:
            - "thelook_soap.inventory_items.id"
            - "thelook_soap.inventory_items.sold_at"
            - "thelook_soap.inventory_items.created_at"
            - "thelook_soap.inventory_items.product_retail_price"
            - "thelook_soap.inventory_items.cost"
        - id: avg-inventory-holding-days
          display_name: "平均在庫滞留日数 (Average Inventory Holding Days)"
          sql_expression: "ROUND(AVG(DATE_DIFF(DATE '2025-01-01', DATE(inv.created_at), DAY)), 2)"
          linked_columns:
            - "thelook_soap.inventory_items.created_at"
            - "thelook_soap.inventory_items.sold_at"
```
