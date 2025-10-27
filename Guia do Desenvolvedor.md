# Guia do Desenvolvedor ZyonERP

## Sumário
1. [Visão geral](#visão-geral)
2. [Setup de ambiente](#setup-de-ambiente)
   - [Pré-requisitos](#pré-requisitos)
   - [Clonagem e restauração](#clonagem-e-restauração)
   - [Configuração do banco](#configuração-do-banco)
   - [Executando o backend](#executando-o-backend)
   - [Executando o cliente .NET MAUI](#executando-o-cliente-net-maui)
3. [Banco de dados e migrations](#banco-de-dados-e-migrations)
4. [Estrutura de código e convenções](#estrutura-de-código-e-convenções)
   - [Nomenclatura](#nomenclatura)
   - [Organização de arquivos](#organização-de-arquivos)
   - [Padrões de código](#padrões-de-código)
   - [Logs e tratamento de erros](#logs-e-tratamento-de-erros)
5. [Fluxo para novas funcionalidades](#fluxo-para-novas-funcionalidades)
   - [Camada Shared](#camada-shared)
   - [Backend (ZyonERP.Core)](#backend-zyonerpcore)
   - [.NET MAUI (ZyonERP.Client)](#net-maui-zyonerpclient)
6. [Testes](#testes)
   - [Backend](#backend)
   - [Frontend](#frontend)
7. [Boas práticas de git e CI](#boas-práticas-de-git-e-ci)
8. [Scripts úteis](#scripts-úteis)
9. [Checklist antes do deploy](#checklist-antes-do-deploy)

---

## Visão geral

O repositório `ZyonERP.sln` contém uma solução .NET 8 multiprojeto. O backend (`ZyonERP.Core`) expõe uma API REST em ASP.NET Core, persistindo dados em MySQL via EF Core. A biblioteca `ZyonERP.Shared` define modelos compartilhados (entidades, DTOs, enums, interfaces). Os aplicativos `.NET MAUI` (`ZyonERP.Client` e `Client`) consomem a API, aplicando o padrão MVVM com CommunityToolkit.

---

## Setup de ambiente

### Pré-requisitos

- **Sistema operacional**: Windows, Linux ou macOS (para MAUI, Windows/macOS com requisitos adicionais).
- **.NET 8 SDK** (`8.0.100` ou superior).
- **MySQL 8** (ou MariaDB compatível) com usuário que possua permissão para criar banco e tabelas.
- **Ferramentas opcionais**:
  - Visual Studio 2022 17.8+ com workloads *Desenvolvimento .NET* e *Desenvolvimento .NET Desktop*.
  - `dotnet-ef` (`dotnet tool install --global dotnet-ef`).
  - MySQL Workbench ou cliente CLI.
  - Postman/Insomnia.
  - Android SDK + emulador (para testes mobile).

### Clonagem e restauração

```bash
git clone <url-do-repositorio>
cd ZyonERP
dotnet restore ZyonERP.sln
```

### Configuração do banco

1. Crie um banco vazio (`CREATE DATABASE ZyonERP CHARACTER SET utf8mb4;`).
2. Ajuste `ZyonERP.Core/appsettings.json` (ou `appsettings.Development.json`):

```json
"ConnectionStrings": {
  "AppDbConnectionString": "Server=localhost;Port=3306;Database=ZyonERP;User=root;Password=<senha>;"
}
```

3. Opcionalmente defina variáveis de ambiente:

```bash
export ConnectionStrings__AppDbConnectionString="Server=localhost;Database=ZyonERP;User=root;Password=senha;"
export Jwt__Secret="segredo-super-seguro"
export Jwt__Issuer="ZyonERP"
export Jwt__Audience="ZyonERPClient"
```

### Executando o backend

```bash
dotnet ef database update --project ZyonERP.Core --startup-project ZyonERP.Core
dotnet run --project ZyonERP.Core
```

- Swagger: `http://localhost:5000/swagger`
- Health check: `http://localhost:5000/api/auth/health`

### Executando o cliente .NET MAUI

1. Configure `ZyonERP.Client/conexao.json`:

```json
{
  "BaseUrl": "http://localhost:5000/api",
  "TimeoutSeconds": 30
}
```

2. Rode conforme plataforma:

```bash
# Windows
dotnet maui run --project ZyonERP.Client -f net8.0-windows10.0.19041.0

# Android
dotnet maui run --project ZyonERP.Client -f net8.0-android
```

---

## Banco de dados e migrations

### Comandos principais

```bash
# Criar migration
dotnet ef migrations add NomeDaMigration --project ZyonERP.Core --startup-project ZyonERP.Core

# Aplicar migrations
dotnet ef database update --project ZyonERP.Core --startup-project ZyonERP.Core

# Reverter migration
dotnet ef database update MigracaoAnterior --project ZyonERP.Core --startup-project ZyonERP.Core

# Remover última migration (não aplicada)
dotnet ef migrations remove --project ZyonERP.Core --startup-project ZyonERP.Core

# Gerar script SQL
dotnet ef migrations script --project ZyonERP.Core --startup-project ZyonERP.Core
```
---

## Estrutura de código e convenções

### Nomenclatura

- Classes, métodos, propriedades: **PascalCase**.
- Variáveis locais, parâmetros, campos privados: **camelCase** (prefixo `_` para campos privados). Ex.: `_logger`, `_dbContext`.
- Interfaces: prefixo `I` (`IProdutoService`).
- Enums: nomes singulares (`StatusVenda`, `TipoPessoa`). Valores com PascalCase (`StatusVenda.Pendente`).
- DTOs: sufixos `CreateDto`, `UpdateDto`, `ListDto`, `DetailsDto`.

### Organização de arquivos

- Agrupe por domínio: `Services/Entidades`, `Validators/Fiscal`, `Views/Cadastros/Produtos`.
- Evite arquivos gigantes; extraia classes auxiliares ou partials quando necessário.
- Mantenha nomes de arquivos idênticos ao tipo principal.

### Padrões de código

- Controllers devem retornar `ApiResponse`/`PagedApiResponse`; evite retornar entidades diretamente.
- Use injeção de dependência (construtores) para todos os serviços.
- Validações de negócio adicionais devem residir na camada de serviço ou nos validadores.
- Utilize `ILogger<T>` com mensagens contextualizadas (`_logger.LogInformation("Produto {ProdutoId}", id);`).
- No MAUI, ViewModels devem herdar de `BaseViewModel` e utilizar `[ObservableProperty]`/`[RelayCommand]` (ou métodos equivalentes).

### Logs e tratamento de erros

- Use níveis apropriados: `LogInformation` para eventos esperado, `LogWarning` para estados anormais, `LogError` para exceções.
- Em controllers, capture exceções específicas (`ValidationException`, `InvalidOperationException`, `KeyNotFoundException`) antes de `Exception` genérica.
- Não exponha detalhes sensíveis em mensagens de erro; forneça guias amigáveis.
- No MAUI, utilize `try/catch` com `DisplayAlert` ou `Snackbar` para feedback ao usuário.

---

## Fluxo para novas funcionalidades

### Camada Shared

1. **Entidades** (`ZyonERP.Shared/Models/...`):
   - Herde de `EntidadeBase` ou outras bases apropriadas.
   - Configure data annotations básicas (comprimento, tipos), mas finalize no `AppDbContext`.
2. **DTOs** (`ZyonERP.Shared/DTOs/...`):
   - Crie `List`, `Details`, `Create`, `Update`, reutilizando bases (`AuditableActivatableListDto`).
   - Inclua apenas informações necessárias para cada cenário (evite expor entidades completas).
3. **Enums/Constantes**: adicione novos valores em `Shared/Enums`.
4. **Interfaces** (`Shared/Interfaces`): defina contratos de serviço quando necessário (ex.: `INovoModuloService`).

### Backend (ZyonERP.Core)

1. **DbContext**:
   - Adicione `DbSet<T>`.
   - Configure mapeamentos em `OnModelCreating` (índices, relacionamentos, tipos). Use Owned types onde aplicável.
2. **AutoMapper**:
   - Crie `MappingProfile` no namespace correspondente e adicione à DI (`AddAutoMapper`).
3. **Validators**:
   - Adicione classes em `Validators/<Domínio>` herdando de `BaseCreateValidator`/`BaseUpdateValidator`.
   - Registre assembly via `AddValidatorsFromAssemblyContaining<T>()`.
4. **Serviços**:
   - Crie classe em `Services/<Domínio>` (geralmente herdando de `BaseCrudService`).
   - Sobrescreva hooks (`BeforeCreate`, `AfterUpdate`) para regras específicas.
   - Registre no DI em `Services/<Domínio>/ServiceCollectionExtensions` (ou similar).
5. **Controllers**:
   - Herde de `BaseCrudController` quando possível.
   - Documente endpoints com comentários XML (aproveitados pelo Swagger).
6. **Migrations**:
   - Gere migration e aplique (ver seção anterior).
7. **Seeds (opcional)**:
   - Caso necessite dados iniciais, atualize `Seeders/AppDbContextSeedExtensions`.
8. **Testes**:
   - Adicione testes em `ZyonERP.Tests` (ver seção testes).

### .NET MAUI (ZyonERP.Client)

1. **DTO/Model**: use DTOs de `Shared` quando possível. Crie modelos específicos apenas para dados de apresentação.
2. **ApiService**: adicione métodos wrappers para novos endpoints (GET/POST/PUT/etc.).
3. **ViewModel**:
   - Herde de `BaseViewModel`.
   - Utilize `ObservableCollection` para listas e `ICommand` para ações.
   - Injete `ApiService`, `NavigationService`, `DialogService` (quando existente).
4. **View (XAML)**:
   - Crie página em `Views/<Domínio>` e associe ao ViewModel.
   - Respeite estilos globais (App.xaml).
5. **DI e Navegação**:
   - Registre ViewModel/View em `MauiProgram.cs` (`builder.Services.AddTransient<NewPage>()`).
   - Adicione rota no `AppShell`. Use `NavigationService.RegisterRoute()` se aplicável.
6. **Testes de UI/ViewModel** (quando aplicável).

---

## Testes

### Backend

- Projeto: `ZyonERP.Tests` (xUnit).
- Dependências: `Moq`, `AutoMapper`, `EF Core InMemory`.
- Recomendações:
  - Use banco InMemory com nome único por teste (`UseInMemoryDatabase(Guid.NewGuid().ToString())`).
  - Configure AutoMapper com perfis reais (`new MapperConfiguration(cfg => cfg.AddProfile<...>())`).
  - Mock serviços externos (`IFiscalIntegrationService`, `IBrasilApiService`).
  - Teste regras de negócio críticas (validações, hooks, transações).

```bash
dotnet test ZyonERP.Tests
```

### Frontend

- Priorize testes de ViewModel utilizando MSTest/xUnit + `Moq` ou `NSubstitute`.
- Mock `ApiService` para simular respostas e cenários offline.
- Teste comandos (`RelayCommand`) e estados (`IsBusy`, `Errors`).
- Para testes de UI automatizados (opcional), considere `UITest` ou Playwright com MAUI (em evolução).

---

## Scripts úteis

| Objetivo | Script |
| --- | --- |
| Executar API em modo hot-reload | `dotnet watch run --project ZyonERP.Core` |
| Rodar MAUI Android com log detalhado | `dotnet build ZyonERP.Client -f net8.0-android -c Debug` e `adb logcat` |
| Publicar backend (Release) | `dotnet publish ZyonERP.Core -c Release -o ./publish` |
| Publicar MAUI Windows (MSIX) | `dotnet publish ZyonERP.Client -c Release -f net8.0-windows10.0.19041.0` |
| Publicar MAUI Android (AAB) | `dotnet publish ZyonERP.Client -c Release -f net8.0-android` |
| Limpando artefatos | `dotnet clean ZyonERP.sln && find . -name bin -o -name obj -type d -exec rm -rf {} +` |

---

## Checklist antes do deploy

1. **Configurações**
   - `appsettings.Production.json` com strings seguras.
   - JWT Secret longo (>32 caracteres).
   - CORS restrito para domínios confiáveis.
2. **Banco**
   - Migrations aplicadas (`Database.Migrate()` roda ao iniciar, mas valide manualmente).
   - Backup do banco antes da atualização.
3. **Infraestrutura**
   - Certificado digital válido e carregado.
   - SMTP configurado para envio de notas.
   - Integração ZeusFiscal com credenciais corretas.
4. **Aplicativos**
   - Build Release realizado e assinado (Windows MSIX, Android AAB/APK).
   - `conexao.json` apontando para endpoint de produção.
5. **Monitoramento**
   - Configure logs persistentes (Serilog/Application Insights) se necessário.
   - Defina alertas para falhas de emissão fiscal, erros de banco, indisponibilidade.