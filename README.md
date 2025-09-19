## 開発・運用フロー

### 開発順序
1. **Database-repo**: Protocol Buffers定義 → Migration → gRPCサーバー実装
2. **Handlers-repo**: Protocol Buffers定義 → gRPCサーバー実装 + gRPCクライアント統合
3. **Server-repo**: 依存性注入 → Gateway設定 → プロトコル変換

### 運用モード切り替え

### 運用モード切り替え

#### 環境変数による動作モード制御
```go
// server-repo/main.go
func main() {
    mode := os.Getenv("DEPLOYMENT_MODE")
    if mode == "" {
        mode = "single"  // デフォルトは単一プロセス
    }
    
    switch mode {
    case "single":
        runSingleProcess()
    case "separate":
        runSeparateProcesses()
    default:
        log.Fatal("Invalid DEPLOYMENT_MODE. Use 'single' or 'separate'")
    }
}
```

#### 単一プロセスモード（本番・デフォルト）
```bash
# 環境変数設定
export DEPLOYMENT_MODE=single
# または環境変数なし（デフォルト）

cd server-repo && go run main.go
```

```go
func runSingleProcess() {
    log.Println("Starting in SINGLE PROCESS mode")
    
    // 1. Database gRPCサーバー起動（インメモリ）
    dbService := database.NewGRPCService(mysqlConnection)
    
    // 2. Database gRPCクライアント作成（bufconn使用）
    dbClient := createLocalGRPCClient(dbService)
    
    // 3. Handlers gRPCサーバー起動（dependency injection）
    handlerService := handlers.NewGRPCService(dbClient)
    
    // 4. Multi-Protocol Gateway設定
    gateway := setupGateway(handlerService)
    
    // 5. 単一ポート起動
    app := fiber.New()
    app.Mount("/", gateway)
    log.Println("Single process server running on :8080")
    app.Listen(":8080")
}
```

#### 分離プロセスモード（開発・テスト用）
```bash
# 環境変数設定
export DEPLOYMENT_MODE=separate
export DATABASE_GRPC_URL=localhost:50051    # 必須
export HANDLERS_GRPC_URL=localhost:50052    # 必須

cd server-repo && go run main.go
```

```go
func runSeparateProcesses() {
    log.Println("Starting in SEPARATE PROCESSES mode")
    
    // 必須環境変数チェック
    dbURL := os.Getenv("DATABASE_GRPC_URL")
    if dbURL == "" {
        log.Fatal("DATABASE_GRPC_URL is required in separate mode")
    }
    
    handlerURL := os.Getenv("HANDLERS_GRPC_URL")
    if handlerURL == "" {
        log.Fatal("HANDLERS_GRPC_URL is required in separate mode")
    }
    
    // 外部gRPCサーバーへのネットワーク接続
    handlerClient := createRemoteGRPCClient(handlerURL)
    
    // Gateway設定
    gateway := setupGateway(handlerClient)
    
    app := fiber.New()
    app.Mount("/", gateway)
    log.Printf("Gateway server running on :8080")
    log.Printf("Connecting to handlers gRPC: %s", handlerURL)
    app.Listen(":8080")
}
```

#### 環境変数設定例

**本番環境**
```bash
export DEPLOYMENT_MODE=single
export DATABASE_URL="user:pass@tcp(localhost:3306)/prod_db"
```

**開発環境**
```bash
export DEPLOYMENT_MODE=separate
export DATABASE_GRPC_URL=localhost:50051
export HANDLERS_GRPC_URL=localhost:50052
export DATABASE_URL="user:pass@tcp(localhost:3306)/dev_db"
```

**Docker Compose環境**
```bash
export DEPLOYMENT_MODE=separate
export DATABASE_GRPC_URL=database-grpc:50051
export HANDLERS_GRPC_URL=handlers-grpc:50052
```
```

### 段階的開発・テスト戦略

#### Phase 1: Database層単体開発（分離モード）
```bash
# 1. Protocol Buffers定義・コード生成
cd database-repo
protoc --go_out=. --go-grpc_out=. proto/database.proto

# 2. 開発用gRPCサーバー起動（分離テスト用）
go run server/main.go  # :50051で独立起動

# 3. 単体テスト
go test ./...

