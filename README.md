# 浜松市公園情報マップ - バックエンドAPI (Strapi)

このリポジトリは、「浜松市公園情報マップ」アプリケーションのバックエンドAPIです。
フロントエンド（Next.js）は、このAPIから公園や遊具の情報を取得し、インタラクティブな地図アプリケーションを構築します。

- **本番環境APIベースURL:** `https://parks-strapi-api.onrender.com`
- **技術スタック:** Strapi, Node.js, PostgreSQL (Neon DB)
- **ホスティング:** Render

---

## 🚀 API利用ガイド for Next.js Developers

このガイドは、Next.jsで公園マップを開発するために必要なAPIの仕様と実践的な使い方を説明します。

### 認証について

全てのAPIリクエストには、HTTPヘッダーにAPIトークンを含める必要があります。
Next.jsのデータフェッチライブラリ（SWR, React Query, または `fetch`）のヘッダー設定に以下を追加してください。

```javascript
// 例: fetch を使う場合
const options = {
  headers: {
    'Authorization': `Bearer <APIトークン>` // トークンは別途共有
  }
};

const response = await fetch('APIのURL', options);```
**`.env.local` ファイルでの管理を推奨します:**
```.env.local
NEXT_PUBLIC_STRAPI_API_URL=https://parks-strapi-api.onrender.com
NEXT_PUBLIC_STRAPI_API_TOKEN=<APIトークン>
```

---

### 機能実装のためのAPI活用レシピ

#### 🗺️ レシピ1: 地図上に全ての公園のピンを立てる

初期表示で、全ての公園の位置情報を取得し、地図上にマーカーとしてプロットします。

- **目的:** 公園の基本情報（特に緯度・経度）を一覧で取得する。
- **エンドポイント:** `GET /api/parks`
- **URL:** `https://parks-strapi-api.onrender.com/api/parks`
- **ワンポイント:**
  データ量を最適化するため、必要なフィールドだけを指定して取得するとパフォーマンスが向上します。
  ```javascript
  // 例: ID, 公園名, 緯度, 経度のみ取得
  const fields = ['park_name', 'lat', 'lon'];
  const query = qs.stringify({ fields }); // qsライブラリ等でクエリを生成
  const url = `${process.env.NEXT_PUBLIC_STRAPI_API_URL}/api/parks?${query}`;
  ```

- **レスポンスの構造 (抜粋):**
  ```json
  {
    "data": [
      {
        "id": 10020,
        "attributes": {
          "park_name": "あい公園",
          "lat": 34.74,
          "lon": 137.66,
          // ... 他の基本情報 ...
        }
      },
      // ... 他の公園データ ...
    ],
    "meta": { /* ページネーション情報 */ }
  }
  ```
  `data` 配列をマップして、各オブジェクトの `id`, `attributes.lat`, `attributes.lon` を使ってピンをプロットします。

---

#### 🎨 レシピ2: 公園のピンの色を条件によって変更する

「各公園の最も危険な遊具のハザードレベル」に応じてピンの色を変えるなど、よりリッチな情報を提供します。
これには、公園情報と関連する遊具情報を一緒に取得する必要があります。

- **目的:** 公園情報と、それに紐づく全ての遊具ハザードレベル情報を一度に取得する。
- **エンドポイント:** `GET /api/parks?populate=park_hazard_levels`
  - `populate` クエリパラメータが、リレーション先のデータを一緒に取得するための鍵です。
- **URL:** `https://parks-strapi-api.onrender.com/api/parks?populate=park_hazard_levels`

- **レスポンスの構造 (抜粋):**
  ```json
  {
    "data": [
      {
        "id": 10020,
        "attributes": {
          "park_name": "あい公園",
          "lat": 34.74,
          "lon": 137.66,
          "park_hazard_levels": {
            "data": [
              {
                "id": 1,
                "attributes": {
                  "equipment_name": "ブランコ",
                  "hazard_level": 2,
                  "note": "修繕が必要です。"
                }
              },
              {
                "id": 2,
                "attributes": {
                  "equipment_name": "すべり台",
                  "hazard_level": 3,
                  "note": "早急に修繕が必要です。"
                }
              }
            ]
          }
        }
      }
    ]
  }
  ```

