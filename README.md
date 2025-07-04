# 浜松市公園情報マップ - バックエンドAPI (Strapi)

このリポジトリは、「浜松市公園情報マップ」アプリケーションのバックエンドAPIです。
フロントエンド（Next.js）は、このAPIから公園や遊具の情報を取得し、インタラクティブな地図アプリケーションを構築します。

- **本番環境APIベースURL:** `https://parks-strapi-api.onrender.com`
- **技術スタック:** Strapi, Node.js, PostgreSQL (Neon DB)
- **ホスティング:** Render

---

## 🚀 API利用ガイド for Next.js Developers

このガイドは、Next.jsで公園マップを開発するために必要なAPIの仕様と実践的な使い方を説明します。

### APIの基本思想とレスポンス構造

StrapiのAPIレスポンスは、常に以下の構造を基本としています。

```json
{
  "data": [ /* データが配列の場合 */ ], 
  "data": { /* データが単一オブジェクトの場合 */ },
  "meta": { /* ページネーション等の付随情報 */ }
}
```

実際のデータ本体は、`data` プロパティの中にあります。さらに、各データは `id` と `attributes` に分かれており、フィールドの値は `attributes` の中に格納されています。

```json
{
  "id": 10010,
  "attributes": {
    "park_name": "相生公園",
    "createdAt": "2025-07-04T...",
    // ... 他のフィールド ...
  }
}
```

### 認証について

全てのAPIリクエストには、HTTPヘッダーにAPIトークンを含める必要があります。
**`.env.local` ファイルでAPIのURLとトークンを管理することを強く推奨します。**

```.env.local
NEXT_PUBLIC_STRAPI_API_URL=https://parks-strapi-api.onrender.com
NEXT_PUBLIC_STRAPI_API_TOKEN=<別途共有されるAPIトークン>
```

リクエストの際は、`Authorization` ヘッダーにトークンを設定します。

```javascript
const response = await fetch(
  `${process.env.NEXT_PUBLIC_STRAPI_API_URL}/api/some-endpoint`,
  {
    headers: {
      'Authorization': `Bearer ${process.env.NEXT_PUBLIC_STRAPI_API_TOKEN}`
    }
  }
);
```

---

## ✨ 機能実装のためのAPI活用レシピ

### 🗺️ レシピ1: 地図上に全ての公園のピンを立て、色分けする (推奨の統合API)

このアプリのメイン機能である「公園のピン立て」と「ハザードレベルに応じた色分け」は、このエンドポイント一つで完結します。
データベース側で事前に公園情報とハザードレベルを集計した、読み取り専用の `park_pin_summary` を利用します。

- **目的:** 地図に表示する全ての公園の位置、名前、およびピンの色分け用データを一括で取得する。
- **エンドポイント:** `GET /api/park-pin-summary`
- **レスポンスの構造:**
  ```json
  {
    "data": [
      {
        "id": 10010,
        "attributes": {
          "park_name": "相生公園",
          "lat": 34.7069954,
          "lon": 137.748491,
          "map_pin_color": 2 
        }
      },
      // ... 全ての公園のサマリーデータ
    ]
  }
  ```
  - `map_pin_color`: ピンの色を決定するための数値です。フロントエンド側で「1なら緑、2ならオレンジ、3なら赤」のように解釈して色を適用します。

- **フロントエンドでの実装例 (SWRを使用):**
  ```javascript
  import useSWR from 'swr';

  const fetcher = (url, token) => fetch(url, {
    headers: { 'Authorization': `Bearer ${token}` }
  }).then(res => res.json());

  function ParkMap() {
    const { data: summary, error } = useSWR(
      [`${process.env.NEXT_PUBLIC_STRAPI_API_URL}/api/park-pin-summary`, process.env.NEXT_PUBLIC_STRAPI_API_TOKEN],
      fetcher
    );

    if (error) return <div>Failed to load</div>;
    if (!summary) return <div>Loading...</div>;

    return (
      <MapContainer>
        {summary.data.map(park => (
          <Marker
            key={park.id}
            position={[park.attributes.lat, park.attributes.lon]}
            icon={getPinIconByColor(park.attributes.map_pin_color)} // ピンの色を返すヘルパー関数
          >
            <Popup>{park.attributes.park_name}</Popup>
          </Marker>
        ))}
      </MapContainer>
    );
  }
  ```

---

### 🖱️ レシピ2: ピンをクリックして公園と全遊具の詳細情報を表示する

ユーザーが特定のピンをクリックした際に、その公園の完全な詳細情報と、設置されている全遊具のリストを表示します。

- **目的:** 特定の`id`を持つ公園と、それに紐づく全ての遊具ハザードレベル情報を取得する。
- **エンドポイント:** `GET /api/parks/:id?populate=park_hazard_levels`
  - `:id` 部分には、クリックされた公園の`id`を動的に設定します。
  - `populate` クエリパラメータが、リレーション先のデータを一緒に取得するための鍵です。

- **URL例:** `https://parks-strapi-api.onrender.com/api/parks/10010?populate=park_hazard_levels`

- **レスポンスの構造 (抜粋):**
  ```json
  {
    "data": {
      "id": 10010,
      "attributes": {
        "park_name": "相生公園",
        "lat": 34.7069954,
        "lon": 137.748491,
        // ...公園の他の全フィールド...
        "park_hazard_levels": {
          "data": [
            {
              "id": 1,
              "attributes": {
                "equipment_name": "ブランコ",
                "hazard_level": 2,
                "note": "修繕が必要です。"
                // ...遊具の他の全フィールド...
              }
            },
            // ... 他の遊具データ ...
          ]
        }
      }
    }
  }
  ```