# 4. 手動テスト（grpcurl使用）
grpcurl -d '{"id": 1}' -plaintext localhost:50051 DatabaseService/GetUserByID
```

#### Phase 2: Handlers層統合開発（分離モード）
```bash
# 1. Database gRPCサーバーを別プロセスで起動
cd database-repo && go run server/main.go &  # :50051

# 2. Handlers開発・テスト（ネットワーク経由でDatabase呼び出し）
cd handlers-repo
export DATABASE_GRPC_URL=localhost:50051
protoc --go_out=. --go-grpc_out=. proto/user.proto

# 3. 統合テスト（分離モード）
go run server/main.go  # :50052で起動、:50051のDBを呼び出し

# 4. Handlers APIテスト
grpcurl -d '{"id": 1}' -plaintext localhost:50052 UserService/GetUser
```

#### Phase 3: Gateway統合・プロトコル変換（モード切り替え）
```bash
# 開発・テスト用（分離モード）
export DEPLOYMENT_MODE=separate
export DATABASE_GRPC_URL=localhost:50051
export HANDLERS_GRPC_URL=localhost:50052
cd server-repo && go run main.go  # :8080で全プロトコル提供

# 本番用（単一プロセスモード）
export DEPLOYMENT_MODE=single
cd server-repo && go run main.go  # 全てが単一プロセス内で動作

# 各プロトコルテスト（両モード共通）
curl http://localhost:8080/api/v1/users/1        # REST API
curl http://localhost:8080/docs                  # Swagger UI
```

### 開発用サーバー実装（分離テスト専用）

#### Database-repo開発サーバー（テスト用分離サーバー）
```go
// database-repo/server/main.go
// ※# gRPC-First 単一プロセス マルチプロトコル アーキテクチャ仕様書

## 設計思想

**gRPC-First + 単一プロセス設計**: gRPCの型安全性とマイクロサービス移行容易性を保ちつつ、単一プロセスでの運用を実現

## アーキテクチャパターン

```
単一プロセス (server-repo)
├── Multi-Protocol Gateway (REST/gRPC-Web/JSON-RPC2)
├── handlers-repo (gRPCサーバー + gRPCクライアント)
└── database-repo (gRPCサーバー)
```

### gRPC通信フロー
```
外部クライアント → server-repo → handlers-repo → database-repo → MySQL
                   ↑            ↑              ↑
                Gateway      gRPCサーバー    gRPCサーバー
                           +gRPCクライアント
```

---

## database-repo: データアクセス gRPCサーバー

### アーキテクチャ上の役割
- **gRPCサーバー**: データアクセス層のgRPCサービス提供
- **データ契約定義**: Protocol Buffersでのデータ構造定義
- **技術的詳細の隠蔽**: 具体的なDB実装を抽象化
- **スキーマ管理**: マイグレーション・シーディングの責任

### 設計原則
1. **gRPC-First**: Protocol Buffersでの契約定義
2. **単一責任**: データアクセスとスキーマ管理のみ
3. **技術非依存**: 上位層は具体的なDB技術を知らない
4. **内部サービス**: handlers-repoからのみアクセス

### 提供するgRPCサービス
```proto
service DatabaseService {
  // User関連
  rpc GetUserByID(GetUserByIDRequest) returns (GetUserByIDResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  
  // 他のエンティティも同様...
}
```

### Migration管理の責任
- **golang-migrate統合**: CLI・ライブラリ両方の活用
- **スキーマ定義**: Protocol Buffersとデータベーススキーマの整合性
- **マイグレーション実行**: Up/Down migration
- **環境別対応**: Dev/Test/Prod環境の差異管理

### 実装のポイント
- GORMでのデータアクセス実装
- Protocol Buffers ↔ GORM構造体の変換
- 単一プロセス内でのgRPCサーバー起動
- トランザクション管理の提供

---

## handlers-repo: ビジネスロジック gRPCサーバー + クライアント

### アーキテクチャ上の役割
- **gRPCサーバー**: 外部向けビジネスロジックAPIの提供
- **gRPCクライアント**: database-repo gRPCサービスの呼び出し
- **ビジネスロジック実装**: ドメインルールの実装
- **データ変換**: 外部API用データ形式への変換

### 設計原則
1. **二重役割**: サーバーとクライアントの両方を実装
2. **ビジネス中心**: ドメインロジックに集中
3. **疎結合**: database-repoをgRPCクライアント経由で利用
4. **型安全性**: Protocol Buffersによる厳密な型管理

