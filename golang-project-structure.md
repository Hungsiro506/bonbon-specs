# Go Project Structure Guide

A comprehensive guide for structuring Go projects based on clean architecture principles. This guide provides a scalable, maintainable structure suitable for microservices and monolithic applications.

## Overview

```
myproject/
├── cmd/                    # Application entry points
├── config/                 # Configuration structs and loading
├── internal/               # Private application code
│   ├── app/               # Application bootstrap and wiring
│   ├── deps/              # Dependency injection container
│   ├── domain/            # Core business logic and entities
│   ├── handler/           # Request handlers (HTTP, gRPC)
│   ├── repo/              # Data access layer
│   ├── usecase/           # Business logic orchestration
│   └── metrics/           # Application metrics
├── pkg/                    # Shared libraries (can be imported by other projects)
├── mocks/                  # Generated mocks for testing
├── docs/                   # Documentation
├── scripts/                # Build and utility scripts
├── charts/                 # Kubernetes Helm charts
├── Makefile               # Build commands
├── .mockery.yaml          # Mock generation config
└── go.mod                 # Go module definition
```

## Directory Responsibilities

### 1. `cmd/` - Application Entry Points

Each subdirectory is a separate executable with its own `main.go`.

```
cmd/
├── api/                   # Main API server
│   └── main.go
├── worker/                # Background job processor
│   └── main.go
├── migration/             # Database migration tool
│   └── main.go
└── cli/                   # Command-line tools
    └── main.go
```

**Responsibility**: Minimal bootstrap code only
- Load configuration
- Create logger
- Initialize application
- Start servers

**Example**:

```go
package main

import (
    "context"
    "log"
    "os"

    "myproject/config"
    "myproject/internal/app"
    "myproject/pkg/logger"
)

func main() {
    ctx := context.Background()

    cfg, err := config.GetConfig()
    if err != nil {
        log.Fatalf("Config error: %s", err)
    }

    appLogger := logger.New("json", logger.AllowInfo(), os.Stdout)
    appLogger.InfoCtx(ctx, "Starting Application", "version", cfg.App.Version)

    application, err := app.New(cfg, appLogger)
    if err != nil {
        log.Fatalf("Failed to create application: %s", err)
    }

    if err := application.Run(); err != nil {
        log.Fatalf("Application error: %s", err)
    }
}
```

### 2. `config/` - Configuration Management

Centralized configuration loading from environment variables.

```
config/
├── config.go              # Main config struct with env tags
└── config_test.go
```

**Responsibility**:
- Define configuration structs with `env` tags
- Singleton pattern for config loading
- Validation of required fields

**Example**:

```go
package config

import (
    "fmt"
    "sync"

    "github.com/caarlos0/env/v9"
)

type Config struct {
    App      App
    HTTP     HTTP
    Database Database
    Redis    Redis
}

type App struct {
    Name    string `env:"APP_NAME" envDefault:"myapp"`
    Version string `env:"APP_VERSION" envDefault:"1.0.0"`
    Env     string `env:"APP_ENV" envDefault:"development"`
}

type HTTP struct {
    Port         string `env:"HTTP_PORT" envDefault:"8080"`
    ReadTimeout  int    `env:"HTTP_READ_TIMEOUT" envDefault:"30"`
    WriteTimeout int    `env:"HTTP_WRITE_TIMEOUT" envDefault:"30"`
}

type Database struct {
    Host     string `env:"DB_HOST" envDefault:"localhost"`
    Port     string `env:"DB_PORT" envDefault:"3306"`
    Name     string `env:"DB_NAME,required"`
    User     string `env:"DB_USER,required"`
    Password string `env:"DB_PASSWORD,required"`
}

var (
    configOnce     sync.Once
    configInstance *Config
    configErr      error
)

func GetConfig() (*Config, error) {
    configOnce.Do(func() {
        configInstance = &Config{}
        configErr = env.Parse(configInstance)
        if configErr != nil {
            configErr = fmt.Errorf("config error: %w", configErr)
        }
    })
    return configInstance, configErr
}
```

### 3. `internal/` - Private Application Code

Code that cannot be imported by other projects.

#### 3.1 `internal/app/` - Application Bootstrap

Wires all components together and starts servers.