- **フロントエンドでの実装例:**
  ```javascript
  // この公園のピンの色を決定するロジック
  const parkData = response.data.attributes;
  const hazardLevels = parkData.park_hazard_levels.data.map(
    item => item.attributes.hazard_level
  );
  
  // 最も高いハザードレベルを取得
  const maxHazardLevel = Math.max(...hazardLevels); 

  // maxHazardLevelに応じてピンの色を返す
  if (maxHazardLevel >= 3) return 'red';
  if (maxHazardLevel === 2) return 'orange';
  return 'green';
  ```

---

#### 🖱️ レシピ3: ピンをクリックして公園と遊具の詳細情報を表示する

ユーザーが特定のピンをクリックした際に、その公園の詳細情報を取得してモーダルウィンドウなどで表示します。

- **目的:** 特定の`id`を持つ公園とその遊具情報を取得する。
- **エンドポイント:** `GET /api/parks/:id?populate=park_hazard_levels`
  - `:id` 部分には、クリックされた公園の`id`を動的に設定します。
- **URL例:** `https://parks-strapi-api.onrender.com/api/parks/10020?populate=park_hazard_levels`
  - このエンドポイントは、SWRやReact Queryの `useSWR('/api/parks/' + parkId + '?populate=...')` のような形で使うと、キャッシュも効いて効果的です。

- **レスポンスの構造:** レシピ2のレスポンスの `data` 配列が、単一のオブジェクトになった形です。
  ```json
  {
    "data": {
      "id": 10020,
      "attributes": {
        // ...公園情報と、紐づくpark_hazard_levels配列...
      }
    },
    "meta": {}
  }
  ```

---

### 高度な使い方: フィルタリング

特定の条件に合致するデータのみを取得したい場合、強力なフィルタリング機能が使えます。
詳細は[公式ドキュメント](https://docs.strapi.io/dev-docs/api/rest/filters-locale-publication)を参照してください。

- **例1: 特定の文字列が`note`に含まれる遊具を検索する**
  ```
  /api/parks-hazard-levels?filters[note][$contains]=修繕が必要
  ```

- **例2: ハザードレベルが3以上の遊具を持つ公園を検索する (ネストしたフィルタ)**
  ```
  /api/parks?filters[park_hazard_levels][hazard_level][$gte]=3
  ```

---

## バックエンドの運用・管理

### デプロイについて

このプロジェクトはRenderでホスティングされており、`main` ブランチへのプッシュをトリガーとして**自動デプロイ**が実行されます。

1. ローカルで変更を加え、コミットします。
2. `main` ブランチにプッシュすると、自動で本番環境に反映されます。

### 管理画面

コンテンツの追加や編集は、Strapiの管理画面から行います。
- **本番環境 管理画面URL:** `https://parks-strapi-api.onrender.com/admin`

---

## ローカル開発環境のセットアップ

バックエンドのコードを修正する場合は、以下の手順でローカル環境を構築します。

1.  **リポジトリをクローン:** `git clone https://github.com/u-yamii/park_strapi.git`
2.  **依存関係をインストール:** `npm install` (または `yarn`)
3.  **`.env` ファイルを作成:** 別途共有される情報を元に、データベース接続情報等を設定します。
4.  **開発サーバーを起動:** `npm run develop` (または `yarn develop`)

---

### データ構造について

APIレスポンスの正確な構造（フィールド名、型など）は、`schema.json`ファイルで定義されています。

- **公園:** `src/api/park/content-types/park/schema.json`
- **遊具ハザードレベル:** `src/api/parks-hazard-level/content-types/parks-hazard-level/schema.json`