### 提供するgRPCサービス（外部向け）
```proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}
```

### database-repo呼び出し（内部向け）
```go
type UserService struct {
    dbClient database.DatabaseServiceClient // gRPCクライアント
}

func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    // database-repoへgRPC呼び出し
    dbResp, err := s.dbClient.GetUserByID(ctx, &database.GetUserByIDRequest{
        ID: req.ID,
    })
    
    // ビジネスロジック処理
    user := s.transformUser(dbResp.User)
    
    return &pb.GetUserResponse{User: user}, nil
}
```

### 実装のポイント
- 外部向けgRPCサーバーの実装
- database-repo gRPCクライアントの管理
- ビジネスロジックの実装
- データ変換・バリデーション

---

## server-repo: マルチプロトコル Gateway + 依存性注入

### アーキテクチャ上の役割
- **Composition Root**: 全ての依存性解決と注入
- **Multi-Protocol Gateway**: gRPCから複数プロトコルへの自動変換
- **単一プロセス管理**: 全gRPCサーバーの統合管理
- **API仕様提供**: OpenAPI・Swagger UIの公開

### 設計原則
1. **単一エントリーポイント**: 1つのプロセス・1つのポート
2. **プロトコル変換**: gRPCから複数プロトコルへの自動変換
3. **依存性統合**: 全リポジトリの依存性を解決
4. **運用の簡素化**: モノリス同様の運用体験

### 提供プロトコル
- **gRPC**: 高性能バイナリプロトコル
- **REST API**: grpc-gatewayによる自動変換
- **gRPC-Web**: ブラウザからの直接gRPC呼び出し
- **JSON-RPC2**: 軽量JSONベースプロトコル
- **OpenAPI**: REST API仕様の自動生成・Swagger UI

### 単一プロセス起動フロー
```go
func main() {
    // 1. Database gRPCサーバー起動（インメモリ）
    dbService := database.NewGRPCService(mysqlConnection)
    
    // 2. Database gRPCクライアント作成（bufconn使用）
    dbClient := createLocalGRPCClient(dbService)
    
    // 3. Handlers gRPCサーバー起動（dependency injection）
    handlerService := handlers.NewGRPCService(dbClient)
    
    // 4. Multi-Protocol Gateway設定
    gateway := setupGateway(handlerService)
    
    // 5. 単一ポート起動
    app := fiber.New()
    app.Mount("/", gateway)
    app.Listen(":8080")
}
```

### インメモリgRPC通信
```go
func createLocalGRPCClient(service database.DatabaseService) database.DatabaseServiceClient {
    // bufconnでネットワークを使わないローカル通信
    lis := bufconn.Listen(bufferSize)
    s := grpc.NewServer()
    database.RegisterDatabaseServiceServer(s, service)
    go s.Serve(lis)
    
    conn, _ := grpc.Dial("bufnet", 
        grpc.WithContextDialer(bufconnDialer(lis)),
        grpc.WithInsecure())
    
    return database.NewDatabaseServiceClient(conn)
}
```

---

## 技術スタック

### Database技術スタック
- **ORM**: GORM（データアクセス実装）
- **Migration**: golang-migrate（スキーマ管理）
- **DB**: MySQL, PostgreSQL等
- **gRPC**: Protocol Buffers + gRPC-Go

### API技術スタック
- **gRPC**: Protocol Buffers中心設計
- **Gateway**: grpc-gateway（gRPC → REST変換）
- **Web**: gRPC-Web（ブラウザ対応）
- **RPC**: カスタムJSON-RPC2実装
- **Documentation**: OpenAPI 3.0 + Swagger UI

### 通信技術スタック
- **プロセス内通信**: bufconn（メモリ内gRPC）
- **プロセス間通信**: TCP gRPC（将来のマイクロサービス分離用）
- **HTTP Gateway**: Fiber（マルチプロトコル提供）

### 自動生成ツール
- **protoc**: Protocol Buffersコンパイラ
- **protoc-gen-go**: Go構造体生成
- **protoc-gen-go-grpc**: gRPCサービス生成
- **protoc-gen-grpc-gateway**: REST Gateway生成
- **protoc-gen-openapiv2**: OpenAPI仕様生成

---

## 開発・運用フロー