```go
package app

type Application struct {
    cfg        *config.Config
    logger     logger.Logger
    deps       *deps.Dependencies
    httpServer *httpserver.Server
    grpcServer *grpcserver.Server
}

func New(cfg *config.Config, logger logger.Logger) (*Application, error) {
    deps, err := deps.Initialize(cfg, logger)
    if err != nil {
        return nil, fmt.Errorf("failed to initialize dependencies: %w", err)
    }

    return &Application{
        cfg:    cfg,
        logger: logger,
        deps:   deps,
    }, nil
}

func (a *Application) Run() error {
    // Start HTTP server
    // Start gRPC server
    // Wait for shutdown signal
    // Graceful shutdown
}
```

#### 3.2 `internal/deps/` - Dependency Injection Container

Central place for creating and wiring all dependencies.

```go
package deps

type Dependencies struct {
    Config *config.Config
    Logger logger.Logger
    DB     *gorm.DB
    Cache  cache.Cache

    // Repositories
    UserRepo    repo.UserRepo
    ProductRepo repo.ProductRepo

    // Use Cases
    UserService    usecase.UserService
    ProductService usecase.ProductService
}

func Initialize(cfg *config.Config, logger logger.Logger) (*Dependencies, error) {
    d := &Dependencies{
        Config: cfg,
        Logger: logger,
    }

    // Initialize in order of dependencies
    if err := d.initDatabase(); err != nil {
        return nil, err
    }
    if err := d.initCache(); err != nil {
        return nil, err
    }
    d.initRepositories()
    d.initUseCases()

    return d, nil
}

func (d *Dependencies) initRepositories() {
    d.UserRepo = user.NewUserRepo(d.DB)
    d.ProductRepo = product.NewProductRepo(d.DB)
}

func (d *Dependencies) initUseCases() {
    d.UserService = useruc.New(d.UserRepo, d.Logger)
    d.ProductService = productuc.New(d.ProductRepo, d.Cache, d.Logger)
}
```

#### 3.3 `internal/domain/` - Core Business Logic

Pure business logic with no external dependencies.

```
internal/domain/
├── user.go                # User entity and value objects
├── product.go             # Product entity
├── errors.go              # Domain-specific errors
└── transformer/           # Complex transformation logic
    ├── transformer.go
    └── transformer_test.go
```

**Responsibility**:
- Entity definitions (structs)
- Value objects
- Domain errors
- Pure business logic (no I/O)

**Example**:

```go
package domain

import "time"

type User struct {
    ID        uint      `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    Role      UserRole  `json:"role"`
    CreatedAt time.Time `json:"createdAt"`
    UpdatedAt time.Time `json:"updatedAt"`
}

type UserRole string

const (
    UserRoleAdmin  UserRole = "admin"
    UserRoleMember UserRole = "member"
)

type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email"`
    Name  string `json:"name" validate:"required,min=2,max=100"`
    Role  string `json:"role" validate:"required,oneof=admin member"`
}

func (r *CreateUserRequest) Validate() error {
    if r.Email == "" {
        return ErrEmailRequired
    }
    return nil
}

type CreateUserResponse struct {
    User *User `json:"user"`
}
```

#### 3.4 `internal/handler/` - Request Handlers

Handles incoming requests (HTTP, gRPC, etc.)

```
internal/handler/
├── http/
│   ├── middleware/
│   │   ├── auth.go
│   │   ├── logging.go
│   │   └── recovery.go
│   └── v1/
│       ├── router.go
│       ├── user_handler.go
│       ├── request/
│       │   └── user_request.go
│       └── response/
│           └── user_response.go
└── grpc/
    └── v1/
        ├── user/
        │   ├── handler.go
        │   └── mapper.go
        ├── mapper/
        │   └── common_mapper.go
        └── errors/
            └── error_handler.go
```

**Responsibility**:
- Parse and validate requests
- Call use cases
- Map responses
- Handle errors

**Example HTTP Handler**:

```go
package v1

type UserHandler struct {
    userService usecase.UserService
    logger      logger.Logger
}

func NewUserHandler(userService usecase.UserService, logger logger.Logger) *UserHandler {
    return &UserHandler{
        userService: userService,
        logger:      logger,
    }
}

func (h *UserHandler) CreateUser(c *fiber.Ctx) error {
    var req domain.CreateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": "invalid request body"})
    }

    if err := req.Validate(); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
    }

    resp, err := h.userService.CreateUser(c.Context(), &req)
    if err != nil {
        h.logger.ErrorCtx(c.Context(), "Failed to create user", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{"error": "internal error"})
    }

    return c.Status(fiber.StatusCreated).JSON(resp)
}
```

**Example gRPC Handler**:

```go
package user

