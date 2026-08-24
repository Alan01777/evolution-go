# Spec — Remoção do licenciamento da Evolution Foundation

**Data:** 2026-08-24
**Status:** Aprovado

## Objetivo

Remover completamente a dependência do servidor de licença da Evolution
Foundation (`license.evolutionfoundation.com.br`) do fork. Após a mudança, o
projeto inicia e opera imediatamente, sem registro, ativação, gate de status,
heartbeat ou telemetria.

## Escopo do sistema de licenciamento atual

O sistema é autocontido em dois lugares:

- `pkg/core/c0.go` — pacote inteiro (978 linhas, ofuscado). Contém: geração de
  `instance_id`, persistência em `runtime_configs`, fluxo de registro/ativação,
  `GateMiddleware` (bloqueia rotas com `503 LICENSE_REQUIRED`), `LicenseRoutes`
  (`/license/register`, `/license/activate`, `/license/status`), heartbeat (30
  min) com telemetria de mensagens, deactivate no shutdown e código
  anti-tamper morto (`ComputeSessionSeed`, `ValidateRouteAccess`,
  `DeriveInstanceToken`, `ActivateIntegrity`, `TrackMessageSent`,
  `TrackMessageRecv`) — nenhum desses é chamado fora do pacote.
- `cmd/evolution-go/main.go` — 8 pontos de chamada ao `core`.

Dependências externas do sistema de licença:

- O manager frontend (`manager/dist`, bundle React) chama `GET /license/status`
  e só libera o login se a resposta tiver `.status === "active"`.

`GLOBAL_API_KEY` **não** faz parte do licenciamento: é a chave de autenticação
admin da própria API (middleware `AuthAdmin` e `GET /ws`). Permanece intacta.

## Mudanças

### 1. Apagar `pkg/core/`

Remover o diretório `pkg/core/` inteiro (contém apenas `c0.go`).

### 2. `cmd/evolution-go/main.go`

Remover os pontos de integração com o `core`:

- import `github.com/evolution-foundation/evolution-go/pkg/core`
- parâmetro `runtimeCtx *core.RuntimeContext` de `setupRouter` e o argumento
  correspondente na chamada
- `r.Use(core.GateMiddleware(runtimeCtx))`
- `core.LicenseRoutes(r, runtimeCtx)`
- `core.SetDB(db)` e o bloco `core.MigrateDB()` (e o `log.Fatal` associado)
- `tier := "evolution-go"` e `core.InitializeRuntime(...)`
- `startTime := time.Now()` (usado apenas pelo heartbeat)
- `heartbeatCtx`/`heartbeatCancel` e `core.StartHeartbeat(...)`
- `core.Shutdown(runtimeCtx)` e a chamada `heartbeatCancel()`

Adicionar um stub para manter o manager funcional:

```go
r.GET("/license/status", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{"status": "active"})
})
```

Nenhum endpoint `/license/register` ou `/license/activate` é necessário: o
manager só os chama quando `status !== "active"`.

### 3. Fora de escopo

- `LICENSE`, `NOTICE`, `TRADEMARKS.md` — documentos legais de copyright/marca,
  não o mecanismo de ativação. Permanecem inalterados.
- Rebranding do manager ("Evolution Go" logo/branding) — relevante apenas se o
  fork for distribuído (TRADEMARKS §4.2); tarefa separada.
- Regeneração do Swagger (`docs/docs.go`, `docs/swagger.json`), que ainda lista
  `/license/register` e `/license/activate`. Desatualização cosmética.

## Comportamento esperado após a mudança

- O servidor inicia sem consultar a internet e sem exigir cadastro.
- `GET /server/ok` responde normalmente.
- `GET /license/status` retorna `{"status":"active"}` para o manager.
- O manager permite login com `GLOBAL_API_KEY` (validação via `/instance/all`).

## Verificação

- `go build ./...` sem erros.
- `go vet ./...` limpo.
- `go test ./...` passa (o único teste que menciona "core" usa a palavra como
  string literal não relacionada ao licenciamento).
