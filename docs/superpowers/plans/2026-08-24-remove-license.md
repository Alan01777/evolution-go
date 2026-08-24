# Remove Evolution Foundation License — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remover completamente a dependência do servidor de licença da Evolution Foundation para que o projeto inicie e opere sem registro/ativação.

**Architecture:** O licenciamento vive em `pkg/core/c0.go` (pacote inteiro, ofuscado) e é chamado apenas por `cmd/evolution-go/main.go`. A mudança apaga o pacote, remove os 8 pontos de integração no `main.go` e adiciona um stub mínimo `GET /license/status` para manter o manager (bundle React em `manager/dist`) funcionando.

**Tech Stack:** Go 1.25, Gin, GORM, whatsmeow.

## Global Constraints

- `GLOBAL_API_KEY` **permanece** — é a autenticação admin da API (`AuthAdmin` + `GET /ws`), não o licenciamento. Não remover nem alterar seu uso em `pkg/middleware/auth_middleware.go` ou no handler `GET /ws`.
- `LICENSE`, `NOTICE`, `TRADEMARKS.md` **não** devem ser alterados.
- Não regenerar o Swagger (`docs/docs.go`, `docs/swagger.json`) — desatualização cosmética está fora de escopo.
- `go build ./...`, `go vet ./...` e `go test ./...` devem passar ao final de cada task.
- Estilo de commit do repositório: mensagens no padrão `type: descrição` em inglês (ex.: `feat:`, `refactor:`).

---

### Task 1: Remover o pacote `core` e toda a integração no `main.go`

**Files:**
- Delete: `pkg/core/c0.go` (e o diretório `pkg/core/`)
- Modify: `cmd/evolution-go/main.go`

**Interfaces:**
- Consumes: nenhum (esta task não depende de outras).
- Produces: um `main.go` que compila sem o pacote `core` e sem a rota `/license/status` (restaurada na Task 2).

- [ ] **Step 1: Apagar o pacote `core`**

```bash
git rm -r pkg/core
```

- [ ] **Step 2: Remover o import do `core` no `main.go`**

Em `cmd/evolution-go/main.go`, remova a linha do import:

```go
	config "github.com/evolution-foundation/evolution-go/pkg/config"
	"github.com/evolution-foundation/evolution-go/pkg/core"
	producer_interfaces "github.com/evolution-foundation/evolution-go/pkg/events/interfaces"
```

fica:

```go
	config "github.com/evolution-foundation/evolution-go/pkg/config"
	producer_interfaces "github.com/evolution-foundation/evolution-go/pkg/events/interfaces"
```

- [ ] **Step 3: Remover o parâmetro `runtimeCtx` da assinatura de `setupRouter`**

Altere:

```go
func setupRouter(db *gorm.DB, authDB *sql.DB, sqliteDB *sql.DB, config *config.Config, conn *amqp.Connection, exPath string, runtimeCtx *core.RuntimeContext) *gin.Engine {
```

para:

```go
func setupRouter(db *gorm.DB, authDB *sql.DB, sqliteDB *sql.DB, config *config.Config, conn *amqp.Connection, exPath string) *gin.Engine {
```

- [ ] **Step 4: Remover o `GateMiddleware`**

Remova a linha:

```go
	r.Use(core.GateMiddleware(runtimeCtx))
```

- [ ] **Step 5: Remover as `LicenseRoutes`**

Remova o bloco:

```go
	// License routes (always accessible, even without license)
	core.LicenseRoutes(r, runtimeCtx)
```

- [ ] **Step 6: Remover `startTime`**

Remova a linha:

```go
	startTime := time.Now()
```

- [ ] **Step 7: Remover o bloco de inicialização do `core` (SetDB/MigrateDB/InitializeRuntime)**

Remova o bloco:

```go
	// Initialize core DB + license runtime
	core.SetDB(db)
	if err := core.MigrateDB(); err != nil {
		log.Fatal("Failed to migrate runtime_configs: ", err)
	}
	tier := "evolution-go"
	runtimeCtx := core.InitializeRuntime(tier, version, cfg.GlobalApiKey)
```

- [ ] **Step 8: Remover o argumento `runtimeCtx` na chamada de `setupRouter`**

Altere:

```go
	r := setupRouter(db, authDB, sqliteDB, cfg, conn, exPath, runtimeCtx)
```

para:

```go
	r := setupRouter(db, authDB, sqliteDB, cfg, conn, exPath)
```

- [ ] **Step 9: Remover o heartbeat**

Remova o bloco:

```go
	// Graceful shutdown with heartbeat
	heartbeatCtx, heartbeatCancel := context.WithCancel(context.Background())
	defer heartbeatCancel()

	core.StartHeartbeat(heartbeatCtx, runtimeCtx, startTime)
```

- [ ] **Step 10: Remover a parada do heartbeat e o `Shutdown` do `core`**

Remova a linha:

```go
	// Stop heartbeat loop
	heartbeatCancel()
```

e remova a linha:

```go
	core.Shutdown(runtimeCtx)
```

- [ ] **Step 11: Verificar que não há mais referências ao `core`**

Run:

```bash
rg -n "core\.|pkg/core|RuntimeContext|runtimeCtx|heartbeat" cmd/evolution-go/main.go
```

Expected: nenhuma saída (o único `context.` restante é o `context.WithTimeout` do shutdown HTTP, que permanece).

- [ ] **Step 12: Build, vet e test**

Run:

```bash
go build ./... && go vet ./... && go test ./...
```

Expected: sucesso nas três etapas.

- [ ] **Step 13: Commit**

```bash
git add -A
git commit -m "refactor: remove Evolution Foundation license requirement"
```

---

### Task 2: Adicionar stub `GET /license/status` para o manager

**Files:**
- Modify: `cmd/evolution-go/main.go`

**Interfaces:**
- Consumes: o `main.go` da Task 1 (sem `core`).
- Produces: rota `GET /license/status` que responde `{"status":"active"}` — o manager (`manager/dist`) chama esse endpoint e só libera o login quando `.status === "active"`.

- [ ] **Step 1: Adicionar o stub logo após o bloco CORS em `setupRouter`**

Em `cmd/evolution-go/main.go`, dentro de `setupRouter`, logo após o `r.Use(func(c *gin.Context) { ... })` do CORS (que termina com `c.Next()` e `})`), adicione:

```go
	// Stub de compatibilidade: o manager (bundle React em manager/dist) consulta
	// GET /license/status e só libera o login quando status === "active".
	// O licenciamento da Evolution Foundation foi removido deste fork.
	r.GET("/license/status", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"status": "active"})
	})
```

> Nota: `http` e `gin` já estão importados no arquivo (usados em `http.Server`, `gin.Default`, etc.). Nenhum import novo é necessário.

- [ ] **Step 2: Build, vet e test**

Run:

```bash
go build ./... && go vet ./... && go test ./...
```

Expected: sucesso nas três etapas.

- [ ] **Step 3: Smoke test manual (opcional, requer Postgres + `.env`)**

Se o ambiente tiver Postgres e um `.env` configurado, iniciar o servidor e verificar:

```bash
curl -s http://localhost:8080/license/status
# expected: {"status":"active"}
curl -s http://localhost:8080/server/ok
# expected: 200 (não 503)
```

Expected: `/license/status` retorna `{"status":"active"}` e `/server/ok` responde 200 (antes da mudança, `/server/ok` também estava na whitelist, mas demais rotas retornavam 503; agora nenhuma rota é bloqueada por licença).

- [ ] **Step 4: Commit**

```bash
git add cmd/evolution-go/main.go
git commit -m "feat: add /license/status stub for manager compatibility"
```

---

## Self-Review

**Spec coverage:**
- Apagar `pkg/core/` → Task 1 Step 1. ✔
- Remover os 8 pontos de integração no `main.go` → Task 1 Steps 2-10. ✔
- Adicionar stub `GET /license/status` → Task 2 Step 1. ✔
- Manter `GLOBAL_API_KEY` → Global Constraints. ✔
- Manter docs legais / não regenerar Swagger → Global Constraints. ✔
- Verificação build/vet/test → Steps 12 (Task 1) e 2 (Task 2). ✔

**Placeholder scan:** nenhum "TBD"/"TODO"; todos os passos de código têm código exato.

**Type consistency:** não há assinaturas/funções novas; apenas remoções e um handler inline. Consistente.