type Handler struct {
    v1.UnimplementedUserServiceServer
    userService  usecase.UserService
    logger       logger.Logger
    mapper       *Mapper
    errorHandler *errors.ErrorHandler
}

func NewHandler(userService usecase.UserService, logger logger.Logger) *Handler {
    return &Handler{
        userService:  userService,
        logger:       logger,
        mapper:       NewMapper(),
        errorHandler: errors.NewErrorHandler(logger),
    }
}

func (h *Handler) CreateUser(ctx context.Context, req *v1.CreateUserRequest) (*v1.CreateUserResponse, error) {
    domainReq := h.mapper.ToCreateUserRequest(req)

    resp, err := h.userService.CreateUser(ctx, domainReq)
    if err != nil {
        return nil, h.errorHandler.HandleError(err, "CreateUser", nil)
    }

    return h.mapper.ToProtoCreateUserResponse(resp), nil
}
```

#### 3.5 `internal/repo/` - Data Access Layer

Abstracts data storage (database, cache, external APIs).

```
internal/repo/
├── contracts.go           # Interface definitions
├── persistent/            # Database implementations
│   ├── user/
│   │   ├── repo.go
│   │   ├── repo_test.go
│   │   └── model.go       # GORM models
│   └── product/
│       └── repo.go
└── apiclient/             # External API clients
    └── payment/
        ├── client.go
        └── client_test.go
```

**Contracts (interfaces)**:

```go
package repo

import (
    "context"
    "myproject/internal/domain"
)

type UserRepo interface {
    Create(ctx context.Context, user *domain.User) (*domain.User, error)
    GetByID(ctx context.Context, id uint) (*domain.User, error)
    GetByEmail(ctx context.Context, email string) (*domain.User, error)
    Update(ctx context.Context, user *domain.User) (*domain.User, error)
    Delete(ctx context.Context, id uint) error
    List(ctx context.Context, filter *domain.UserFilter) ([]*domain.User, error)
}

type ProductRepo interface {
    Create(ctx context.Context, product *domain.Product) (*domain.Product, error)
    GetByID(ctx context.Context, id uint) (*domain.Product, error)
}

type PaymentClient interface {
    CreateCharge(ctx context.Context, req *domain.ChargeRequest) (*domain.ChargeResponse, error)
    RefundCharge(ctx context.Context, chargeID string) error
}
```

**Implementation**:

```go
package user

import (
    "context"
    "fmt"

    "gorm.io/gorm"
    "myproject/internal/domain"
)

type UserRepoImpl struct {
    db *gorm.DB
}

func NewUserRepo(db *gorm.DB) *UserRepoImpl {
    return &UserRepoImpl{db: db}
}

func (r *UserRepoImpl) Create(ctx context.Context, user *domain.User) (*domain.User, error) {
    model := toModel(user)
    if err := r.db.WithContext(ctx).Create(model).Error; err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }
    return toEntity(model), nil
}

func (r *UserRepoImpl) GetByID(ctx context.Context, id uint) (*domain.User, error) {
    var model UserModel
    if err := r.db.WithContext(ctx).First(&model, id).Error; err != nil {
        if err == gorm.ErrRecordNotFound {
            return nil, nil
        }
        return nil, fmt.Errorf("failed to get user: %w", err)
    }
    return toEntity(&model), nil
}

func toModel(entity *domain.User) *UserModel {
    return &UserModel{
        ID:    entity.ID,
        Email: entity.Email,
        Name:  entity.Name,
        Role:  string(entity.Role),
    }
}

func toEntity(model *UserModel) *domain.User {
    return &domain.User{
        ID:        model.ID,
        Email:     model.Email,
        Name:      model.Name,
        Role:      domain.UserRole(model.Role),
        CreatedAt: model.CreatedAt,
        UpdatedAt: model.UpdatedAt,
    }
}
```

#### 3.6 `internal/usecase/` - Business Logic Orchestration

Coordinates repositories and implements business rules.

```
internal/usecase/
├── contracts.go           # Interface definitions
├── user/
│   ├── service.go
│   └── service_test.go
└── product/
    ├── service.go
    └── service_test.go
