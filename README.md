# db-handler-server

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

### 設計原則
1. **契約ファースト**: インターフェースを先に定義し、実装は後
2. **単一責任**: データアクセスのみに責任を持つ
3. **技術非依存**: 上位層は具体的なDB技術を知らない

### 提供すべき抽象化レベル
- ビジネスエンティティレベルでの操作
- CRUD以上の意味のある操作
- ドメイン固有の検索・集約機能

---

## handlers-repo: ビジネスロジックの実装者

### アーキテクチャ上の役割
- **アプリケーションサービス**: ビジネスフローの制御
- **プレゼンテーション層**: Fiber HTTP APIの提供
- **ビジネスルール**: ドメインロジックの実装

### 設計原則
1. **依存性逆転**: インターフェースに依存、実装に依存しない
2. **責任分離**: プレゼンテーション層とビジネス層を分離
3. **疎結合**: データ層の実装詳細を知らない

### 実装すべき層
- **ハンドラー層**: Fiber HTTPリクエスト/レスポンス処理
- **サービス層**: ビジネスロジックの実装
- **ミドルウェア層**: Fiber middleware（認証、ログ、CORS等）

### Fiber実装のポイント
- `fiber.Ctx`を使ったリクエスト処理
- `fiber.Router`でのルーティング設計
- `fiber.Map`でのJSONレスポンス
- エラーハンドリングの統一

---

## server-repo: 統合の責任者

### アーキテクチャ上の役割
- **Composition Root**: 依存性の解決と注入
- **Fiber アプリケーション起動**: サーバー設定と初期化
- **環境設定**: 設定値の管理

### 設計原則
1. **依存性注入**: 実装の組み合わせを決定
2. **設定の外部化**: 環境に応じた設定管理
3. **起動の責任**: Fiber アプリケーションライフサイクル管理

### Fiber実装のポイント
- `fiber.New()`でのアプリケーション初期化
- ミドルウェアの設定
- ルートの登録
- サーバー起動設定

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
1. **Database-repo**: インターフェース定義から開始
2. **Handlers-repo**: インターフェースを使ったビジネスロジック
3. **Server-repo**: 依存性の組み立て

### テスト戦略  
- **Database-repo**: 実装のテスト
- **Handlers-repo**: モックを使った単体テスト
- **Server-repo**: 統合テスト

### 技術選択の自由度
各リポジトリで独立して技術選択可能：
- Database: GORM, MongoDB, Redis等
- **Handler & Server: Fiber（統一実装）**

### Fiberの特徴（選定理由）
- Express.js風の直感的API
- Go最速クラスのパフォーマンス
- 豊富なミドルウェア
- WebSocket等の機能が充実
- handlers-repo と server-repo で統一された開発体験

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
1. Database-repo: インターフェース定義
2. Database-repo: 実装追加  
3. Handlers-repo: ビジネスロジック実装
4. Server-repo: 依存性追加

### 別のDB技術への移行
Database-repoの実装のみ変更、他は影響なし

### API version管理
Handlers-repoで複数バージョン対応、DB層は共通利用