### 開発順序
1. **Database-repo**: Protocol Buffers定義 → Migration → gRPCサーバー実装
2. **Handlers-repo**: Protocol Buffers定義 → gRPCサーバー実装 + gRPCクライアント統合
3. **Server-repo**: 依存性注入 → Gateway設定 → プロトコル変換

### 段階的開発・テスト戦略

#### Phase 1: Database層単体開発
```bash
# 1. Protocol Buffers定義・コード生成
cd database-repo
protoc --go_out=. --go-grpc_out=. proto/database.proto

# 2. 開発用gRPCサーバー起動
go run server/main.go  # :50051で起動

# 3. 単体テスト
go test ./...

# 4. 手動テスト（grpcurl使用）
grpcurl -d '{"id": 1}' -plaintext localhost:50051 DatabaseService/GetUserByID
```

#### Phase 2: Handlers層統合開発
```bash
# 1. Database gRPCサーバーを別プロセスで起動
cd database-repo && go run server/main.go &  # :50051

# 2. Handlers開発・テスト
cd handlers-repo
export DATABASE_GRPC_URL=localhost:50051
protoc --go_out=. --go-grpc_out=. proto/user.proto

# 3. 統合テスト
go run server/main.go  # :50052で起動、:50051のDBを呼び出し

# 4. Handlers APIテスト
grpcurl -d '{"id": 1}' -plaintext localhost:50052 UserService/GetUser
```

#### Phase 3: Gateway統合・プロトコル変換
```bash
# 1. 全層統合
cd server-repo
go run main.go  # :8080で全プロトコル提供

# 2. 各プロトコルテスト
curl http://localhost:8080/api/v1/users/1        # REST API
curl http://localhost:8080/docs                  # Swagger UI
# gRPC-Web、JSON-RPC2テスト等
```

### 開発用サーバー実装

#### Database-repo開発サーバー
```go
// database-repo/server/main.go
func main() {
    // MySQL接続
    db := setupDatabase()
    defer db.Close()
    
    // gRPCサーバー起動
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }
    
    s := grpc.NewServer()
    dbService := NewDatabaseService(db)
    RegisterDatabaseServiceServer(s, dbService)
    
    log.Println("Database gRPC server running on :50051")
    s.Serve(lis)
}
```

#### Handlers-repo開発サーバー
```go
// handlers-repo/server/main.go  
func main() {
    // Database gRPCクライアント接続
    dbURL := os.Getenv("DATABASE_GRPC_URL")
    if dbURL == "" {
        dbURL = "localhost:50051"
    }
    
    dbConn, err := grpc.Dial(dbURL, grpc.WithInsecure())
    if err != nil {
        log.Fatal(err)
    }
    defer dbConn.Close()
    
    dbClient := database.NewDatabaseServiceClient(dbConn)
    
    // Handlers gRPCサーバー起動
    lis, err := net.Listen("tcp", ":50052")
    if err != nil {
        log.Fatal(err)
    }
    
    s := grpc.NewServer()
    userService := NewUserService(dbClient)
    RegisterUserServiceServer(s, userService)
    
    log.Println("Handlers gRPC server running on :50052")
    log.Printf("Database gRPC client connecting to: %s", dbURL)
    s.Serve(lis)
}
```

### 開発用Docker Compose（分離モード専用）
```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: testdb

  database-grpc:
    build: 
      context: ./database-repo
      dockerfile: Dockerfile.dev
    ports:
      - "50051:50051"
    depends_on:
      - mysql
    environment:
      DATABASE_URL: "root:password@tcp(mysql:3306)/testdb"

  handlers-grpc:
    build:
      context: ./handlers-repo  
      dockerfile: Dockerfile.dev
    ports:
      - "50052:50052"
    depends_on:
      - database-grpc
    environment:
      DATABASE_GRPC_URL: "database-grpc:50051"

  gateway:
    build:
      context: ./server-repo
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    depends_on:
      - handlers-grpc
    environment:
      DEPLOYMENT_MODE: "separate"
      DATABASE_GRPC_URL: "database-grpc:50051"
      HANDLERS_GRPC_URL: "handlers-grpc:50052"
```

### 本番用Docker Compose（単一プロセスモード）
```yaml
# docker-compose.prod.yml  
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: proddb

  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
    ports:
      - "8080:8080"
    depends_on:
      - mysql
    environment:
      DEPLOYMENT_MODE: "single"  # 単一プロセスモード
      DATABASE_URL: "root:password@tcp(mysql:3306)/proddb"
```

