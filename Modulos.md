# Documentação de Módulos ZyonERP

Este documento descreve, para cada módulo funcional, sua arquitetura, entidades envolvidas, serviços, endpoints, validações e telas no cliente .NET MAUI.

## Sumário
1. [Produtos](#produtos)
2. [Clientes](#clientes)
3. [Fornecedores](#fornecedores)
4. [Fiscal](#fiscal)
   - [Grupos de tributação](#grupos-de-tributação)
   - [Configurações fiscais](#configurações-fiscais)
   - [Notas fiscais](#notas-fiscais)
5. [PDV e Vendas](#pdv-e-vendas)
   - [PDV](#pdv)
   - [Caixa](#caixa)
   - [Vendas](#vendas)
6. [Integrações](#integrações)
   - [BrasilAPI](#brasilapi)
   - [ZeusFiscal](#zeusfiscal)
   - [SMTP / E-mail](#smtp--e-mail)
7. [Outros domínios relevantes](#outros-domínios-relevantes)

---

## Produtos

### Entidades e relacionamentos
- `Produto` (tabela `CadProduto_Base`): campos de identificação (código ref, barras, descrição, categorias), fiscais (NCM, CEST, CFOP, CSTs, alíquotas), preços (custo, venda, promoção, margem), estoque (atual, mínimo, máximo), características físicas e relacionamentos.
- `ProdutoMovimento` (`CadProduto_Movimentos`): histórico de entradas/saídas com quantidade, custo, estoque anterior/posterior.
- `ProdutoImagem` (`CadProduto_Imagens`): meta-informações sobre imagens vinculadas.
- Relacionamento opcional com `GrupoTributacao` (aplicação de defaults fiscais).

### DTOs principais
- `ProdutoCreateDto` / `ProdutoUpdateDto`: espelham principais campos com validações.
- `ProdutoListDto`: resumo para listagem (id, código, descrição, preço, status, estoque atual).
- `ProdutoDetailsDto`: inclui dados fiscais, estoque, tributação, imagens e últimos movimentos.

### Validações de negócio
- Código de referência obrigatório, mínimo 3 caracteres, único.
- Código de barras (quando informado) único.
- NCM com 8 dígitos numéricos; CEST 7 dígitos.
- Alíquotas (ICMS, PIS, COFINS, IPI, MVA, Redução) entre 0 e 100.
- Estoque máximo ≥ estoque mínimo; estoque não pode ficar negativo quando `PermiteVendaEstoqueNegativo=false`.
- Preço promocional/minimo ≤ preço de venda.
- Aplicação de defaults fiscais ao associar `GrupoTributacao`.

### Serviços
- `ProdutoService` (herda `BaseCrudService`):
  - Hooks `BeforeCreate`/`AfterCreate` para normalização e geração de movimento inicial.
  - `BeforeUpdate` ajusta estoque quando quantidade alterada, atualiza auditoria.
  - Métodos customizados: `GetProdutosEstoqueBaixoAsync`, `GetByCodigoBarrasAsync`, `GetByCodigoRefAsync`, `GetByCategoriaAsync`, `AtualizarEstoqueAsync`, `ProcessarMovimentacaoEstoqueBatchAsync`.

### Endpoints principais
- `GET /api/produto` (paginação), `GET /api/produto/{id}`
- `POST /api/produto`, `PUT /api/produto/{id}`, `DELETE` (soft delete)
- `PATCH /api/produto/{id}/ativar` / `desativar`
- `GET /api/produto/estoque-baixo`, `por-codigo-barras/{ean}`, `por-codigo-ref/{codigo}`, `por-categoria/{nome}`
- `GET /api/produto/{id}/movimentacoes`
- `POST /api/produto/{id}/atualizar-estoque`
- `POST /api/estoque/movimentos/batch` (batch)

### Telas MAUI
- `ProdutosPage.xaml`: lista com filtros (categoria, estoque baixo, status) e indicadores.
- `NovoProdutoViewModel` / `ProdutoFormPopup`: formulário com abas (Identificação, Fiscal, Valores, Estoque).
- `ProdutoDetalhesViewModel`: exibe preços, estoque, tributação e imagens.
- `ProdutosFiltrosPopupViewModel`: filtros adicionais (faixa de preços, status, data). 

---

## Clientes

### Entidades
- `Cliente` (`CadCliente_Base`): herda `PessoaBase`, possui endereço integrado, contatos, status, observações.
- `ClientePF`, `ClientePJ`: dados específicos (CPF, RG, data nascimento, CNPJ, inscrição estadual/municipal).
- Relacionamento com `Venda` (cliente pode possuir múltiplas vendas).

### DTOs
- `ClienteCreateDto` / `ClienteUpdateDto`: campos completos com endereço e contatos.
- `ClienteListDto`: resumo (nome, documento, e-mail, status, data criação).
- `ClienteDetailsDto`: expõe dados adicionais, telefones, endereços.

### Validações
- CPF/CNPJ obrigatórios conforme tipo, validados com algoritmos brasileiros.
- CPF/CNPJ únicos (não podem se repetir).
- E-mail principal obrigatório para notas fiscais; e-mails validados e únicos.
- Telefones (comercial, celular, residencial) aceitam apenas números com DDD.
- Endereço: se CEP informado, deve preencher logradouro, número, bairro, cidade, UF com 2 caracteres.
- Ao atualizar, CPF/CNPJ não podem ser alterados.

### Serviços
- `ClienteService`: CRUD padrão + busca avançada.
- Métodos: `GetPagedAsync`, `SearchAsync` (nome/documento/e-mail), `GetByCpfAsync`, `GetByCnpjAsync`, `AtivarAsync`, `DesativarAsync`.

### Endpoints
- `GET /api/cliente`, `/api/cliente/paged`, `/api/cliente/{id}`
- `GET /api/cliente/cpf/{cpf}`, `/cnpj/{cnpj}`, `/buscar?termo=`
- `POST /api/cliente`, `PUT /api/cliente/{id}`, `DELETE` (desativação)
- `PATCH /api/cliente/{id}/ativar`, `/desativar`

### Telas MAUI
- `ClientesPage.xaml`: grade com busca, filtros por status e exportação (quando habilitada).
- `ClienteFormViewModel`: formulário com alternância PF/PJ.
- `ClienteDetailsViewModel`: exibe dados completos e histórico resumido.

---

## Fornecedores

### Status atual
- Entidades (`Fornecedor`, `FornecedorPF`, `FornecedorPJ`) e DTOs completos estão implementados.
- Validators (`FornecedorCreate/UpdateValidator`) garantem unicidade de documento, e-mail, formato de CEP/telefone.
- Perfis AutoMapper (`FornecedorMappingProfile`) prontos.
- **Controller/Service**: ainda não incluídos. Ao adicionar o módulo, siga o padrão dos demais cadastros (CRUD + filtros).

### Telas MAUI
- `FornecedoresPage.xaml`: lista com status e busca.
- `FornecedorFormPopup.xaml`: formulário modal com abas "Identificação", "Contato", "Endereço", "Dados bancários".
- `FornecedorListItemViewModel`, `FornecedoresViewModel`: gerenciam carregamento e ações (aguardando implementação backend).

### Planejamento de endpoints
- Rota sugerida: `/api/fornecedor`
- Endpoints similares a clientes (incluindo filtros `buscar?termo=` e `/{id}/ativar`).

---

## Fiscal

### Grupos de tributação
- Entidade `GrupoTributacao`: definem defaults (alíquotas ICMS/PIS/COFINS/IPI, CSTs, origem, regime tributário).
- DTOs: `GrupoTributacaoCreate/Update/List/DetailsDto`.
- Serviço `GrupoTributarioService` provê filtros avançados e toggling de status.
- Endpoints: `/api/grupos-tributacao` com filtros `search`, `codigo`, `status`, `regime`, `origem`.
- UI: tela de gestão (lista com filtros, formulário popup) para atribuir grupos aos produtos.

### Configurações fiscais
- Entidade `ConfiguracaoFiscal`: guarda ambiente, CSC, séries, diretórios, parâmetros de emissão.
- `ConfiguracaoFiscalSerie`: séries individuais (modelo 55, 65 etc.).
- Serviços `IFiscalService`, `IConfiguracaoFiscalService`.
- Endpoints: `/api/fiscal/configuracoes/{empresaId}`, `/api/configuracoes-fiscais` (CRUD, upload certificado, incrementar número).
- UI: tela com abas "Ambiente", "Séries", "Certificado", "Integração".

### Notas fiscais
- Entidades `NotaFiscal`, `NotaFiscalItem`, `NotaFiscalEvento`, `NotaFiscalDocumento`.
- Serviço `FiscalService`: integra com ZeusFiscal, emissões, reprocessamento, envio de e-mail.
- Endpoints: `/api/fiscal/notas`, `/api/fiscal/notas/avulsa`, `/api/fiscal/notas/reprocessar`, `/api/fiscal/notas/{id}/documentos/...`.
- UI: listagem com status (autorizada, pendente, rejeitada), ações para download/exportação.

---

## PDV e Vendas

### PDV
- Interfaces `IPdvService`, DTOs `PdvProdutoConsultaDto`, `PdvVendaRequestDto`, `PdvPagamentoItemDto`, `PdvVendaResumoDto`.
- `PdvService` coordena busca de produtos/clientes, registro de venda (cria `Venda`, `VendaItem`, `VendaPagamento`, movimenta caixa, integra fiscal).
- Endpoints: `/api/vendas/pdv/produtos`, `/clientes`, `/vendas`, `/vendas/{id}/resumo`.
- UI: `PdvViewModel`, `PdvPagamentoViewModel`. Telas com busca instantânea, carrinho, resumo e finalização.

### Caixa
- Entidades: `CaixaPontoVenda`, `CaixaTurno`, `CaixaOperacao`, `CaixaSaldoDia` (mapeadas com auditoria e totais).
- `CaixaService`: abertura, reabertura, fechamento, operações, resumo por ponto de venda e turno.
- Endpoints: `/api/vendas/caixa/abrir`, `/reabrir`, `/fechar`, `/operacoes`, `/sangria`, `/suprimento`, `/resumo/...`.
- UI: `CaixaAberturaViewModel`, `CaixaFechamentoViewModel`, `CaixaMovimentacoesViewModel`, `CaixaTurnoViewModel`.

### Vendas
- Entidade `Venda` + `VendaItem`, `VendaPagamento`, `VendaFormaPagamento`.
- Serviços `IVendaService`, `VendaService`: CRUD orientado a PDV, aplicação de descontos, cancelamento, integração fiscal.
- Endpoints: `/api/vendas` com subrotas `itens`, `desconto`, `finalizar`, `cancelar`, `pagamentos`.
- UI: histórico de vendas (quando habilitado), relatórios no front (filtro por status, cliente, período).

---

## Integrações

### BrasilAPI
- Serviço `IBrasilApiService` (`BrasilApiService`): busca CEP (`/cep/v1/{cep}`) e CNPJ (`/cnpj/v1/{cnpj}`), com caching em memória.
- Controller `BrasilApiController` expõe `/api/brasilapi/cep/{cep}`, `/cnpj/{cnpj}` (sem autenticação).
- Aplicações: preenchimento automático de endereço (clientes/fornecedores) no MAUI.

### ZeusFiscal
- `IFiscalIntegrationService` implementado por `FiscalIntegrationService`.
- Responsável por enviar lotes de NFe/NFCe, consultar status, cancelar, emitir inutilizações.
- Configurado via `ZeusFiscalOptions` (ambiente, CSC, token, endpoint, certificado).
- Erros e retornos são tratados em `FiscalService` e refletidos nos DTOs de resposta.

### SMTP / E-mail
- `ISmtpClientFactory` (`MailKitSmtpClientFactory`) + `FiscalEmailService`.
- Utilizado para envio de DANFE/boletos e comunicação de notas para clientes.
- Configurações: `SmtpSettings` (host, porta, usuário, senha, SSL).

---

## Outros domínios relevantes

- **Usuários / Autenticação**:
  - `Usuario`, `LoginRequest`, `LoginResultDto`.
  - `UsuarioService` (hash de senha, autenticação, ativação/desativação) e `AuthController`.

- **Catálogo (Categoria/Grupo/Subgrupo/Marca)**:
  - Entidades simples com nome/descrição, relacionamentos opcionais.
  - Serviços `CatalogoServices` registram CRUD e prevalência de itens "Diversos" (seed).

- **Formas de pagamento**:
  - `FormaPagamento` define modalidade fiscal, flags (aceita troco, parcelamento, integra fiscal).
  - `FormaPagamentoService` permite ativar/desativar e obter apenas ativos.

- **Estoque**:
  - Operações bateladas via DTO `MovimentoEstoqueBatchDto` (lista de itens com tipo, quantidade, observação).
  - Serviço `ProdutoService.ProcessarMovimentacaoEstoqueBatchAsync` garante atomicidade com transações.
