# DB-Repo側インターフェース定義 アーキテクチャ仕様書

## 設計思想

**データ中心設計**: Database-repoがドメインの契約を定義し、他の層がそれに従う

## アーキテクチャパターン

```
server-repo (Composition Root) 
    ↓
handlers-repo (Application Layer) 
    ↓
database-repo (Data Contract + Infrastructure)
```

---

## database-repo: データ契約の定義者

### アーキテクチャ上の役割
- **契約の決定者**: インターフェースとデータ構造を定義
- **データの所有者**: エンティティの正規形を決定
- **技術的詳細の隠蔽**: 具体的なDB実装を抽象化
- **スキーマ管理**: マイグレーション・シーディングの責任

### 設計原則
1. **契約ファースト**: インターフェースを先に定義し、実装は後
2. **単一責任**: データアクセスとスキーマ管理のみに責任を持つ
3. **技術非依存**: 上位層は具体的なDB技術を知らない
4. **バージョン管理**: スキーマ変更の履歴管理

### 提供すべき抽象化レベル
- ビジネスエンティティレベルでの操作
- CRUD以上の意味のある操作
- ドメイン固有の検索・集約機能
- **マイグレーション・ロールバック機能**
- **テストデータ・本番初期データの投入**

### Migration管理の責任
- **スキーマ定義**: テーブル・インデックス設計
- **golang-migrate統合**: CLI・ライブラリ両方の活用
- **マイグレーション実行**: Up/Down migration
- **データ投入**: Seeding・初期データ管理
- **環境別対応**: Dev/Test/Prod環境の差異管理
- **バージョン追跡**: マイグレーション履歴の管理

---

## handlers-repo: ビジネスロジックの実装者

### アーキテクチャ上の役割
- **アプリケーションサービス**: ビジネスフローの制御
- **プレゼンテーション層**: Fiber HTTP APIの提供
- **ビジネスルール**: ドメインロジックの実装
- **API仕様管理**: OpenAPI/Swagger仕様の定義

### 設計原則
1. **依存性逆転**: インターフェースに依存、実装に依存しない
2. **責任分離**: プレゼンテーション層とビジネス層を分離
3. **疎結合**: データ層の実装詳細を知らない
4. **API First**: OpenAPIを先に定義、実装はそれに従う

### 実装すべき層
- **ハンドラー層**: Fiber HTTPリクエスト/レスポンス処理
- **サービス層**: ビジネスロジックの実装
- **ミドルウェア層**: Fiber middleware（認証、ログ、CORS等）
- **API仕様層**: OpenAPI 3.0定義・Swaggerドキュメント

### Fiber実装のポイント
- `fiber.Ctx`を使ったリクエスト処理
- `fiber.Router`でのルーティング設計
- `fiber.Map`でのJSONレスポンス
- エラーハンドリングの統一
- **Swagger統合**: API仕様の自動生成・提供

---

## server-repo: 統合の責任者

### アーキテクチャ上の役割
- **Composition Root**: 依存性の解決と注入
- **Fiber アプリケーション起動**: サーバー設定と初期化
- **環境設定**: 設定値の管理
- **API仕様提供**: Swagger UIの公開

### 設計原則
1. **依存性注入**: 実装の組み合わせを決定
2. **設定の外部化**: 環境に応じた設定管理
3. **起動の責任**: Fiber アプリケーションライフサイクル管理
4. **API可視化**: 開発・テスト用のAPI仕様提供

### Fiber実装のポイント
- `fiber.New()`でのアプリケーション初期化
- ミドルウェアの設定
- ルートの登録
- サーバー起動設定
- **Swagger UI**: `/docs`エンドポイントでAPI仕様提供

---

## 層間の契約

### Interface定義の所有権
**Database-repo**がインターフェースを定義する理由：
- データの整合性を保証
- スキーマ変更の影響を局所化
- データアクセスパターンの標準化

### 依存の方向性
```
server-repo 
  ├─ handlers-repo (具象)
  └─ database-repo (具象)

handlers-repo
  └─ database-repo/interfaces (抽象)
```

### 変更の影響範囲
- **DB schema変更**: database-repoのみ
- **ビジネスロジック変更**: handlers-repoのみ  
- **API変更**: handlers-repo + server-repo
- **インターフェース変更**: 全体に影響（慎重に）

---

## 実装指針

### 開発順序
1. **Database-repo**: インターフェース定義 → マイグレーション → 実装
2. **Handlers-repo**: インターフェースを使ったビジネスロジック
3. **Server-repo**: 依存性の組み立て → マイグレーション実行

### マイグレーション戦略
- **手動実行**: CLI経由での手動マイグレーション
- **CI/CD統合**: デプロイパイプラインでのmigration実行
- **ロールバック対応**: Down migrationによる安全な巻き戻し

### Migration開発フロー
1. `migrate create -ext sql -dir migrations -seq [name]`でファイル作成
2. up.sql/down.sqlでスキーマ定義
3. 開発環境で`migrate up`テスト
4. CI/CDパイプラインに統合

### テスト戦略  
- **Database-repo**: 実装のテスト
- **Handlers-repo**: モックを使った単体テスト
- **Server-repo**: 統合テスト

### 技術選択の自由度
各リポジトリで独立して技術選択可能：
- **Database: GORM + golang-migrate（推奨構成）**
- **Handler & Server: Fiber + OpenAPI/Swagger（統一実装）**

### Database技術スタック
- **ORM**: GORM（Go最も人気のORM）
- **Migration**: golang-migrate（業界標準）
- **DB**: MySQL, PostgreSQL等

### API技術スタック
- **Framework**: Fiber（高性能・直感的API）
- **API仕様**: OpenAPI 3.0 / Swagger
- **ドキュメント**: Swagger UI統合
- **バリデーション**: APIスキーマ検証

### golang-migrateの特徴
- CLIとライブラリ両方対応
- Up/Down migration完全対応
- 豊富なDB driver対応
- バージョン管理機能
- CI/CD統合が容易

### OpenAPI統合の特徴
- API設計の標準化
- 自動ドキュメント生成
- クライアントSDK生成可能
- テスト自動化対応

---

## 適用基準

### このパターンが適している場合
- データ構造が比較的安定
- チーム規模が中程度
- DB中心の設計思想

### 注意すべき点
- Handler側がDB-repoに依存する循環
- インターフェース変更の影響範囲
- 過度に複雑なインターフェース設計

---

## 拡張性の考慮

### 新しいエンティティ追加
1. Database-repo: インターフェース定義 → マイグレーション作成
2. Database-repo: 実装追加  
3. Handlers-repo: ビジネスロジック実装
4. Server-repo: 依存性追加

### 別のDB技術への移行
Database-repoの実装・マイグレーションのみ変更、他は影響なし

### スキーマ変更管理
1. Database-repo: `golang-migrate`でマイグレーションファイル作成
2. インターフェース更新（必要に応じて）
3. 段階的デプロイ・`migrate down`でのロールバック対応

### Fiberの特徴（選定理由）
- Express.js風の直感的API
- Go最速クラスのパフォーマンス
- 豊富なミドルウェア
- WebSocket等の機能が充実
- handlers-repo と server-repo で統一された開発体験