### Protocol Buffers開発フロー
```bash
# 1. .protoファイル作成
database-repo/proto/database.proto
handlers-repo/proto/user.proto

# 2. 各層でコード生成
cd database-repo
protoc --go_out=. --go-grpc_out=. proto/database.proto

cd handlers-repo  
protoc --go_out=. --go-grpc_out=. proto/user.proto

cd server-repo
protoc --go_out=. --go-grpc_out=. --grpc-gateway_out=. proto/user.proto

# 3. 実装
# 4. 段階的テスト
```

### Migration戦略
- **手動実行**: CLI経由での手動マイグレーション
- **CI/CD統合**: デプロイパイプラインでのmigration実行
- **ロールバック対応**: Down migrationによる安全な巻き戻し

### デプロイ戦略
- **単一バイナリ**: すべてのリポジトリを統合
- **設定ファイル**: 環境別設定の外部化
- **ヘルスチェック**: gRPCヘルスチェックプロトコル対応

---

## 層間通信フロー

### リクエストフロー
```
1. 外部クライアント → server-repo (REST/gRPC-Web/JSON-RPC2)
2. server-repo → handlers-repo (gRPC, インメモリ)
3. handlers-repo → database-repo (gRPC, インメモリ)
4. database-repo → MySQL (SQL)
5. 逆順でレスポンス返却
```

### エラーハンドリングフロー
```
database-repo: gRPC Status Code返却
     ↓
handlers-repo: ビジネスエラー変換
     ↓
server-repo: プロトコル別エラー変換 (HTTP Status, JSON-RPC Error等)
```

### トランザクション管理
- 単一プロセス内のため従来通りのトランザクション管理
- database-repo層でトランザクション制御
- handlers-repo層からトランザクション境界指定

---

## テスト戦略

### 各層のテスト
- **Database-repo**: gRPCサーバーテスト・データアクセステスト
- **Handlers-repo**: gRPCサーバーテスト・モックgRPCクライアントテスト  
- **Server-repo**: 統合テスト・プロトコル変換テスト・E2Eテスト

### 段階的テスト手法

#### 1. Database層単体テスト
```go
// database-repo/service_test.go
func TestDatabaseService_GetUserByID(t *testing.T) {
    // テスト用DB接続
    db := setupTestDB()
    defer db.Close()
    
    service := NewDatabaseService(db)
    
    resp, err := service.GetUserByID(context.Background(), &GetUserByIDRequest{
        ID: 1,
    })
    
    assert.NoError(t, err)
    assert.Equal(t, int32(1), resp.User.ID)
}
```

#### 2. Handlers層テスト（モッククライアント）
```go
// handlers-repo/service_test.go
func TestUserService_GetUser(t *testing.T) {
    // モックDatabase gRPCクライアント
    mockDBClient := &MockDatabaseServiceClient{}
    mockDBClient.On("GetUserByID", mock.Anything, mock.Anything).Return(
        &database.GetUserByIDResponse{User: &database.User{ID: 1}}, nil)
    
    userService := &UserService{dbClient: mockDBClient}
    
    resp, err := userService.GetUser(context.Background(), &GetUserRequest{ID: 1})
    
    assert.NoError(t, err)
    assert.Equal(t, int32(1), resp.User.ID)
}
```

#### 3. 統合テスト（実際のgRPCサーバー使用）
```go
// handlers-repo/integration_test.go
func TestUserService_Integration(t *testing.T) {
    // Database gRPCサーバー起動
    dbServer := startTestDatabaseServer(t)
    defer dbServer.Stop()
    
    // Database gRPCクライアント作成
    conn, err := grpc.Dial("localhost:50051", grpc.WithInsecure())
    require.NoError(t, err)
    defer conn.Close()
    
    dbClient := database.NewDatabaseServiceClient(conn)
    userService := NewUserService(dbClient)
    
    // テスト実行
    resp, err := userService.GetUser(context.Background(), &GetUserRequest{ID: 1})
    assert.NoError(t, err)
}
```

### 開発用テストツール

