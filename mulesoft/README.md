# MuleSoft Study — Integração Salesforce → Supabase

Projeto de estudo em Mule 4 (parte do repositório [integracao-sales](..)) que integra o **Salesforce** ao **Supabase** de forma orientada a eventos:
quando uma Account é publicada no Salesforce via Platform Event, o Mule replica o registro no Supabase
e devolve o ID gerado para o Salesforce (sincronização de volta).

## Arquitetura

```
Salesforce                         Mule 4 app                              Supabase
─────────                         ──────────                              ────────
Platform Event            ┌─ Replay channel listener
SendAccount__e  ────────► │  (/event/SendAccount__e, ONLY_NEW)
                          ├─ Transform (DataWeave): campos __c → snake_case
                          ├─ HTTP POST /rest/v1/account  ─────────────────► tabela `account`
                          │◄───────────────────────────── retorna registro (id)
                          ├─ Transform: monta { CNPJ__c, Id_Supabase__c, Status_Integracao__c }
Account (upsert por CNPJ__c) ◄─ Salesforce Upsert
                          └─ Error handler: on-error-continue + log estruturado
```

## Fluxos

| Arquivo | Flow | Descrição |
|---|---|---|
| [receiveaccount-flow.xml](src/main/mule/receiveaccount-flow.xml) | `Receive Account` | Fluxo principal: escuta o evento `SendAccount__e`, insere a Account no Supabase e faz upsert no Salesforce (chave externa `CNPJ__c`) gravando `Id_Supabase__c` e `Status_Integracao__c = "Integrado"`. |
| [hw-flow.xml](src/main/mule/hw-flow.xml) | `hello-world-flow` | Endpoint de teste `POST /hellow` na porta `8081` que retorna um JSON fixo. Usado para validar o ambiente. |
| [global-configs.xml](src/main/mule/global-configs.xml) | — | Configurações globais: properties e `Supabase-Config` (HTTPS, timeout 15s). |
| [mule-soft-study.xml](src/main/mule/mule-soft-study.xml) | — | Arquivo base gerado pelo Studio (vazio). |

### Mapeamento Salesforce → Supabase

Os campos do Platform Event (`Campo__c`) são convertidos para `snake_case` no DataWeave. Exemplos:

| Evento Salesforce | Coluna Supabase |
|---|---|
| `Account_Name__c` | `account_name` |
| `CNPJ__c` | `cnpj` |
| `Annual_Revenue__c` | `annual_revenue` |
| `Billing_PostalCode__c` | `billing_postal_code` |
| `Status_da_Conta__c` | `status_da_conta` |

O mapeamento completo (27 campos: dados da conta, contato principal, próxima ação e endereços de cobrança/entrega)
está no transform do [receiveaccount-flow.xml](src/main/mule/receiveaccount-flow.xml).

### Tratamento de erros

Falhas no fluxo caem em `on-error-continue`, que registra em `ERROR` a mensagem, o tipo do erro, a descrição
e o corpo da resposta (quando existir), sem derrubar o listener.

## Contrato da API (RAML)

O rascunho do contrato `account-api` (`POST /account`, recebe `name`, `taxId`, `billingCity`, `phone`, `status`)
está em [api/account-api.raml](api/account-api.raml).

## Stack

- Mule Runtime **4.12.3**, Java **17**
- Mule Maven Plugin 4.10.0
- HTTP Connector 1.11.3
- Salesforce Connector 11.4.0 (OAuth 2.0 Client Credentials)

## Como rodar

1. Copie [config.properties.example](config.properties.example) para `src/main/mule/config.properties` e preencha os valores (Salesforce e Supabase)
   (esse arquivo está no `.gitignore` e **nunca** deve ser commitado).
2. Abra o projeto no Anypoint Studio / Anypoint Code Builder (VS Code).
3. Execute o app (`Run Mule application`) ou faça o build com `mvn clean package`.
4. Teste o ambiente: `curl -X POST http://localhost:8081/hellow`.
5. Para testar a integração, publique um evento `SendAccount__e` no Salesforce e acompanhe os logs.

## Configuração

| Propriedade | Descrição |
|---|---|
| `sfdc.client.id` | Consumer Key da Connected App do Salesforce |
| `sfdc.client.secret` | Consumer Secret da Connected App |
| `sfdc.token.url` | URL do token OAuth da org (`.../services/oauth2/token`) |
| `supabase.host` | Host do projeto Supabase (sem `https://`) |
| `supabase.apikey` | Chave de API do Supabase (usada em `apikey` e `Authorization: Bearer`) |

## Estrutura

```
├── src/main/mule/          # flows e configs
├── src/main/resources/     # log4j2, schemas, application-types
├── api/                    # contrato RAML da account-api
├── pom.xml                 # dependências e plugin Maven
├── mule-artifact.json      # versão mínima do runtime e do Java
└── config.properties.example
```
