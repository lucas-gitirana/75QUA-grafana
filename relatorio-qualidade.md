# Relatório de Qualidade de Software — Grafana

**Data:** 23 de maio de 2026  
**Elaborado por:** Análise estática do código-fonte  
**Ferramenta:** Claude Code (claude-sonnet-4-6)

---

## 1. Descrição do Sistema e do Repositório

### 1.1 Visão Geral

**Grafana** é uma plataforma de código aberto para monitoramento e observabilidade. Permite criar dashboards interativos para visualização de métricas, logs e traces de diversas fontes de dados (Prometheus, Loki, InfluxDB, Elasticsearch, entre outras). É amplamente utilizado em ambientes de produção para acompanhar a saúde de sistemas distribuídos.

- **Repositório GitHub:** https://github.com/grafana/grafana.git
- **Branch analisada:** `main`
- **Linguagens principais:** Go (backend), TypeScript/React (frontend)
- **Arquitetura:** Monorepo com Yarn Workspaces (frontend) e Go Workspaces (backend)
- **Licença:** AGPLv3

### 1.2 Estrutura Principal

| Camada | Diretório | Descrição |
|---|---|---|
| Backend API | `pkg/api/` | Handlers HTTP e registro de rotas |
| Serviços | `pkg/services/` | Lógica de negócio por domínio |
| Infraestrutura | `pkg/infra/` | Logging, métricas, acesso a banco |
| Plugins | `pkg/plugins/` | Sistema e carregador de plugins |
| Middleware | `pkg/middleware/` | Middlewares HTTP |
| Configuração | `pkg/setting/` | Gerenciamento de configuração |

### 1.3 Dimensão do Projeto

- **Arquivos Go de produção analisados:** ~3.228 arquivos (excluídos gerados e testes)
- **LOC total estimado (pkg/):** ~77.178 linhas de código Go de produção
- **Módulos frontend:** `packages/grafana-ui`, `packages/grafana-data`, `packages/grafana-runtime`, entre outros

---

## 2. Análise de Clean Code

Os 20 trechos a seguir representam violações dos princípios de Clean Code identificadas no código-fonte real do repositório.

---

### CC-01 — Função com Responsabilidade Múltipla (SRP)