#### 1. grpcurl（コマンドライン）
```bash
# インストール
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# サービス一覧表示
grpcurl -plaintext localhost:50051 list

# メソッド呼び出し
grpcurl -d '{"id": 1}' -plaintext localhost:50051 DatabaseService/GetUserByID
grpcurl -d '{"id": 1}' -plaintext localhost:50052 UserService/GetUser

# サービス詳細表示
grpcurl -plaintext localhost:50051 describe DatabaseService
```

#### 2. Evans（インタラクティブgRPCクライアント）
```bash
# インストール
go install github.com/ktr0731/evans@latest

# 接続・使用例
evans --host localhost --port 50051 --package database --service DatabaseService

# インタラクティブモード
> call GetUserByID
> {"id": 1}
> 
> show service
> show message User
```

#### 3. Postman（GUI）
- gRPC対応（v9.12以降）
- Protocol Buffersファイルインポート
- リクエスト履歴・コレクション管理

### CI/CD統合テスト
```yaml
# .github/workflows/test.yml
name: gRPC Integration Test
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.21'
      
      - name: Install grpcurl
        run: go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
      
      # Database層テスト
      - name: Test Database gRPC
        run: |
          cd database-repo
          go test ./...
          
          # 開発サーバー起動
          go run server/main.go &
          sleep 5
          
          # gRPCサーバーテスト
          grpcurl -plaintext localhost:50051 list
          grpcurl -d '{"id": 1}' -plaintext localhost:50051 DatabaseService/GetUserByID
        env:
          DATABASE_URL: "root:password@tcp(localhost:3306)/testdb"
      
      # Handlers層統合テスト  
      - name: Test Handlers gRPC Integration
        run: |
          cd handlers-repo
          go test ./...
          
          # 統合テスト用サーバー起動
          export DATABASE_GRPC_URL=localhost:50051
          go run server/main.go &
          sleep 5
          
          # 統合テスト実行
          grpcurl -plaintext localhost:50052 list
          grpcurl -d '{"id": 1}' -plaintext localhost:50052 UserService/GetUser
      
      # Gateway統合テスト
      - name: Test Gateway Integration
        run: |
          cd server-repo
          export HANDLERS_GRPC_URL=localhost:50052
          go run main.go &
          sleep 5
          
          # 各プロトコルテスト
          curl -f http://localhost:8080/api/v1/users/1  # REST
          curl -f http://localhost:8080/docs            # Swagger UI
```

### 負荷テスト
```bash
# ghzを使ったgRPC負荷テスト
go install github.com/bojand/ghz/cmd/ghz@latest

# Database層負荷テスト
ghz --insecure \
    --proto database-repo/proto/database.proto \
    --call DatabaseService.GetUserByID \
    -d '{"id": 1}' \
    -c 50 -n 1000 \
    localhost:50051

# Handlers層負荷テスト  
ghz --insecure \
    --proto handlers-repo/proto/user.proto \
    --call UserService.GetUser \
    -d '{"id": 1}' \
    -c 50 -n 1000 \
    localhost:50052
```

---

## 適用基準

### このパターンが適している場合
- gRPCの型安全性を活用したい
- 将来のマイクロサービス分離を想定
- 複数のクライアント（Web、Mobile、Desktop）対応
- チーム分割の可能性がある

### 注意すべき点
- Protocol Buffersの学習コスト
- gRPCデバッグの複雑性
- シリアライゼーションオーバーヘッド
- コード生成の管理

---

## 拡張性・移行戦略

### マイクロサービス分離
```go
// 現在：ローカル接続
dbClient := createLocalGRPCClient(dbService)

// 将来：ネットワーク接続
dbClient := grpc.Dial("database-service:50051")
```

### 新しいエンティティ追加
1. Database-repo: Protocol Buffers定義 → Migration → 実装
2. Handlers-repo: Protocol Buffers定義 → ビジネスロジック実装
3. Server-repo: Gateway設定追加（自動的に全プロトコル対応）

### 技術変更
- **Database層**: GORM → MongoDB等（gRPCインターフェースは維持）
- **API層**: 新プロトコル追加時もserver-repoのみ変更
- **言語変更**: 各層で独立して言語変更可能（Protocol Buffers互換性）

### 段階的移行
1. **Phase 1**: 単一プロセス運用
2. **Phase 2**: database-repo分離
3. **Phase 3**: handlers-repo分離
4. **Phase 4**: 完全マイクロサービス
