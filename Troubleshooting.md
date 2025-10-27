# Troubleshooting ZyonERP

Este guia lista problemas comuns encontrados durante desenvolvimento, implantação ou uso do ZyonERP, com possíveis causas e soluções recomendadas.

## Sumário
1. [Backend / Banco de dados](#backend--banco-de-dados)
2. [Autenticação e segurança](#autenticação-e-segurança)
3. [Configuração fiscal e certificado digital](#configuração-fiscal-e-certificado-digital)
4. [Integração SEFAZ / ZeusFiscal](#integração-sefaz--zeusfiscal)
5. [Aplicativo .NET MAUI](#aplicativo-net-maui)
6. [PDV e caixa](#pdv-e-caixa)
7. [Ferramentas e debugging](#ferramentas-e-debugging)

---

## Backend / Banco de dados

| Sintoma | Possível causa | Como resolver |
| --- | --- | --- |
| `MySqlException: Unknown database 'ZyonERP'` | Banco não criado ou string de conexão incorreta. | Crie o banco manualmente (`CREATE DATABASE ZyonERP;`) e atualize `AppDbConnectionString`. |
| `Access denied for user` | Usuário/senha incorretos ou sem permissão. | Verifique credenciais; conceda privilégios (`GRANT ALL ON ZyonERP.* TO 'usuario'@'%';`). |
| `An item with the same key has already been added` ao aplicar migrations | Migration previamente aplicada ou snapshot divergente. | Remova migration duplicada (`dotnet ef migrations remove`) e gere novamente. |
| Aplicação não sobe após `docker-compose` | Porta em uso ou variável faltando. | Confirme portas 5000/5001 livres; verifique variáveis (`Jwt__Secret`). |
| `SSL connection error` (MySQL) | SSL forçado no servidor, client não configurado. | Acrescente `SslMode=None;` na string ou configure certificado cliente. |

### Dicas adicionais
- Ative logs de EF Core (`Microsoft.EntityFrameworkCore.Database.Command` nível `Information`) para depurar SQL.
- Utilize `dbContext.Database.Migrate()` manualmente em testes para garantir sincronização.

---

## Autenticação e segurança

| Sintoma | Causa provável | Solução |
| --- | --- | --- |
| `InvalidOperationException: As configurações de JWT não foram definidas` | `JwtSettings` incompletos no `appsettings`. | Preencha `Secret`, `Issuer`, `Audience`. Não use strings curtas. |
| `401 Unauthorized` após login | Token expirado ou não enviado. | Verifique header `Authorization`. Certifique-se de usar `Bearer {token}`. |
| `IDX10223: Lifetime validation failed` | Divergência grande de horário entre servidor e cliente. | Sincronize relógio ou ajuste `ClockSkew` (padrão zero). |
| Token expira antes do esperado | `ExpiresInMinutes` configurado baixo. | Ajuste no `appsettings`. |

---

## Configuração fiscal e certificado digital

| Sintoma | Possível causa | Como resolver |
| --- | --- | --- |
| Upload de certificado retorna "Formato inválido" | Arquivo não é `.pfx`/`.p12`. | Converta para PFX e tente novamente. |
| Mensagem "Certificado inválido ou senha incorreta" | Senha incorreta ou arquivo corrompido. | Valide em ferramentas externas (e.g., `certutil -p senha -dump arquivo.pfx`). |
| Validade expirada na tela | Certificado expirou. | Renove certificado com autoridade certificadora e reenvie. |
| NFe rejeitada por "CSC inválido" | CSC configurado incorretamente. | Verifique código e ID do token fornecidos pela SEFAZ. |
| NFCe autorizada em homologação, mas não em produção | Ainda configurado em ambiente de Homologação. | Alterar `Ambiente` para Produção e usar CSC/token de produção. |

---

## Integração SEFAZ / ZeusFiscal

| Sintoma | Possível causa | Solução |
| --- | --- | --- |
| Rejeição "Falha na comunicação" | Timeout na API ZeusFiscal ou internet instável. | Reenvie com conexão estável; ajuste timeout se necessário. |
| Rejeição "Erro 215 - Falha de Schema" | Campos fiscais incorretos (CFOP, CST, NCM). | Revise produto/grupo tributação; use notas da SEFAZ como referência. |
| `409 Conflict` ao emitir nota | Número da nota já utilizado. | Use endpoint de incremento para avançar numeração ou inutilize número anterior. |
| Email de nota não chega | SMTP não configurado ou bloqueado por firewall. | Verifique `SmtpSettings`, porta e credenciais; teste com ferramenta externa (Telnet/SMTP tester). |

---

## Aplicativo .NET MAUI

| Sintoma | Possível causa | Como resolver |
| --- | --- | --- |
| Tela em branco após login | API inacessível (URL incorreta). | Revise `conexao.json`, utilize menu Configuração para testar conexão. |
| Erro "System.Net.Http.HttpRequestException" | Timeout ou servidor offline. | Garanta que API esteja rodando; teste health check. |
| Build Android falha com erro de workload | Workload MAUI desatualizada. | `dotnet workload update maui maui-android`. |
| Não executa em dispositivos iOS | Projeto `Client` não configurado para iOS. | Adicionar destino iOS ou utilizar macOS com requisitos Apple. |
| Dados não persistem após login | Token não salvo (permissões de armazenamento). | Em Android, verifique permissões; no desktop, tokens ficam em `Preferences`. |

### Depuração
- Use `Debug.WriteLine` em ViewModels para acompanhar fluxo.
- Rode `adb logcat` (Android) ou `dotnet-dsrouter` para capturar logs.
- Em desktop, as mensagens aparecem na janela Output/Console.

---

## PDV e caixa

| Sintoma | Possível causa | Solução |
| --- | --- | --- |
| Não é possível finalizar venda (estoque insuficiente) | Produto controla estoque e saldo negativo não permitido. | Repor estoque via movimentação ou habilitar venda negativa (não recomendado). |
| Diferença de caixa ao fechar | Lançamentos manuais esquecidos ou vendas não finalizadas. | Revise lista de operações e finalize/cancele vendas pendentes antes do fechamento. |
| Valores duplicados em caixa | Operação registrada duas vezes. | Utilize endpoint de cancelamento/estorno (se implementado) ou ajuste manual com justificativa. |

---

## Ferramentas e debugging

- **Logs do backend**: por padrão são impressos no console. Ajuste `Logging` em `appsettings` para aumentar nível.
- **Swagger**: use `http://localhost:5000/swagger` para testar endpoints rapidamente.
- **Postman/Insomnia**: crie collection com variável de ambiente `{{baseUrl}}` e scripts de login automático.
- **MySQL Workbench**: monitore consultas e indices; utilize `SHOW ENGINE INNODB STATUS` em caso de deadlocks.
- **EF Core Logging**: adicione no `Program.cs` (`builder.Logging.AddConsole().AddFilter(...)`) para inspecionar SQL.