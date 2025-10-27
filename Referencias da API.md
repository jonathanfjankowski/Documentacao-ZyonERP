# Referência da API ZyonERP.Core

## Sumário
1. [Introdução](#introdução)
2. [Autenticação](#autenticação)
3. [Padrão de respostas](#padrão-de-respostas)
4. [Controllers e endpoints](#controllers-e-endpoints)
   - [AuthController (`/api/auth`)](#authcontroller)
   - [Produtos (`/api/produto`)](#produtos)
   - [Clientes (`/api/cliente`)](#clientes)
   - [Categorias (`/api/categoria`)](#categorias)
   - [Grupos (`/api/grupo`), Subgrupos (`/api/subgrupo`), Marcas (`/api/marca`)](#catalogos)
   - [Grupos de Tributação (`/api/grupos-tributacao`)](#grupos-de-tributação)
   - [Empresas (`/api/company`)](#empresas)
   - [Fornecedores (`/api/fornecedor`)](#fornecedores)
   - [Integração BrasilAPI (`/api/brasilapi`)](#brasilapi)
   - [Formas de pagamento (`/api/formas-pagamento`)](#formas-de-pagamento)
   - [Estoque (`/api/estoque`)](#estoque)
   - [PDV (`/api/vendas/pdv`)](#pdv)
   - [Caixa (`/api/vendas/caixa`)](#caixa)
   - [Vendas (`/api/vendas`)](#vendas)
   - [Configurações fiscais (`/api/fiscal/configuracoes`)](#configurações-fiscais)
   - [Configuração fiscal detalhada (`/api/configuracoes-fiscais`)](#configuração-fiscal-detalhada)
   - [Notas fiscais (`/api/fiscal/notas`)](#notas-fiscais)
5. [Códigos de status e erros comuns](#códigos-de-status-e-erros-comuns)

---

## Introdução

- **Base URL**: `http://<host>:5000/api`
- **Autenticação**: JWT Bearer. Inclua o header `Authorization: Bearer {token}` em requisições protegidas.
- **Formato**: JSON. Todas as respostas seguem o wrapper `ApiResponse` (ou `PagedApiResponse`).

## Autenticação

1. Realize `POST /api/auth/login` com credenciais válidas.
2. Receba `accessToken`, `expiresAt`, `tokenType`.
3. Inclua `Authorization: Bearer {accessToken}` em todas as requisições subsequentes.

Tokens expiram conforme `JwtSettings:ExpiresInMinutes` (default 60 minutos). Não há refresh token automático; o cliente deve solicitar um novo login quando expirar.

## Padrão de respostas

```json
{
  "success": true,
  "message": "Mensagem opcional",
  "data": { ... },
  "errors": null
}
```

Para resultados paginados (`GET .../paged` ou `GetAll` customizado), a API utiliza:

```json
{
  "success": true,
  "message": "...",
  "items": [ ... ],
  "totalCount": 123,
  "pageNumber": 1,
  "pageSize": 20
}
```

### Endpoints CRUD padrão

Controllers derivados de `BaseCrudController` oferecem automaticamente as rotas abaixo (substitua `:base` pela rota do controller):

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/:base` | Lista registros (pode aceitar filtros personalizados na sobrescrita). |
| `GET` | `/:base/paged?pageNumber=1&pageSize=20` | Lista paginada (pageSize máximo 100). |
| `GET` | `/:base/{id}` | Busca registro por ID. |
| `GET` | `/:base/{id}/exists` | Verifica existência. |
| `GET` | `/:base/count` | Total de registros. |
| `POST` | `/:base` | Cria novo registro. |
| `PUT` | `/:base/{id}` | Atualiza registro existente (ID na URL deve coincidir com o corpo). |
| `DELETE` | `/:base/{id}` | Soft delete ou remoção conforme serviço. |

Alguns controllers acrescentam endpoints extras (`PATCH` para ativar/desativar, buscas específicas, operações complexas). Esses estão detalhados a seguir.

---

## Controllers e endpoints

### AuthController

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `POST` | `/api/auth/login` | `{ "nomeUsuario": "admin", "senha": "123" }` | Autentica e retorna token JWT (`LoginResultDto`). |
| `GET` | `/api/auth/health` | - | Health check público (sem autenticação). |
| `GET` | `/api/auth` | Header `Bearer` | Lista usuários (`UsuarioListDto`). |
| `GET` | `/api/auth/paged` | Header `Bearer` | Lista paginada de usuários. |
| `GET` | `/api/auth/{id}` | Header `Bearer` | Detalhes do usuário. |
| `POST` | `/api/auth` | `UsuarioCreateDto` | Cria usuário. |
| `PUT` | `/api/auth/{id}` | `UsuarioUpdateDto` | Atualiza usuário existente. |
| `DELETE` | `/api/auth/{id}` | - | Remove usuário. |
| `PATCH` | `/api/auth/{id}/ativar` | - | Ativa usuário. |
| `PATCH` | `/api/auth/{id}/desativar` | - | Desativa usuário. |

> **Observação:** O login e o health check não exigem token. Todos os demais endpoints do controller requerem autorização.

### Produtos

Base: `/api/produto`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `GET` | `/api/produto` | Query `page`, `pageSize` | Lista paginada (override controla paginação em `ProdutoController`). |
| `GET` | `/api/produto/paged` | Query `pageNumber`, `pageSize` | Alternativa de paginação padrão. |
| `GET` | `/api/produto/{id}` | - | Detalhes (`ProdutoDetailsDto`). |
| `POST` | `/api/produto` | `ProdutoCreateDto` | Cria produto (gera movimento de estoque inicial se estoque > 0). |
| `PUT` | `/api/produto/{id}` | `ProdutoUpdateDto` | Atualiza produto (ajusta estoque e tributação). |
| `DELETE` | `/api/produto/{id}` | - | Soft delete (desativação lógica). |
| `PATCH` | `/api/produto/{id}/ativar` | - | Ativa produto. |
| `PATCH` | `/api/produto/{id}/desativar` | - | Desativa produto. |
| `GET` | `/api/produto/estoque-baixo` | - | Lista produtos com estoque abaixo do mínimo. |
| `GET` | `/api/produto/por-codigo-barras/{codigo}` | - | Busca por código de barras (EAN). |
| `GET` | `/api/produto/por-codigo-ref/{codigo}` | - | Busca por código de referência. |
| `GET` | `/api/produto/por-categoria/{nome}` | - | Busca por categoria literal. |
| `GET` | `/api/produto/{id}/movimentacoes` | Query `dataInicio`, `dataFim` | Histórico de movimentos (`ProdutoMovimento`). |
| `POST` | `/api/produto/{id}/atualizar-estoque` | `{ "quantidade": 10, "tipoMovimento": "Entrada", "observacao": "..." }` | Movimenta estoque do produto. |
| `POST` | `/api/produto/{id}` | - | (não usado) |

### Clientes

Base: `/api/cliente`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `GET` | `/api/cliente` | Query `page`, `pageSize` | Lista paginada de clientes. |
| `GET` | `/api/cliente/paged` | Query `pageNumber`, `pageSize` | Paginação padrão. |
| `GET` | `/api/cliente/{id}` | - | Detalhes. |
| `POST` | `/api/cliente` | `ClienteCreateDto` | Cria cliente (valida CPF/CNPJ, e-mails, endereços). |
| `PUT` | `/api/cliente/{id}` | `ClienteUpdateDto` | Atualiza dados (CPF/CNPJ imutáveis). |
| `DELETE` | `/api/cliente/{id}` | - | Desativa cliente. |
| `PATCH` | `/api/cliente/{id}/ativar` | - | Ativa. |
| `PATCH` | `/api/cliente/{id}/desativar` | - | Desativa. |
| `GET` | `/api/cliente/cpf/{cpf}` | - | Busca por CPF (apenas dígitos). |
| `GET` | `/api/cliente/cnpj/{cnpj}` | - | Busca por CNPJ (apenas dígitos). |
| `GET` | `/api/cliente/buscar` | Query `termo`, `page`, `pageSize` | Busca inteligente por nome/documento/e-mail. |

### Categorias

Base: `/api/categoria` (CRUD padrão).

Nenhum endpoint adicional além dos herdados da classe base.

### Grupos, Subgrupos, Marcas {#catalogos}

Rotas:
- `/api/grupo`
- `/api/subgrupo`
- `/api/marca`

Todos seguem o CRUD padrão. Use `GetAll`/`Create`/`Update`/`Delete` conforme necessário.

### Grupos de Tributação

Base: `/api/grupos-tributacao` (`/api/grupos-tributarios` alternativa)

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/api/grupos-tributacao` | Lista com filtros opcionais: `page`, `pageSize`, `search`, `codigo`, `status`, `ativo`, `regimeTributario`, `origem`. |
| Demais | CRUD padrão (`{id}`, `paged`, `count`, etc.). |
| `PATCH` | `/api/grupos-tributacao/{id}/status` | Alterna status ativo/inativo. |

### Empresas

Base: `/api/company`

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/api/company` | Lista paginada (query `page`, `pageSize`). |
| `GET` | `/api/company/cnpj/{cnpj}` | Busca por CNPJ. |
| `POST` | `/api/company` | Cria empresa (`EmpresaCreateDto`). |
| `PUT` | `/api/company/{id}` | Atualiza empresa. |
| `DELETE` | `/api/company/{id}` | Desativação lógica. |
| `PATCH` | `/api/company/{id}/ativar` | Ativa. |
| `PATCH` | `/api/company/{id}/desativar` | Desativa. |

### Fornecedores

**Estado atual**: o domínio (`Fornecedor`, DTOs, validadores, mapeamentos) está implementado, porém o controller e serviços correspondentes não foram incluídos nesta branch. Consequentemente, as rotas `*/api/fornecedor*` não estão disponíveis no backend no momento. Consulte `docs/MODULES.md` para entender a estrutura de dados e como adicionar o controller quando o módulo for priorizado.

### BrasilAPI

Base: `/api/brasilapi` (sem autenticação obrigatória)

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/api/brasilapi/cep/{cep}` | Consulta CEP (formato `99999999` ou com máscara). |
| `GET` | `/api/brasilapi/cnpj/{cnpj}` | Consulta CNPJ (somente dígitos). |

### Formas de pagamento

Base: `/api/formas-pagamento`

| Método | Caminho | Descrição |
| --- | --- | --- |
| Padrão | CRUD completo (`Get`, `Get/{id}`, `Post`, `Put`, `Delete`). |
| `GET` | `/api/formas-pagamento/ativas` | Lista apenas formas ativas. |
| `PATCH` | `/api/formas-pagamento/{id}/status` | Alterna ativo/inativo. |

### Estoque

Base: `/api/estoque`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `POST` | `/api/estoque/movimentos/batch` | `MovimentoEstoqueBatchDto` | Processa lote transacional de movimentos (todos os itens são confirmados ou nenhum é aplicado). |

### PDV

Base: `/api/vendas/pdv`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `GET` | `/api/vendas/pdv/produtos?termo=` | - | Busca produtos para PDV (consulta rápida com filtros). |
| `GET` | `/api/vendas/pdv/clientes?documento=` | - | Localiza cliente por CPF/CNPJ. |
| `POST` | `/api/vendas/pdv/vendas` | `PdvVendaRequestDto` | Registra venda completa (itens, pagamentos, caixa). |
| `GET` | `/api/vendas/pdv/vendas/{id}/resumo` | - | Retorna resumo consolidado da venda. |

### Caixa

Base: `/api/vendas/caixa` (alias `/api/caixa`)

| Método | Caminho | Descrição |
| --- | --- | --- |
| `POST` | `/abrir` | Abre caixa (turno) — `CaixaAberturaDto`. |
| `POST` | `/reabrir` | Reabre turno fechado — `CaixaReaberturaDto`. |
| `POST` | `/fechar` | Fecha turno — `CaixaFechamentoDto`. |
| `POST` | `/operacoes` | Registra operação genérica (suprimento/sangria/manual). |
| `GET` | `/resumo/ponto-venda/{caixaPontoVendaId}` | Resumo por ponto de venda. |
| `GET` | `/resumo/turno/{caixaTurnoId}` | Resumo por turno. |
| `GET` | `/{id}` | Detalhes do turno. |
| `GET` | `/aberto?operadorId=&caixaPontoVendaId=` | Turno aberto vigente. |
| `POST` | `/{id}/sangria` | Sangria específica (`CaixaSangriaDto`). |
| `POST` | `/{id}/suprimento` | Suprimento específico (`CaixaSuprimentoDto`). |
| `GET` | `/{id}/operacoes` | Operações do turno. |
| `GET` | `/{id}/relatorio/fechamento` | Relatório de fechamento (`CaixaFechamentoResumoDto`). |
| `GET` | `/dashboard` | Indicadores do caixa (quando habilitado no serviço). |

> Todos os endpoints de caixa validam model state e retornam erros de validação detalhados (`ApiResponse.Fail` com lista de mensagens).

### Vendas

Base: `/api/vendas`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `POST` | `/api/vendas` | `CreateVendaDto` | Cria venda (persistida com status pendente). |
| `POST` | `/api/vendas/{id}/itens` | `CreateItemVendaDto` | Adiciona item. |
| `PUT` | `/api/vendas/{id}/itens/{itemId}` | `UpdateItemVendaDto` | Atualiza item. |
| `DELETE` | `/api/vendas/{id}/itens/{itemId}` | - | Remove item. |
| `POST` | `/api/vendas/{id}/desconto` | `AplicarDescontoDto` | Aplica desconto total. |
| `POST` | `/api/vendas/{id}/finalizar` | `FinalizarVendaDto` | Finaliza (gera pagamentos, vincula caixa). |
| `POST` | `/api/vendas/{id}/cancelar` | `CancelarVendaDto` | Cancela venda (registra motivo, estorna operações). |
| `GET` | `/api/vendas/{id}` | - | Detalhes (`VendaDto`). |
| `GET` | `/api/vendas/{id}/pagamentos` | - | Pagamentos vinculados. |
| `GET` | `/api/vendas/{id}/itens` | - | Itens da venda. |
| `GET` | `/api/vendas` | Query `page`, `pageSize`, `status`, `clienteId`, etc. (implementado no serviço). |

### Configurações fiscais

Base: `/api/fiscal/configuracoes`

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/api/fiscal/configuracoes/{empresaId}` | Retorna `ConfiguracaoFiscalDto` para empresa. |
| `PUT` | `/api/fiscal/configuracoes/{empresaId}` | Atualiza configuração (certificado, séries, CSC, parâmetros). |

### Configuração fiscal detalhada

Base: `/api/configuracoes-fiscais`

| Método | Caminho | Descrição |
| --- | --- | --- |
| `GET` | `/api/configuracoes-fiscais?empresaId=` | Busca por empresa. |
| `GET` | `/api/configuracoes-fiscais/{id}` | Busca por ID. |
| `POST` | `/api/configuracoes-fiscais` | Cria `ConfiguracaoFiscal`. |
| `PUT` | `/api/configuracoes-fiscais/{id}` | Atualiza configuração existente. |
| `POST` | `/api/configuracoes-fiscais/upload-certificado` | Upload de certificado PFX (`multipart/form-data`) com campos `certificado`, `senha`, `empresaId`. Retorna informações (thumbprint, validade) e caminho armazenado. |
| `POST` | `/api/configuracoes-fiscais/{id}/incrementar-nfe?serie=` | Incrementa número da NFe. |
| `POST` | `/api/configuracoes-fiscais/{id}/incrementar-nfce?serie=` | Incrementa número da NFCe. |

> Esses endpoints coexistem com o controller anterior para garantir compatibilidade retroativa.

### Notas fiscais

Base: `/api/fiscal/notas`

| Método | Caminho | Corpo | Descrição |
| --- | --- | --- | --- |
| `GET` | `/api/fiscal/notas` | Query (`empresaId`, `status`, `tipoNota`, `page`, `pageSize`, datas) | Consulta paginada (`NotaFiscalResumoDto`). |
| `GET` | `/api/fiscal/notas/{notaId}/documentos/{tipoDocumento}` | - | Download de XML/PDF (retorna `FileContentResult`). |
| `POST` | `/api/fiscal/notas/reprocessar?empresaId=` | - | Reprocessa notas pendentes (retorna quantidade). |
| `POST` | `/api/fiscal/notas/avulsa` | `NotaFiscalAvulsaDto` | Emite nota avulsa (gera `NotaFiscalOperacaoResultadoDto`). |
| `POST` | `/api/fiscal/notas/{notaId}/documentos/email` | `NotaFiscalEnvioDocumentoDto` | Envia XML/DANFE por e-mail. |

---

## Códigos de status e erros comuns

| Código | Significado | Observações |
| --- | --- | --- |
| `200 OK` | Sucesso. | Para operações de consulta/atualização. |
| `201 Created` | Recurso criado. | `Location` header presente quando aplicável. |
| `400 Bad Request` | Validação ou parâmetros inválidos. | Corpo contém `errors` com mensagens amigáveis. |
| `401 Unauthorized` | Token ausente ou inválido. | Refaça login. |
| `403 Forbidden` | Usuário autenticado sem permissão (não utilizado na versão atual). |
| `404 Not Found` | Recurso não encontrado. | Verifique IDs. |
| `409 Conflict` | Conflitos de negócio (ex.: duplicidade). | Retornado por alguns serviços. |
| `500 Internal Server Error` | Erro inesperado. | Consultar logs do servidor (`ILogger`). |

### Mensagens frequentes

- **"Token inválido" / `Unauthorized`**: verifique `Issuer`, `Audience` e `Secret` configurados na API e no cliente.
- **"Erro de validação"**: confira o array `errors`. A API sempre descreve qual campo/regra falhou.
- **"Não é possível excluir ... existem movimentos associados"**: `ProdutoDeleteValidator` bloqueia exclusão física; use desativação.
- **"Configuração fiscal não encontrada"**: garanta que a empresa possua registro em `/api/configuracoes-fiscais` antes de emitir notas.