- **フロントエンドでの実装:**
  クリックイベントで公園の `id` を取得し、その `id` を使ってこのAPIを呼び出し、返ってきたデータをモーダルウィンドウ等に表示します。

---

### 🛠️ APIクエリ応用編: パラメータで取得データをカスタマイズする

StrapiのAPIは非常に柔軟です。URLの末尾にクエリパラメータを追加することで、取得するデータを細かく制御できます。
複雑なクエリは、`qs`ライブラリなどを使ってオブジェクトから生成すると便利です。

- **`fields` - フィールドの指定:** 必要なフィールドだけを取得してレスポンスを軽量化します。
  - `GET /api/parks?fields[0]=park_name&fields[1]=area_sqm`
  - (qs) `qs.stringify({ fields: ['park_name', 'area_sqm'] })`

- **`populate` - リレーションデータの取得:** 上記レシピでも使用。ネスト（入れ子）も可能です。
  - `GET /api/parks?populate=*` (全ての第一階層のリレーションを読み込む)
  - `GET /api/parks?populate[park_hazard_levels][fields][0]=equipment_name` (遊具の`equipment_name`だけ取得)

- **`filters` - 条件による絞り込み:** 特定の条件に合致するデータのみを取得します。
  - `GET /api/parks?filters[park_name][$contains]=中央` (公園名に「中央」を含む)
  - `GET /api/parks-hazard-levels?filters[hazard_level][$gte]=3` (ハザードレベルが3以上)

- **`sort` - 並び替え:**
  - `GET /api/parks?sort=park_name:asc` (公園名で昇順ソート)

- **`pagination` - ページネーション:**
  - `GET /api/parks?pagination[page]=2&pagination[pageSize]=25` (2ページ目の25件を取得)

**フィルタリング演算子一覧 (よく使うもの):**
| 演算子 | 説明 | 例 |
| :--- | :--- | :--- |
| `$eq` | 等しい | `filters[park_name][$eq]=あい公園` |
| `$ne` | 等しくない | `filters[hazard_level][$ne]=1` |
| `$lt` | より小さい | `filters[hazard_level][$lt]=3` |
| `$lte`| 以下 | `filters[hazard_level][$lte]=2` |
| `$gt` | より大きい | `filters[hazard_level][$gt]=2` |
| `$gte`| 以上 | `filters[hazard_level][$gte]=3` |
| `$in` | いずれかに合致 | `filters[id][$in][0]=10010&filters[id][$in][1]=10020` |
| `$contains` | (部分)文字列を含む | `filters[note][$contains]=修繕` |

---

## 📚 APIデータ構造 (テーブル定義)

APIから返ってくるデータのフィールド名と型の一覧です。

#### `park_info` (エンドポイント: `/api/parks`)
| フィールド名 | データ型 | 説明 |
| :--- | :--- | :--- |
| `id` | `integer` | 市が定める公園の管理番号 (主キー) |
| `park_name` | `string` | 公園名 |
| `park_name_kana` | `string` | 公園名のカナ |
| `lat` | `double` | 緯度 |
| `lon` | `double` | 経度 |
| `park_type` | `string` | 公園種別 (街区公園など) |
| `area_sqm` | `double` | 面積(㎡) |
| `link_url` | `string` | 関連URL |
| `image1`, `image2`| `string` | 画像URL |
| `number` | `integer`| (用途確認中) |
| `ward` | `string` | 区 |
| `locale` | `string` | (用途確認中) |

#### `park_hazard_levels` (エンドポイント: `/api/parks-hazard-levels`)
| フィールド名 | データ型 | 説明 |
| :--- | :--- | :--- |
| `id` | `integer`| このレコードのユニークID (主キー) |
| `equipment_id` | `integer` | 遊具の管理番号 |
| `equipment_name`| `string` | 遊具名 |
| `park_name` | `string` | (参考用) 公園名 |
| `address` | `text` | (参考用) 住所 |
| `comprehensive_evaluation` | `text` | 総合評価 |
| `usability` | `string` | 使用可否 |
| `deterioration` | `string` | 劣化状況 |
| `hazard_level` | `integer` | ハザードレベル |
| `note` | `text` | 特記事項 |
| `park_id` | `integer` | 関連する `park_info` の `id` |

#### `park_pin_summary` (エンドポイント: `/api/park-pin-summary`)
※ このエンドポイントは読み取り専用です。
| フィールド名 | データ型 | 説明 |
| :--- | :--- | :--- |
| `id` | `integer` | 公園の管理番号 (主キー) |
| `park_name` | `string` | 公園名 |
| `lat` | `double` | 緯度 |
| `lon` | `double` | 経度 |
| `map_pin_color` | `integer`| 地図のピンの色を示す数値 (例: 1=安全, 2=注意, 3=危険) |


### 管理画面
- **URL:** `https://parks-strapi-api.onrender.com/admin`

### ローカル開発
1.  **クローン:** `git clone https://github.com/u-yamii/park_strapi.git`
2.  **依存関係インストール:** `npm install`
3.  **`.env` ファイル作成:** `.env.example` をコピーし、共有された値を設定。
4.  **起動:** `npm run develop`