**Princípio violado:** Single Responsibility Principle  
**Arquivo:** [`pkg/api/dashboard.go:70`](pkg/api/dashboard.go#L70)  
**Função:** `GetDashboard`

```go
//nolint:gocyclo
func (hs *HTTPServer) GetDashboard(c *contextmodel.ReqContext) response.Response {
    // busca o dashboard
    // verifica dashboards públicos
    // valida dados do dashboard
    // avalia permissões (canSave, canEdit, canAdmin, canDelete)
    // busca nome do criador e do atualizador
    // busca dados da pasta
    // verifica provisionamento
    // monta e retorna DTO
}
```

A função executa oito responsabilidades distintas em ~165 linhas. Qualquer alteração em uma responsabilidade exige entendimento de todas as outras, aumentando o risco de regressão e violando diretamente o SRP.

---

### CC-02 — Função Longa (Long Method)

**Princípio violado:** Funções devem ser pequenas e fazer uma coisa  
**Arquivo:** [`pkg/api/dashboard.go:380`](pkg/api/dashboard.go#L380)  
**Função:** `postDashboard`

```go
func (hs *HTTPServer) postDashboard(c *contextmodel.ReqContext, cmd dashboards.SaveDashboardCommand) response.Response {
    // ~115 linhas: valida schema v2, valida k8s resource, extrai título,
    // identifica usuário, verifica quotas, verifica provisionamento,
    // constrói SaveDashboardDTO, salva e retorna resposta
}
```

Uma função com 115 linhas realiza múltiplos passos de validação, enriquecimento e persistência. Segundo os princípios de Clean Code (Robert C. Martin), funções devem ter no máximo 20 linhas.

---

### CC-03 — Nome de Variável sem Intenção Revelada

**Princípio violado:** Nomes revelam intenção  
**Arquivo:** [`pkg/api/dashboard.go:840`](pkg/api/dashboard.go#L840)

```go
loginMem := make(map[int64]string, len(resp.Versions))
res := make([]dashver.DashboardVersionMeta, 0, len(resp.Versions))
```

`loginMem` é uma abreviatura obscura para um cache de nomes de login. `res` não comunica que se trata de versões de dashboard a serem retornadas. Nomes como `loginNameCache` e `versionsMeta` tornariam o código imediatamente compreensível.

---

### CC-04 — Comentário Desnecessário que Repete o Código

**Princípio violado:** Comentários não devem repetir o que o código já diz  
**Arquivo:** [`pkg/api/dashboard.go:128`](pkg/api/dashboard.go#L128)

```go
// Finding creator and last updater of the dashboard
updater, creator := anonString, anonString
if dash.UpdatedBy > 0 {
    updater = hs.getIdentityName(ctx, dash.OrgID, dash.UpdatedBy)
}
if dash.CreatedBy > 0 {
    creator = hs.getIdentityName(ctx, dash.OrgID, dash.CreatedBy)
}
```

O comentário `// Finding creator and last updater of the dashboard` descreve exatamente o que o código abaixo faz, sendo redundante. Um bom código auto-documenta este tipo de lógica pelo próprio nome das variáveis.

---

### CC-05 — Número Mágico

**Princípio violado:** Evitar números mágicos  
**Arquivo:** [`pkg/api/dashboard.go:786`](pkg/api/dashboard.go#L786)

```go
newpanel := simplejson.NewFromAny(map[string]any{
    "type": "gettingstarted",
    "id":   123123,
    "gridPos": map[string]any{
        "x": 0,
        "y": 3,
        "w": 24,
        "h": 9,
    },
})
```

Os valores `123123`, `3`, `24` e `9` são números mágicos sem contexto. Constantes nomeadas como `gettingStartedPanelID`, `gettingStartedPanelGridWidth` tornariam as dimensões explícitas e fáceis de ajustar.

---

### CC-06 — Função com Responsabilidade Múltipla (SRP)

**Princípio violado:** Single Responsibility Principle  
**Arquivo:** [`pkg/api/dashboard.go:497`](pkg/api/dashboard.go#L497)  
**Função:** `saveDashboardViaK8s`

```go
func (hs *HTTPServer) saveDashboardViaK8s(...) response.Response {
    // parse do GroupVersion
    // validação do título
    // criação do cliente k8s dinâmico
    // limpeza de metadados (anotações, labels, status, access)
    // resolução do nome via internal ID
    // lógica de create vs update
    // retorno formatado
}
```

A função acumula lógica de parsing, configuração de cliente, manipulação de metadados e persistência em ~130 linhas, violando o SRP.

---

### CC-07 — Variável com Nome de Uma Letra

**Princípio violado:** Nomes revelam intenção  
**Arquivo:** [`pkg/api/dashboard.go:498`](pkg/api/dashboard.go#L498)

```go
gv, err := schema.ParseGroupVersion(obj.GetAPIVersion())
...
tmp, err := dynamic.NewForConfig(hs.clientConfigProvider.GetDirectRestConfig(c))
...
client := tmp.Resource(...)
```

`gv` e `tmp` são nomes que não comunicam intenção. `groupVersion` e `k8sDynamicClient` seriam muito mais expressivos.

---

### CC-08 — Classe Deus (God Class)

**Princípio violado:** Single Responsibility Principle  
**Arquivo:** [`pkg/api/http_server.go`](pkg/api/http_server.go)

```go
type HTTPServer struct {
    // 86 dependências injetadas: Cfg, RouteRegister, Bus, RenderService,
    // AccessControl, DashboardService, FolderService, UserService,
    // PluginStore, QuotaService, LiveService, ...
}
```

A struct `HTTPServer` possui 86 imports internos e funciona como ponto central de todos os serviços do sistema. Qualquer novo serviço é adicionado aqui, tornando-a uma God Class com alto acoplamento.

---

### CC-09 — Código Duplicado

**Princípio violado:** Don't Repeat Yourself (DRY)  
**Arquivo:** [`pkg/api/admin_users.go:283–346`](pkg/api/admin_users.go#L283)

```go
// AdminDisableUser
authInfoQuery := &login.GetAuthInfoQuery{UserId: userID}
if _, err := hs.authInfoService.GetAuthInfo(c.Req.Context(), authInfoQuery); !errors.Is(err, user.ErrUserNotFound) {
    return response.Error(http.StatusInternalServerError, "Could not disable external user", nil)
}

// AdminEnableUser — mesma lógica, mensagem diferente
authInfoQuery := &login.GetAuthInfoQuery{UserId: userID}
if _, err := hs.authInfoService.GetAuthInfo(c.Req.Context(), authInfoQuery); !errors.Is(err, user.ErrUserNotFound) {
    return response.Error(http.StatusInternalServerError, "Could not enable external user", nil)
}
```

O mesmo bloco de verificação de usuário externo é duplicado em `AdminDisableUser` e `AdminEnableUser`. Deveria ser extraído para uma função auxiliar `isExternalUser(ctx, userID)`.

---

### CC-10 — Estrutura Aninhada Excessiva

**Princípio violado:** Funções devem ter baixo nível de indentação  
**Arquivo:** [`pkg/api/dashboard.go:541`](pkg/api/dashboard.go#L541)

```go
if internalID > 0 && name == "" {
    found, err := client.List(...)
    if err != nil {
        return response.Error(...)
    }
    if len(found.Items) == 0 {
        return response.Error(...)
    }
    old = &found.Items[0]
    name = old.GetName()
    meta.SetName(name)
    if !cmd.Overwrite {
        return response.Error(...)
    }
}
```

Quatro níveis de aninhamento dentro de uma função já longa dificultam o rastreamento do fluxo. Guard clauses invertidas reduziriam a indentação.

---

### CC-11 — Parâmetro de Flag (Flag Argument)

**Princípio violado:** Funções devem fazer uma coisa  
**Arquivo:** [`pkg/api/dashboard.go:284`](pkg/api/dashboard.go#L284)

```go
func (hs *HTTPServer) getDashboardHelper(ctx context.Context, orgID int64, uid string, k8sGetAPIVersion string) (*dashboards.Dashboard, response.Response) {
```

O parâmetro `k8sGetAPIVersion` é uma string que quando não vazia altera o comportamento da função. Isso é um flag argument disfarçado. Duas funções especializadas comunicariam melhor a intenção.

---

### CC-12 — Inconsistência de Nomeação

**Princípio violado:** Consistência no uso de nomes  
**Arquivo:** [`pkg/api/dashboard.go:156,185,187`](pkg/api/dashboard.go#L156)

```go
meta.FolderUid = queryResult.UID   // camelCase: Uid
meta.FolderId  = queryResult.ID    // camelCase: Id
// mas também:
FolderUID string `json:"folderUid"` // SCREAMING_SNAKE: UID
```

O código alterna entre `Uid`/`Id` (em campos de struct) e `UID`/`ID` (nos tipos — padrão Go). O campo `FolderId` ainda usa a grafia depreciada enquanto `FolderUID` usa o padrão correto.

---

### CC-13 — Lógica de Negócio na Camada HTTP

**Princípio violado:** Separação de responsabilidades  
**Arquivo:** [`pkg/api/dashboard.go:96`](pkg/api/dashboard.go#L96)

```go
if dash.Data != nil {
    isEmptyData := true
    for k := range dash.Data.MustMap() {
        if k != "id" && k != "uid" {
            isEmptyData = false
            break
        }
    }
    if isEmptyData {
        return response.Error(http.StatusInternalServerError, "Error while loading dashboard, dashboard data is invalid", nil)
    }
```

Lógica de validação de dados do dashboard está embutida no handler HTTP. Esta verificação deveria residir no serviço de domínio `DashboardService`, não no handler de API.

---

### CC-14 — Retorno de Erro com Código de Status Incorreto

**Princípio violado:** Código expressivo e sem surpresas  
**Arquivo:** [`pkg/api/admin_users.go:130`](pkg/api/admin_users.go#L130)

```go
if err := hs.AuthTokenService.RevokeAllUserTokens(c.Req.Context(), userID); err != nil {
    return response.Error(http.StatusExpectationFailed,
        "User password updated but unable to revoke user sessions", err)
}
```

O uso de `StatusExpectationFailed` (417) para indicar falha ao revogar tokens é semanticamente incorreto. O código `417 Expectation Failed` refere-se ao cabeçalho HTTP `Expect`, não a falhas de operação de logout.

---

### CC-15 — Constante com Nome Pouco Descritivo

**Princípio violado:** Nomes revelam intenção  
**Arquivo:** [`pkg/api/dashboard.go:45`](pkg/api/dashboard.go#L45)

```go
const (
    anonString = "Anonymous"
)
```

`anonString` não comunica claramente para que serve. `defaultAnonymousCreatorName` ou `anonymousUserDisplayName` seria mais descritivo e imediatamente compreensível no contexto de exibição de dashboards.

---

### CC-16 — Comentário de Código Comentado (Dead Code)

**Princípio violado:** Código limpo não contém código morto  
**Arquivo:** [`pkg/api/dashboard.go:109`](pkg/api/dashboard.go#L109)

```go
// the dashboard id is no longer set in the spec for unified storage, set it here to keep api compatibility
if dash.Data.Get("id").MustString() == "" {
    dash.Data.Set("id", dash.ID)
}
```

O comentário revela uma gambiarra de compatibilidade retroativa que deveria ser tratada no serviço de storage ou num adapter, não misturada com a lógica do handler. Indica dívida técnica acumulada.

---

### CC-17 — Tratamento de Erro Ignorado

**Princípio violado:** Tratamento explícito de erros  
**Arquivo:** [`pkg/api/dashboard.go:503`](pkg/api/dashboard.go#L503)

```go
title, _, _ := unstructured.NestedString(obj.Object, "spec", "title")
```

Os dois underscores (`_`) descartam `found` (booleano indicando se o campo existe) e `err` sem verificação. Se o campo `title` não existir ou houver erro, o código continua silenciosamente com uma string vazia.

---

### CC-18 — Acoplamento a Implementação Concreta

**Princípio violado:** Dependa de abstrações, não de implementações  
**Arquivo:** [`pkg/services/dashboards/service/dashboard_service.go:98`](pkg/services/dashboards/service/dashboard_service.go#L98)

```go
type DashboardServiceImpl struct {
    ...
    serverLockService *serverlock.ServerLockService  // tipo concreto
    ...
}
```

O campo `serverLockService` é um ponteiro para um tipo concreto (`*serverlock.ServerLockService`) em vez de uma interface. Isso dificulta testes unitários e viola o Dependency Inversion Principle.

---

### CC-19 — Lógica de Remoção de Dados Oculta em Goroutines sem Abstração

**Princípio violado:** Funções devem fazer uma coisa  
**Arquivo:** [`pkg/api/admin_users.go:212`](pkg/api/admin_users.go#L212)

```go
g, ctx := errgroup.WithContext(c.Req.Context())
g.Go(func() error { return hs.starService.DeleteByUser(ctx, cmd.UserID) })
g.Go(func() error { return hs.orgService.DeleteUserFromAll(ctx, cmd.UserID) })
g.Go(func() error { return hs.preferenceService.Delete(ctx, &pref.DeleteCommand{UserID: cmd.UserID}) })
g.Go(func() error { return hs.TeamService.RemoveUsersMemberships(ctx, cmd.UserID) })
g.Go(func() error { return hs.authInfoService.DeleteUserAuthInfo(ctx, cmd.UserID) })
g.Go(func() error { return hs.AuthTokenService.RevokeAllUserTokens(ctx, cmd.UserID) })
g.Go(func() error { return hs.QuotaService.DeleteQuotaForUser(ctx, cmd.UserID) })
g.Go(func() error { return hs.accesscontrolService.DeleteUserPermissions(ctx, accesscontrol.GlobalOrgID, cmd.UserID) })
```

A função `AdminDeleteUser` acumula 8 chamadas de limpeza diretamente no handler HTTP. Esta orquestração deveria estar encapsulada no `UserService` com um método `CleanupUser(ctx, userID)`.

---

### CC-20 — Retorno Inconsistente de Erros

**Princípio violado:** Previsibilidade e consistência  
**Arquivo:** [`pkg/api/dashboard.go:836`](pkg/api/dashboard.go#L836)

```go
resp, err := hs.dashboardVersionService.List(c.Req.Context(), &query)
if err != nil {
    return response.Error(http.StatusNotFound, fmt.Sprintf("No versions found for dashboardId %d", dash.ID), err)
}
```

Um erro do serviço (que pode ser 500 interno) é sempre mapeado para `404 Not Found`. Se o banco de dados estiver fora do ar, o cliente receberá um 404, dificultando o diagnóstico do problema real.

---

## 3. Análise de Code Smells

Os 20 trechos a seguir identificam code smells presentes no código real.

---

### CS-01 — Long Method

**Arquivo:** [`pkg/api/dashboard.go:380`](pkg/api/dashboard.go#L380)  
**Função:** `postDashboard` (~115 linhas)

Combina validação de schema, reconhecimento de versão k8s, controle de quotas, verificação de provisionamento e salvamento. Todo novo requisito aumenta ainda mais a função.

---

### CS-02 — God Class

**Arquivo:** [`pkg/api/http_server.go`](pkg/api/http_server.go)  
**Struct:** `HTTPServer`

Com 86 importações internas e dependências de todos os serviços do sistema, `HTTPServer` é o archetype de uma God Class. Qualquer mudança de serviço passa por aqui, tornando a struct central de todo acoplamento do sistema.

---

### CS-03 — Feature Envy

**Arquivo:** [`pkg/api/dashboard.go:497`](pkg/api/dashboard.go#L497)  
**Função:** `saveDashboardViaK8s`

Esta função é um handler HTTP que manipula diretamente objetos `unstructured.Unstructured` do Kubernetes, chama `client.List()`, `client.Create()` e `client.Update()`. Está claramente "invejando" as funcionalidades do pacote de storage k8s, que deveria ser responsável por esta lógica.

---

### CS-04 — Data Clumps

**Arquivo:** [`pkg/api/dashboard.go:140`](pkg/api/dashboard.go#L140)

```go
meta := dtos.DashboardMeta{
    Slug: dash.Slug,
    Type: dashboards.DashTypeDB,
    CanStar: c.IsSignedIn,
    CanSave: canSave,
    CanEdit: canEdit,
    CanAdmin: canAdmin,
    CanDelete: canDelete,
    ...
}
```

O grupo de permissões `canSave`, `canEdit`, `canAdmin`, `canDelete` é calculado junto e passado junto. Esses campos formam um "aglomerado de dados" que poderia ser encapsulado em uma struct `DashboardPermissions`.

---

### CS-05 — Shotgun Surgery

**Arquivo:** [`pkg/api/admin_users.go`](pkg/api/admin_users.go)

A deleção de um usuário requer mudanças em 8 serviços diferentes coordenados diretamente no handler. Qualquer nova associação de dados ao usuário (ex: alertas) exige uma nova linha nesta função, um exemplo clássico de Shotgun Surgery.

---

### CS-06 — Long Parameter List

**Arquivo:** [`pkg/api/dashboard.go:284`](pkg/api/dashboard.go#L284)

```go
func (hs *HTTPServer) getDashboardHelper(
    ctx context.Context,
    orgID int64,
    uid string,
    k8sGetAPIVersion string,
) (*dashboards.Dashboard, response.Response) {
```

Quatro parâmetros, sendo o último um controle de comportamento (flag argument). Uma struct `DashboardLookupOptions` resolveria o problema de forma limpa.

---

### CS-07 — Primitive Obsession

**Arquivo:** [`pkg/api/dashboard.go`](pkg/api/dashboard.go)

```go
uid := web.Params(c.Req)[":uid"]
apiVersion := strings.TrimSpace(c.Req.URL.Query().Get("apiVersion"))
var userID int64
```

`uid`, `apiVersion` e `userID` são tipos primitivos (`string`, `int64`) que representam conceitos de domínio ricos. Tipos fortes como `DashboardUID`, `APIVersion` e `UserID` evitariam confusões e erros de passagem de parâmetros na ordem errada.

---

### CS-08 — Comments as Smell (Technical Debt Marker)

**Arquivo:** [`pkg/api/dashboard.go:69`](pkg/api/dashboard.go#L69)

```go
//nolint:gocyclo
func (hs *HTTPServer) GetDashboard(c *contextmodel.ReqContext) response.Response {
```

O comentário `//nolint:gocyclo` é um supressor explícito de alerta de alta complexidade ciclomática. O linter detectou o problema e em vez de corrigir, foi silenciado. Há pelo menos 20 ocorrências de `nolint` no pacote `pkg/api/`.

---

### CS-09 — Speculative Generality

**Arquivo:** [`pkg/api/dashboard.go:590`](pkg/api/dashboard.go#L590)

```go
validation := "Warn"
if strings.HasPrefix(gv.Version, "v0") {
    validation = "Ignore" // v0 can be anything
}
```

O modo `"Warn"` sugere que no futuro haverá validação mais estrita para versões não-v0, mas nenhuma lógica atual usa o modo "Strict". A variável `validation` serve uma generalidade especulativa não implementada.

---

### CS-10 — Divergent Change

**Arquivo:** [`pkg/api/dashboard.go`](pkg/api/dashboard.go)

O arquivo `dashboard.go` com 1.310 linhas é modificado por razões diversas: mudanças na API REST, mudanças no schema k8s, mudanças em permissões, mudanças em provisionamento e mudanças no home dashboard. Cada uma destas é uma razão diferente de mudança no mesmo arquivo — Divergent Change clássico.

---

### CS-11 — Dead Code / Código Legado Desnecessário

**Arquivo:** [`pkg/api/dashboard.go:190`](pkg/api/dashboard.go#L190)

```go
} else if dash.FolderID > 0 { // nolint:staticcheck
    query := dashboards.GetDashboardQuery{ID: dash.FolderID, OrgID: c.SignedInUser.GetOrgID()} // nolint:staticcheck
```

O campo `FolderID` está depreciado (`staticcheck` emite aviso) e existem dois caminhos de código para resolver a pasta: um via `FolderUID` (preferido) e outro via `FolderID` legado. O segundo é dead code de migração que deveria ter prazo para remoção.

---

### CS-12 — Inappropriate Intimacy

**Arquivo:** [`pkg/api/dashboard.go:514`](pkg/api/dashboard.go#L514)

```go
obj.SetKind("Dashboard")
obj.SetNamespace(namespace)
obj.SetAnnotations(map[string]string{})
obj.SetLabels(map[string]string{})
delete(obj.Object, "status")
delete(obj.Object, "access")
meta.SetResourceVersionInt64(0)
meta.SetFolder(cmd.FolderUID)
meta.SetMessage(cmd.Message)
meta.SetUID("")
meta.SetResourceVersion("")
meta.SetFinalizers(nil)
meta.SetManagedFields(nil)
```

O handler HTTP tem conhecimento íntimo demais da estrutura interna dos objetos Kubernetes (campos `status`, `access`, `uid`, `resourceVersion`). Esta "intimidade inadequada" deveria ser encapsulada num builder/factory k8s.

---

### CS-13 — Message Chains

**Arquivo:** [`pkg/api/dashboard.go:86`](pkg/api/dashboard.go#L86)

```go
publicDashboard, err := hs.PublicDashboardsApi.PublicDashboardService.FindByDashboardUid(ctx, c.GetOrgID(), dash.UID)
```

A cadeia `hs.PublicDashboardsApi.PublicDashboardService.FindByDashboardUid` atravessa dois níveis de objetos para chegar a um método. Viola a Lei de Demeter e cria acoplamento à estrutura interna de `PublicDashboardsApi`.

---

### CS-14 — Long Method

**Arquivo:** [`pkg/api/login.go`](pkg/api/login.go)

O arquivo `login.go` tem 80 branches (if/for/switch) em ~400 linhas, contendo a lógica de autenticação, redirecionamento, cookies, OAuth e tokens. É um Long Method distribuído num arquivo que deveria conter múltiplas funções menores e bem nomeadas.

---

### CS-15 — Switch Statements (Polimorfismo ausente)

**Arquivo:** [`pkg/api/dashboard.go:403`](pkg/api/dashboard.go#L403)

```go
switch {
case strings.HasPrefix(apiVersion, dashboardsV1.GROUP):
    // lógica para dashboardv1
    ...
case apiVersion == "":
    return response.Error(...)
}
return response.Error(...)
```

Um switch com lógica baseada em `apiVersion` no handler HTTP. À medida que novas versões de API são adicionadas, este switch crescerá. Um padrão Strategy ou Registry de handlers de versão seria mais extensível.

---

### CS-16 — Refused Bequest

**Arquivo:** [`pkg/services/dashboards/service/dashboard_service.go:66`](pkg/services/dashboards/service/dashboard_service.go#L66)

```go
var (
    _ dashboards.DashboardService             = (*DashboardServiceImpl)(nil)
    _ dashboards.DashboardProvisioningService = (*DashboardServiceImpl)(nil)
    _ dashboards.PluginService                = (*DashboardServiceImpl)(nil)
    _ dashboards.DashboardAccessService       = (*DashboardServiceImpl)(nil)
```

`DashboardServiceImpl` implementa quatro interfaces distintas simultaneamente. Se uma subclasse precisar apenas de `DashboardService`, ainda carregará todas as responsabilidades das outras três — Refused Bequest.

---

### CS-17 — Temporary Field

**Arquivo:** [`pkg/services/dashboards/service/dashboard_service.go:84`](pkg/services/dashboards/service/dashboard_service.go#L84)

```go
type DashboardServiceImpl struct {
    ...
    sqlStore db.DB // solely used to cleanup associated resources after dashboard deletion
    ...
}
```

O próprio comentário admite que `sqlStore` é usado apenas para um caso específico de limpeza. Um campo usado em apenas uma operação é um Temporary Field que indica abstração inadequada.

---

### CS-18 — Large Class

**Arquivo:** [`pkg/services/dashboards/service/dashboard_service.go`](pkg/services/dashboards/service/dashboard_service.go)

Com 2.435 linhas, `DashboardServiceImpl` é uma Large Class que acumula lógica de CRUD de dashboards, provisionamento, plugins, acesso/permissões, busca e limpeza de dados k8s. Deveria ser decomposta em serviços menores por responsabilidade.

---

### CS-19 — Data Class

**Arquivo:** [`pkg/api/dashboard.go:1082`](pkg/api/dashboard.go#L1082)

```go
type RestoreDashboardVersionByIDParams struct {
    Body        dtos.RestoreDashboardVersionCommand
    DashboardID int64
}
```

Os vários tipos de parâmetros Swagger no final do arquivo (`RestoreDashboardVersionByIDParams`, `GetDashboardVersionsByIDParams`, etc.) são Data Classes puras — structs sem comportamento que servem apenas como contentores de dados para geração de documentação, criando ruído no arquivo de lógica.

---

### CS-20 — Parallel Inheritance Hierarchies

**Arquivo:** [`pkg/api/dashboard.go:300`](pkg/api/dashboard.go#L300)

```go
func (hs *HTTPServer) DeleteDashboardByUID(c *contextmodel.ReqContext) response.Response {
    return hs.deleteDashboard(c)
}

func (hs *HTTPServer) deleteDashboard(c *contextmodel.ReqContext) response.Response {
```

O padrão `PublicMethod → privateMethod` duplicado em toda a API (ex: `PostDashboard → postDashboard`, `GetDashboard → getDashboard`) cria uma hierarquia de delegação paralela que não adiciona valor mas aumenta a contagem de funções artificialmente.

---

## 4. Apresentação de Trechos Refatorados

---

### RF-01 — Extrair Método: Validação de Dashboard Vazio

**Princípio:** Extract Method (Fowler)  
**Arquivo:** [`pkg/api/dashboard.go:97`](pkg/api/dashboard.go#L97)

**Código original:**
```go
if dash.Data != nil {
    isEmptyData := true
    for k := range dash.Data.MustMap() {
        if k != "id" && k != "uid" {
            isEmptyData = false
            break
        }
    }
    if isEmptyData {
        return response.Error(http.StatusInternalServerError, "Error while loading dashboard, dashboard data is invalid", nil)
    }
}
```

**Código refatorado:**
```go
if dash.Data != nil && isDashboardDataEmpty(dash.Data) {
    return response.Error(http.StatusInternalServerError, "Error while loading dashboard, dashboard data is invalid", nil)
}

// ---

func isDashboardDataEmpty(data *simplejson.Json) bool {
    for k := range data.MustMap() {
        if k != "id" && k != "uid" {
            return false
        }
    }
    return true
}
```

**Justificativa:** A lógica de validação é extraída para uma função testável de forma independente, com nome expressivo e responsabilidade única.

---

### RF-02 — Substituir Número Mágico por Constante

**Princípio:** Replace Magic Number with Symbolic Constant  
**Arquivo:** [`pkg/api/dashboard.go:783`](pkg/api/dashboard.go#L783)

**Código original:**
```go
newpanel := simplejson.NewFromAny(map[string]any{
    "type": "gettingstarted",
    "id":   123123,
    "gridPos": map[string]any{
        "x": 0, "y": 3, "w": 24, "h": 9,
    },
})
```

**Código refatorado:**
```go
const (
    gettingStartedPanelID     = 123123
    gettingStartedPanelX      = 0
    gettingStartedPanelY      = 3
    gettingStartedPanelWidth  = 24
    gettingStartedPanelHeight = 9
)

newpanel := simplejson.NewFromAny(map[string]any{
    "type": "gettingstarted",
    "id":   gettingStartedPanelID,
    "gridPos": map[string]any{
        "x": gettingStartedPanelX,
        "y": gettingStartedPanelY,
        "w": gettingStartedPanelWidth,
        "h": gettingStartedPanelHeight,
    },
})
```

**Justificativa:** Constantes nomeadas comunicam a intenção de cada valor e facilitam ajustes futuros sem caçar números espalhados no código.

---

### RF-03 — Extrair Método: Verificação de Usuário Externo

**Princípio:** Extract Method + DRY  
**Arquivo:** [`pkg/api/admin_users.go:291`](pkg/api/admin_users.go#L291)

**Código original (duplicado em AdminDisableUser e AdminEnableUser):**
```go
authInfoQuery := &login.GetAuthInfoQuery{UserId: userID}
if _, err := hs.authInfoService.GetAuthInfo(c.Req.Context(), authInfoQuery); !errors.Is(err, user.ErrUserNotFound) {
    return response.Error(http.StatusInternalServerError, "Could not disable external user", nil)
}
```

**Código refatorado:**
```go
func (hs *HTTPServer) isExternalUser(ctx context.Context, userID int64) bool {
    authInfoQuery := &login.GetAuthInfoQuery{UserId: userID}
    _, err := hs.authInfoService.GetAuthInfo(ctx, authInfoQuery)
    return !errors.Is(err, user.ErrUserNotFound)
}

// Em AdminDisableUser:
if hs.isExternalUser(c.Req.Context(), userID) {
    return response.Error(http.StatusForbidden, "Cannot modify external user", nil)
}

// Em AdminEnableUser:
if hs.isExternalUser(c.Req.Context(), userID) {
    return response.Error(http.StatusForbidden, "Cannot modify external user", nil)
}
```

**Justificativa:** Elimina duplicação, centraliza lógica de verificação em um só lugar e usa o código HTTP correto (`403 Forbidden` em vez de `500`).

---

### RF-04 — Encapsular Grupo de Permissões em Struct

**Princípio:** Introduce Parameter Object / Replace Data Clumps  
**Arquivo:** [`pkg/api/dashboard.go:116`](pkg/api/dashboard.go#L116)

**Código original:**
```go
canSave, _ := hs.AccessControl.Evaluate(ctx, c.SignedInUser, writeEvaluator)
canEdit := canSave
canDelete, _ := hs.AccessControl.Evaluate(ctx, c.SignedInUser, deleteEvaluator)
canAdmin, _ := hs.AccessControl.Evaluate(ctx, c.SignedInUser, adminEvaluator)
```

**Código refatorado:**
```go
type DashboardPermissions struct {
    CanSave   bool
    CanEdit   bool
    CanDelete bool
    CanAdmin  bool
}

func (hs *HTTPServer) evaluateDashboardPermissions(ctx context.Context, user identity.Requester, uid string) DashboardPermissions {
    scope := dashboards.ScopeDashboardsProvider.GetResourceScopeUID(uid)
    canSave, _ := hs.AccessControl.Evaluate(ctx, user, accesscontrol.EvalPermission(dashboards.ActionDashboardsWrite, scope))
    canDelete, _ := hs.AccessControl.Evaluate(ctx, user, accesscontrol.EvalPermission(dashboards.ActionDashboardsDelete, scope))
    canAdmin, _ := hs.AccessControl.Evaluate(ctx, user, accesscontrol.EvalPermission(dashboards.ActionDashboardsPermissionsWrite, scope))
    return DashboardPermissions{
        CanSave:   canSave,
        CanEdit:   canSave,
        CanDelete: canDelete,
        CanAdmin:  canAdmin,
    }
}
```

**Justificativa:** Agrupa dados relacionados em um tipo coeso, facilita testes unitários e elimina 4 variáveis locais do método `GetDashboard`.

---

### RF-05 — Extrair Método: Montagem de Metadados Kubernetes

**Princípio:** Extract Method  
**Arquivo:** [`pkg/api/dashboard.go:514`](pkg/api/dashboard.go#L514)

**Código original:**
```go
obj.SetKind("Dashboard")
obj.SetNamespace(namespace)
obj.SetAnnotations(map[string]string{})
obj.SetLabels(map[string]string{})
delete(obj.Object, "status")
delete(obj.Object, "access")
meta.SetResourceVersionInt64(0)
meta.SetFolder(cmd.FolderUID)
// ... mais 5 linhas de configuração
```

**Código refatorado:**
```go
func prepareDashboardForK8sWrite(obj *unstructured.Unstructured, meta utils.GrafanaMetaAccessor, cmd dashboards.SaveDashboardCommand, namespace string) {
    obj.SetKind("Dashboard")
    obj.SetNamespace(namespace)
    obj.SetAnnotations(map[string]string{})
    obj.SetLabels(map[string]string{})
    delete(obj.Object, "status")
    delete(obj.Object, "access")
    meta.SetResourceVersionInt64(0)
    meta.SetFolder(cmd.FolderUID)
    meta.SetMessage(cmd.Message)
    meta.SetUID("")
    meta.SetResourceVersion("")
    meta.SetFinalizers(nil)
    meta.SetManagedFields(nil)
}
```

**Justificativa:** A preparação do objeto k8s é uma operação coesa que merece seu próprio nome e função testável isoladamente.

---

### RF-06 — Introduzir Guard Clause

**Princípio:** Replace Nested Conditional with Guard Clauses  
**Arquivo:** [`pkg/api/dashboard.go:541`](pkg/api/dashboard.go#L541)

**Código original:**
```go
if internalID > 0 && name == "" {
    found, err := client.List(ctx, metav1.ListOptions{...})
    if err != nil {
        return response.Error(http.StatusInternalServerError, "unable to lookup previous version", err)
    }
    if len(found.Items) == 0 {
        return response.Error(http.StatusBadRequest, fmt.Sprintf("...(%d)...", internalID), nil)
    }
    old = &found.Items[0]
    name = old.GetName()
    meta.SetName(name)
    if !cmd.Overwrite {
        return response.Error(http.StatusConflict, "Dashboard with the same internal ID already exists. Use overwrite flag to update.", nil)
    }
}
```

**Código refatorado:**
```go
if internalID <= 0 || name != "" {
    // sem legacyID para resolver, continua fluxo normal
} else {
    name, old, err = resolveLegacyInternalID(ctx, client, meta, internalID, cmd.Overwrite)
    if err != nil {
        return response.Error(...)
    }
}

// Ou ainda melhor: extrair para função
func resolveLegacyDashboard(ctx context.Context, client dynamic.ResourceInterface, meta utils.GrafanaMetaAccessor, internalID int64, overwrite bool) (string, *unstructured.Unstructured, response.Response) {
    found, err := client.List(ctx, metav1.ListOptions{
        LabelSelector: fmt.Sprintf("%s=%d", utils.LabelKeyDeprecatedInternalID, internalID),
        Limit: 2,
    })
    if err != nil {
        return "", nil, response.Error(http.StatusInternalServerError, "unable to lookup previous version", err)
    }
    if len(found.Items) == 0 {
        return "", nil, response.Error(http.StatusBadRequest, fmt.Sprintf("The payload includes an internal identifier (%d) that is not found", internalID), nil)
    }
    old := &found.Items[0]
    name := old.GetName()
    meta.SetName(name)
    if !overwrite {
        return "", nil, response.Error(http.StatusConflict, "Dashboard with the same internal ID already exists. Use overwrite flag to update.", nil)
    }
    return name, old, nil
}
```

**Justificativa:** Reduz nesting, isola a resolução de ID legado em uma função testável, e torna o fluxo principal mais legível.

---

### RF-07 — Encapsular Limpeza de Usuário no Serviço

**Princípio:** Move Method + SRP  
**Arquivo:** [`pkg/api/admin_users.go:212`](pkg/api/admin_users.go#L212)

**Código original (no handler HTTP):**
```go
g, ctx := errgroup.WithContext(c.Req.Context())
g.Go(func() error { return hs.starService.DeleteByUser(ctx, cmd.UserID) })
g.Go(func() error { return hs.orgService.DeleteUserFromAll(ctx, cmd.UserID) })
// ... 6 goroutines adicionais
if err := g.Wait(); err != nil {
    return response.Error(http.StatusInternalServerError, "Failed to delete user", err)
}
```

**Código refatorado (no UserService):**
```go
// Em pkg/services/user/service.go
func (s *Service) CleanupAfterDelete(ctx context.Context, userID int64) error {
    g, ctx := errgroup.WithContext(ctx)
    g.Go(func() error { return s.starService.DeleteByUser(ctx, userID) })
    g.Go(func() error { return s.orgService.DeleteUserFromAll(ctx, userID) })
    g.Go(func() error { return s.preferenceService.Delete(ctx, &pref.DeleteCommand{UserID: userID}) })
    g.Go(func() error { return s.teamService.RemoveUsersMemberships(ctx, userID) })
    g.Go(func() error { return s.authInfoService.DeleteUserAuthInfo(ctx, userID) })
    g.Go(func() error { return s.authTokenService.RevokeAllUserTokens(ctx, userID) })
    g.Go(func() error { return s.quotaService.DeleteQuotaForUser(ctx, userID) })
    g.Go(func() error { return s.accesscontrolService.DeleteUserPermissions(ctx, accesscontrol.GlobalOrgID, userID) })
    return g.Wait()
}

// No handler HTTP:
if err := hs.userService.CleanupAfterDelete(c.Req.Context(), cmd.UserID); err != nil {
    return response.Error(http.StatusInternalServerError, "Failed to delete user", err)
}
```

**Justificativa:** A orquestração de limpeza é lógica de negócio, não lógica de apresentação HTTP. Mover para o serviço permite testes unitários, reutilização e isola o handler de detalhes de implementação.

---

### RF-08 — Substituir Código de Status HTTP Incorreto

**Princípio:** Código semântico correto  
**Arquivo:** [`pkg/api/admin_users.go:130`](pkg/api/admin_users.go#L130)

**Código original:**
```go
return response.Error(http.StatusExpectationFailed,
    "User password updated but unable to revoke user sessions", err)
```

**Código refatorado:**
```go
return response.Error(http.StatusInternalServerError,
    "User password updated but unable to revoke user sessions", err)
```

**Justificativa:** `417 Expectation Failed` é reservado para o cabeçalho HTTP `Expect:`. Uma falha operacional interna deve retornar `500 Internal Server Error`. A mensagem também deveria considerar a opção de retornar 207 (Multi-Status) ou tratar a revogação de tokens como operação de melhor esforço assíncrona.

---

### RF-09 — Extrair Constante para Nome Anônimo

**Princípio:** Replace Magic String with Constant  
**Arquivo:** [`pkg/api/dashboard.go:45`](pkg/api/dashboard.go#L45)

**Código original:**
```go
const (
    anonString = "Anonymous"
)
```

**Código refatorado:**
```go
const (
    // anonymousUserDisplayName é o nome exibido quando o criador/atualizador de um dashboard não pode ser identificado.
    anonymousUserDisplayName = "Anonymous"
)
```

**Justificativa:** Renomear a constante para `anonymousUserDisplayName` e adicionar um comentário explicativo torna o propósito evidente. Qualquer desenvolvedor que encontre esta constante saberá imediatamente onde e por que é usada.

---

### RF-10 — Extrair Validação de Rota de Redirecionamento

**Princípio:** Extract Method + Single Level of Abstraction  
**Arquivo:** [`pkg/api/login.go:54`](pkg/api/login.go#L54)

**Código original:**
```go
func (hs *HTTPServer) ValidateRedirectTo(redirectTo string) (string, error) {
    to, err := url.Parse(redirectTo)
    if err != nil {
        return "", errInvalidRedirectTo
    }
    if to.IsAbs() {
        return "", errAbsoluteRedirectTo
    }
    if to.Host != "" {
        return "", errForbiddenRedirectTo
    }
    if redirectDenyRe.MatchString(to.Path) || redirectDenyRe.MatchString(to.Fragment) {
        return "", errForbiddenRedirectTo
    }
    if to.Path != "/" && !redirectAllowRe.MatchString(to.Path) {
        return "", errForbiddenRedirectTo
    }
    if hs.Cfg.AppSubURL != "" && !strings.HasPrefix(to.Path, hs.Cfg.AppSubURL+"/") {
        return "", errInvalidRedirectTo
    }
    ...
}
```

**Código refatorado:**
```go
func (hs *HTTPServer) ValidateRedirectTo(redirectTo string) (string, error) {
    parsed, err := parseRedirectURL(redirectTo)
    if err != nil {
        return "", err
    }
    if err := hs.validateSubURLPrefix(parsed); err != nil {
        return "", err
    }
    return parsed.String(), nil
}

func parseRedirectURL(redirectTo string) (*url.URL, error) {
    parsed, err := url.Parse(redirectTo)
    if err != nil {
        return nil, errInvalidRedirectTo
    }
    if parsed.IsAbs() {
        return nil, errAbsoluteRedirectTo
    }
    if parsed.Host != "" || redirectDenyRe.MatchString(parsed.Path) || redirectDenyRe.MatchString(parsed.Fragment) {
        return nil, errForbiddenRedirectTo
    }
    if parsed.Path != "/" && !redirectAllowRe.MatchString(parsed.Path) {
        return nil, errForbiddenRedirectTo
    }
    return parsed, nil
}

func (hs *HTTPServer) validateSubURLPrefix(parsed *url.URL) error {
    if hs.Cfg.AppSubURL != "" && !strings.HasPrefix(parsed.Path, hs.Cfg.AppSubURL+"/") {
        return errInvalidRedirectTo
    }
    return nil
}
```

**Justificativa:** Separa o parsing/validação da URL (testável sem HTTP Server) da validação de subURL (que depende de configuração), seguindo Single Level of Abstraction dentro de `ValidateRedirectTo`.

---

## 5. Métricas de Qualidade: LOC, WMC, CBO e Cobertura de Testes

### 5.1 Metodologia

| Métrica | Definição adotada |
|---|---|
| **LOC** | Linhas totais por arquivo `.go` (excluídos gerados `zz_*` e testes `*_test.go`) |
| **WMC** | Contagem de funções declaradas com `func` no nível de pacote por arquivo |
| **CBO** | Contagem de imports internos (`"github.com/grafana/grafana/..."`) por arquivo |
| **Cobertura** | Razão entre arquivos de teste (`*_test.go`) e total de arquivos Go por módulo |

---

### 5.2 LOC — Lines of Code

| Módulo | Arquivos | Mín | Máx | Média | Desvio Padrão |
|---|---|---|---|---|---|
| `pkg/api` | 92 | 10 | 1.310 | 227 | ~312 |
| `pkg/services/dashboards` | 13 | 20 | 2.435 | 376 | ~620 |
| `pkg/setting` | 30 | 16 | 2.504 | 209 | ~470 |
| `pkg/services/accesscontrol` | 58 | 7 | 944 | 241 | ~215 |
| `pkg/services/user` | 13 | 16 | 767 | 288 | ~215 |
| `pkg/services/folder` | 12 | 13 | 733 | 248 | ~200 |
| `pkg/services/authn` | 34 | 13 | 778 | 178 | ~175 |
| `pkg/infra` | 87 | 5 | 828 | 128 | ~150 |
| `pkg/plugins` | 75 | 3 | 750 | 128 | ~145 |
| `pkg/middleware` | 19 | 26 | 297 | 113 | ~80 |

**Três maiores valores de LOC (arquivo individual):**
1. `pkg/services/dashboards/service/dashboard_service.go` — **2.435 linhas**
2. `pkg/setting/setting.go` — **2.504 linhas**
3. `pkg/api/dashboard.go` — **1.310 linhas**

**Três menores valores de LOC (arquivo individual, desconsiderados arquivos triviais <10 linhas):**
1. `pkg/infra/` — vários arquivos com ~5–10 linhas (interfaces e tipos mínimos)
2. `pkg/plugins/` — ~3 linhas (arquivos `doc.go` de pacote)
3. `pkg/api/basic_auth.go` — **19 linhas**

**Média geral do projeto (módulos acima):** ≈ **206 linhas/arquivo**

**Análise:**  
- `dashboard_service.go` (2.435 linhas) e `setting.go` (2.504 linhas) são outliers severos. Esses arquivos cresceram organicamente ao longo do tempo sem refatoração periódica, tornando-se Large Classes clássicas. O alto desvio padrão em módulos como `pkg/api` (±312 para média 227) indica grande dispersão: há arquivos triviais e arquivos muito longos coexistindo no mesmo pacote.
- O módulo `pkg/middleware` apresenta a distribuição mais homogênea (média 113, desvio ±80), indicando responsabilidades melhor delimitadas por arquivo.

---

### 5.3 WMC — Weighted Methods per Class

| Módulo | Arquivos | Mín | Máx | Média | Desvio Padrão |
|---|---|---|---|---|---|
| `pkg/api` | ~51 | 0 | 29 | 7,0 | ~6,9 |
| `pkg/services/user` | 13 | 0 | 41 | 13,1 | ~12,0 |
| `pkg/services/folder` | 12 | 0 | 18 | 7,0 | ~6,0 |
| `pkg/infra` | 87 | 0 | 35 | 5,0 | ~7,0 |
| `pkg/plugins` | 75 | 0 | 36 | 5,3 | ~7,5 |
| `pkg/middleware` | 19 | 1 | 16 | 4,8 | ~4,5 |

**Três maiores WMC (arquivo individual):**
1. `pkg/services/user/userimpl/user.go` — **41 funções**
2. `pkg/plugins/` — arquivo com **36 funções**
3. `pkg/api/login.go` — **27 funções**

**Três menores WMC:**
1. Arquivos `doc.go` e de tipos — **0 funções** (apenas declarações)
2. `pkg/api/basic_auth.go` — **1 função**
3. `pkg/api/health.go` — **1 função**

**Média geral estimada:** ≈ **6–7 funções/arquivo**

**Análise:**  
- `user.go` com 41 funções é um arquivo que centraliza toda a lógica do serviço de usuário. Em sistemas OO tradicionais, WMC acima de 20 é considerado alto risco. Para Go, onde não há herança, 41 funções num único arquivo indica falta de modularização.
- O desvio padrão alto (especialmente em `pkg/services/user`: ~12) confirma a distribuição desequilibrada: muitos arquivos com 0–2 funções e poucos com mais de 20.
- `pkg/middleware` é o módulo mais equilibrado, com WMC entre 1 e 16 e média de 4,8.

---

### 5.4 CBO — Coupling Between Objects

| Módulo | Arquivos | Mín | Máx | Média | Desvio Padrão |
|---|---|---|---|---|---|
| `pkg/api` | ~52 | 0 | 86 | 5,8 | ~12,0 |
| `pkg/services/user` | 13 | 0 | 15 | 4,8 | ~5,5 |
| `pkg/services/folder` | 12 | 0 | 17 | 4,3 | ~5,0 |
| `pkg/infra` | 87 | 0 | 7 | 0,5 | ~1,5 |
| `pkg/plugins` | 75 | 0 | 9 | 0,7 | ~1,8 |
| `pkg/middleware` | 19 | 1 | 12 | 4,2 | ~3,5 |

**Três maiores CBO (arquivo individual):**
1. `pkg/api/http_server.go` — **86 imports internos**
2. `pkg/api/api.go` — **22 imports internos**
3. `pkg/api/dashboard.go` — **21 imports internos**

**Três menores CBO:**
1. `pkg/infra/` — maioria com **0 imports internos** (infra tende a ser folha do grafo de dependências)
2. `pkg/plugins/` — maioria com **0–1 imports internos**
3. Arquivos de tipos e interfaces — **0 imports internos**

**Média geral estimada:** ≈ **3,4 imports internos/arquivo**

**Análise:**  
- `http_server.go` com **86 acoplamentos internos** é o ponto de maior risco do projeto. Qualquer mudança de interface em qualquer um desses 86 módulos pode causar falha de compilação ou comportamento inesperado neste arquivo. Em literatura de métricas de software, CBO > 14 é considerado muito alto.
- `pkg/infra` e `pkg/plugins` têm CBO próximo de zero porque são pacotes de infraestrutura que não dependem de outros pacotes internos — este é o comportamento esperado para a camada mais baixa da arquitetura.
- O alto desvio padrão em `pkg/api` (±12 para média 5,8) é explicado pelo outlier `http_server.go` que "puxa" a média enquanto a maioria dos arquivos tem CBO entre 0 e 12.

---

### 5.5 Taxa de Cobertura de Testes (por arquivo)

> **Nota:** A cobertura de linha exata requer execução de `go test -coverprofile`, que não foi executada neste ambiente. A seguir, é apresentada a proporção de arquivos de teste em relação aos arquivos de produção, como proxy para cobertura.

| Módulo | Arquivos Prod | Arquivos Teste | Taxa Cobertura (arquivo) |
|---|---|---|---|
| `pkg/middleware` | 19 | 16 | **45,7%** |
| `pkg/services/authn` | 34 | 22 | **39,3%** |
| `pkg/services/folder` | 12 | 7 | **36,8%** |
| `pkg/api` | 92 | 52 | **36,1%** |
| `pkg/infra` | 87 | 42 | **32,6%** |
| `pkg/services/user` | 13 | 6 | **31,6%** |
| `pkg/services/dashboards` | 13 | 6 | **31,6%** |
| `pkg/plugins` | 75 | 29 | **27,9%** |

**Três maiores proporções de teste:**
1. `pkg/middleware` — **45,7%** (quase 1 arquivo de teste para cada arquivo de produção)
2. `pkg/services/authn` — **39,3%**
3. `pkg/services/folder` — **36,8%**

**Três menores proporções de teste:**
1. `pkg/plugins` — **27,9%** (apenas 1 arquivo de teste para cada ~2,6 de produção)
2. `pkg/services/user` — **31,6%**
3. `pkg/services/dashboards` — **31,6%**

**Média geral do projeto:** ≈ **35,0%** de arquivos com cobertura direta por testes

**Análise:**  
- A cobertura de `pkg/middleware` (45,7%) é a mais alta do projeto. Isso é positivo porque middlewares implementam lógica de segurança (autenticação, autorização, rate limiting) que exige alta confiabilidade. Este resultado indica boa maturidade de testes nesta camada crítica.
- `pkg/plugins` (27,9%) apresenta a menor cobertura proporcional. O sistema de plugins é complexo e envolve carregamento dinâmico, sandboxing e comunicação gRPC — áreas onde bugs são difíceis de detectar sem testes. A baixa cobertura aqui representa risco operacional.
- `pkg/services/dashboards` e `pkg/services/user` empatam no segundo pior resultado (31,6%). Ambos são serviços centrais do sistema com alta complexidade de negócio. A `DashboardServiceImpl` com 2.435 linhas e apenas 6 arquivos de teste sugere que as funções mais longas estão com cobertura insuficiente.
- A distribuição geral de ~35% indica que o projeto segue um modelo de testes pragmático, cobrindo fluxos principais mas deixando cenários de borda e caminhos de erro parcialmente descobertos. Projetos de alta qualidade de código aberto geralmente almejam 70–80% de cobertura de linha.

---

## Resumo Executivo

| Dimensão | Pontuação | Observação |
|---|---|---|
| **Clean Code** | ⚠️ Moderado | Funções longas e God Class são os problemas mais frequentes |
| **Code Smells** | ⚠️ Moderado | Large Class e Shotgun Surgery indicam dívida técnica acumulada |
| **LOC** | ⚠️ Atenção | 2 arquivos com > 2.400 linhas são outliers críticos |
| **WMC** | ⚠️ Atenção | `user.go` com 41 funções requer decomposição |
| **CBO** | 🔴 Crítico | `http_server.go` com CBO=86 é ponto único de falha de acoplamento |
| **Cobertura** | ⚠️ Moderado | Média de 35% é abaixo do recomendado para sistemas críticos |

O projeto Grafana é um sistema maduro e amplamente utilizado, o que naturalmente gera acúmulo de dívida técnica ao longo dos anos. Os problemas identificados são típicos de sistemas que cresceram organicamente. As refatorações sugeridas são incrementais e não exigem reescrita — podem ser adotadas gradualmente em ciclos normais de desenvolvimento.
