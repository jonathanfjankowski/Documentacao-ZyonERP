# ZyonERP

> Plataforma ERP modular construída com .NET 8 e .NET MAUI que automatiza cadastros, vendas, PDV e requisitos fiscais brasileiros em uma arquitetura em camadas.

## Visão geral

O ZyonERP é composto por um backend RESTful, uma biblioteca de domínio compartilhada e aplicativos .NET MAUI para desktop e dispositivos móveis. Os módulos de produtos, clientes, fiscal, PDV e integrações trabalham de forma coordenada para sustentar operações de varejo com foco em obrigações fiscais brasileiras (NFe/NFCe, SEFAZ, tributação) e fluxo de caixa de PDV.

### Principais funcionalidades

- Cadastro completo de produtos (estoque, preços, imagens, dados fiscais e tributários).
- Gestão de clientes (PF/PJ) com validações de documentos, endereços e status.
- Controle de fornecedores e relacionamento bancário (estrutura de dados pronta para integração).
- Grupos de tributação, configurações fiscais e emissão/gestão de notas fiscais (NFe/NFCe).
- PDV com abertura/fechamento de caixa, movimentações, formas de pagamento e integração com vendas.
- Integrações com BrasilAPI (CEP/CNPJ) e serviços fiscais externos (ZeusFiscal).
- Aplicações MAUI com MVVM, navegação centralizada, consumo seguro da API e suporte multi-plataforma.

### Tecnologias principais

| Camada | Tecnologias |
| --- | --- |
| Backend (`ZyonERP.Core`) | .NET 8, ASP.NET Core Web API, Entity Framework Core (Pomelo MySQL), AutoMapper, FluentValidation, JWT, Swagger/OpenAPI |
| Domínio compartilhado (`ZyonERP.Shared`) | Entidades auditáveis, DTOs, enums, interfaces e contratos de serviços |
| Frontend (`ZyonERP.Client` / `Client`) | .NET MAUI, CommunityToolkit.MVVM, NavigationService, ApiService centralizado, Shell/Popups, SecureStorage |
| Integrações | BrasilAPI, ZeusFiscal, SMTP, Health Checks, Logging estruturado |

## Estrutura da solução

| Projeto | Tipo | Responsabilidades principais | Dependências |
| --- | --- | --- | --- |
| `ZyonERP.Core` | ASP.NET Core Web API | Controllers, serviços de domínio, integrações externas, migrations, autenticação | Referencia `ZyonERP.Shared` |
| `ZyonERP.Shared` | Biblioteca de classes | Modelos de domínio, DTOs, enums, interfaces, respostas padrão | Independente |
| `ZyonERP.Client` | .NET MAUI | Aplicativo principal (Windows/Android), MVVM, navegação, consumo de API | Referencia `ZyonERP.Shared` |
| `Client` | .NET MAUI | Variante multiplataforma com estrutura equivalente ao cliente principal | Referencia `ZyonERP.Shared` |
| `ZyonERP.Manager` | Utilitários | Scripts e ferramentas administrativas secundárias | Opcional |
| `ZyonERP.Tests` | xUnit | Testes automatizados do backend e serviços de domínio | Referencia `ZyonERP.Core` e `ZyonERP.Shared` |

## Início rápido

### 1. Pré-requisitos

- .NET 8 SDK (`8.0.100` ou superior).
- MySQL 8 (ou MariaDB compatível) com um banco dedicado para o ZyonERP.
- Workloads .NET MAUI instaladas (`dotnet workload install maui maui-windows maui-android`).
- Ferramentas opcionais: Visual Studio 2022 17.8+, MySQL Workbench/CLI, Postman/Insomnia.

### 2. Backend

```bash
# Clonar e restaurar
git clone <url-do-repositorio>
cd ZyonERP
dotnet restore ZyonERP.sln

# Configurar a string de conexão em ZyonERP.Core/appsettings.json
# ou exportar variáveis ConnectionStrings__AppDbConnectionString, Jwt__*, ZeusFiscalOptions__*, etc.

# Aplicar migrations
dotnet ef database update --project ZyonERP.Core --startup-project ZyonERP.Core

# Executar a API (porta padrão 5000)
dotnet run --project ZyonERP.Core
```

- Swagger disponível em `http://localhost:5000/swagger` (apenas em Development).
- Health check público: `http://localhost:5000/api/auth/health`.

### 3. Frontend (.NET MAUI)

1. Ajuste `ZyonERP.Client/conexao.json` com o endpoint da API (ex.: `http://localhost:5000/api`).
2. Execute:

```bash
# Windows
dotnet maui run --project ZyonERP.Client -f net8.0-windows10.0.19041.0

# Android (emulador/dispositivo conectado)
dotnet maui run --project ZyonERP.Client -f net8.0-android
```

Alternativamente utilize o Visual Studio (F5) e selecione o destino desejado.

### 4. Testes automatizados

```bash
dotnet test ZyonERP.Tests
```