```

**Contracts**:

```go
package usecase

import (
    "context"
    "myproject/internal/domain"
)

//go:generate mockery --name=UserService --output=../../mocks/usecase --filename=mock_user_service.go

type UserService interface {
    CreateUser(ctx context.Context, req *domain.CreateUserRequest) (*domain.CreateUserResponse, error)
    GetUser(ctx context.Context, id uint) (*domain.User, error)
    UpdateUser(ctx context.Context, id uint, req *domain.UpdateUserRequest) (*domain.User, error)
    DeleteUser(ctx context.Context, id uint) error
}

type ProductService interface {
    CreateProduct(ctx context.Context, req *domain.CreateProductRequest) (*domain.CreateProductResponse, error)
    GetProduct(ctx context.Context, id uint) (*domain.Product, error)
}
```

**Implementation**:

```go
package user

import (
    "context"
    "fmt"

    "myproject/internal/domain"
    "myproject/internal/repo"
    "myproject/pkg/logger"
)

type Service struct {
    userRepo repo.UserRepo
    logger   logger.Logger
}

func New(userRepo repo.UserRepo, logger logger.Logger) *Service {
    return &Service{
        userRepo: userRepo,
        logger:   logger,
    }
}

func (s *Service) CreateUser(ctx context.Context, req *domain.CreateUserRequest) (*domain.CreateUserResponse, error) {
    existing, err := s.userRepo.GetByEmail(ctx, req.Email)
    if err != nil {
        return nil, fmt.Errorf("failed to check existing user: %w", err)
    }
    if existing != nil {
        return nil, domain.ErrUserAlreadyExists
    }

    user := &domain.User{
        Email: req.Email,
        Name:  req.Name,
        Role:  domain.UserRole(req.Role),
    }

    created, err := s.userRepo.Create(ctx, user)
    if err != nil {
        return nil, fmt.Errorf("failed to create user: %w", err)
    }

    s.logger.InfoCtx(ctx, "User created successfully", "userId", created.ID)

    return &domain.CreateUserResponse{User: created}, nil
}
```

### 4. `pkg/` - Shared Libraries

Reusable packages that can be imported by other projects.

```
pkg/
├── logger/                # Structured logging
│   ├── logger.go
│   └── context.go
├── httpserver/            # HTTP server wrapper
│   └── server.go
├── grpcserver/            # gRPC server wrapper
│   └── server.go
├── mysql/                 # Database connection
│   └── mysql.go
├── cache/                 # Cache interface and implementations
│   ├── cache.go
│   └── redis.go
├── auth/                  # Authentication utilities
│   └── jwt.go
├── errorutils/            # Error handling utilities
│   └── errors.go
└── worker/                # Background job utilities
    └── worker.go
```

**Example Logger Package**:

```go
package logger

import (
    "context"
    "io"
    "log/slog"
)

type Logger interface {
    Info(msg string, args ...interface{})
    Warn(msg string, args ...interface{})
    Error(msg string, err error, args ...interface{})
    InfoCtx(ctx context.Context, msg string, args ...interface{})
    WarnCtx(ctx context.Context, msg string, args ...interface{})
    ErrorCtx(ctx context.Context, msg string, err error, args ...interface{})
}

type slogLogger struct {
    logger *slog.Logger
}

func New(format string, level slog.Leveler, output io.Writer) Logger {
    var handler slog.Handler
    if format == "json" {
        handler = slog.NewJSONHandler(output, &slog.HandlerOptions{Level: level})
    } else {
        handler = slog.NewTextHandler(output, &slog.HandlerOptions{Level: level})
    }
    return &slogLogger{logger: slog.New(handler)}
}
```

### 5. `mocks/` - Generated Mocks

Auto-generated mock implementations for testing.

```yaml
# .mockery.yaml
all: false
dir: 'mocks/{{.SrcPackageName}}'
filename: 'mock_{{.InterfaceName | snakecase}}.go'
structname: 'Mock{{.InterfaceName}}'
pkgname: 'mock{{.SrcPackageName}}'
packages:
  myproject/internal/repo:
    interfaces:
      UserRepo:
      ProductRepo:
  myproject/internal/usecase:
    interfaces:
      UserService:
      ProductService:
```

Generate mocks with:
```bash
make gen-mockery
```

## Layer Dependencies

```
cmd/
 └── internal/app/
      └── internal/deps/
           ├── internal/handler/  ← depends on usecase
           ├── internal/usecase/  ← depends on repo, domain
           ├── internal/repo/     ← depends on domain
           └── internal/domain/   ← no dependencies
```

**Rules**:
1. `domain` has NO external dependencies
2. `repo` depends only on `domain`
3. `usecase` depends on `repo` and `domain`
4. `handler` depends on `usecase` and `domain`
5. `deps` wires everything together
6. `app` bootstraps the application

## Testing Strategy

### Unit Tests

Test each layer in isolation using mocks.

```go
package user_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"

    "myproject/internal/domain"
    mockrepo "myproject/mocks/repo"
    "myproject/internal/usecase/user"
    "myproject/pkg/logger"
)

func TestService_CreateUser(t *testing.T) {
    testCases := []struct {
        name          string
        setupMocks    func(*mockrepo.MockUserRepo)
        request       *domain.CreateUserRequest
        expectError   bool
        errorContains string
    }{
        {
            name: "success",
            setupMocks: func(repo *mockrepo.MockUserRepo) {
                repo.EXPECT().GetByEmail(mock.Anything, "test@example.com").Return(nil, nil)
                repo.EXPECT().Create(mock.Anything, mock.Anything).Return(&domain.User{ID: 1}, nil)
            },
            request:     &domain.CreateUserRequest{Email: "test@example.com", Name: "Test", Role: "member"},
            expectError: false,
        },
        {
            name: "user_already_exists",
            setupMocks: func(repo *mockrepo.MockUserRepo) {
                repo.EXPECT().GetByEmail(mock.Anything, "test@example.com").Return(&domain.User{}, nil)
            },
            request:       &domain.CreateUserRequest{Email: "test@example.com"},
            expectError:   true,
            errorContains: "already exists",
        },
    }

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            mockRepo := mockrepo.NewMockUserRepo(t)
            tc.setupMocks(mockRepo)

            svc := user.New(mockRepo, logger.NewNilLogger())
            resp, err := svc.CreateUser(context.Background(), tc.request)

            if tc.expectError {
                require.Error(t, err)
                assert.Contains(t, err.Error(), tc.errorContains)
            } else {
                require.NoError(t, err)
                assert.NotNil(t, resp)
            }

            mockRepo.AssertExpectations(t)
        })
    }
}
```

## Makefile Commands

```makefile
.PHONY: build run test lint gen-mockery

build:
	go build -o bin/app ./cmd/app

run:
	go run ./cmd/app

test-unit:
	go test -v -race -covermode atomic -coverprofile=coverage.txt ./internal/...

test-integration:
	go test -v -tags=integration ./integration-test/...

lint:
	golangci-lint run ./...

gen-mockery:
	@mockery

migrate-up:
	go run ./cmd/migration up

migrate-down:
	go run ./cmd/migration down
```

## Quick Start Checklist

1. [ ] Create `go.mod` with `go mod init myproject`
2. [ ] Create directory structure
3. [ ] Set up `config/` with environment variable parsing
4. [ ] Create `internal/domain/` with entities
5. [ ] Define interfaces in `internal/repo/contracts.go` and `internal/usecase/contracts.go`
6. [ ] Implement repositories in `internal/repo/persistent/`
7. [ ] Implement use cases in `internal/usecase/`
8. [ ] Create handlers in `internal/handler/`
9. [ ] Wire dependencies in `internal/deps/`
10. [ ] Bootstrap in `internal/app/`
11. [ ] Create entry point in `cmd/app/main.go`
12. [ ] Set up `.mockery.yaml` and generate mocks
13. [ ] Write tests

## Summary Table

| Layer | Location | Responsibility | Dependencies |
|-------|----------|----------------|--------------|
| Entry Point | `cmd/` | Bootstrap, minimal code | app |
| Config | `config/` | Load env vars, validation | none |
| Bootstrap | `internal/app/` | Start servers | deps |
| DI Container | `internal/deps/` | Wire components | all internal |
| Domain | `internal/domain/` | Entities, business rules | none |
| Use Case | `internal/usecase/` | Business logic orchestration | repo, domain |
| Repository | `internal/repo/` | Data access abstraction | domain |
| Handler | `internal/handler/` | Request/response handling | usecase, domain |
| Shared Libs | `pkg/` | Reusable utilities | external only |
