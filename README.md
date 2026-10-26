# Integração Salesforce → Supabase

Integração orientada a eventos entre o **Salesforce** e o **Supabase**, construída com **MuleSoft (Mule 4)**.
Quando uma Account é publicada no Salesforce por um Platform Event, o Mule replica o registro no Supabase
e devolve o ID gerado ao Salesforce, marcando a conta como integrada.

O lado Salesforce (app GTech Vendas, leads e contas B2B) vive em
[processo-vendas](https://github.com/gspedev/processo-vendas).

## Como funciona

```
Salesforce (SendAccount__e)
        │ Platform Event
        ▼
Mule 4 ── Replay channel listener
        ├─ DataWeave: campos __c → snake_case (27 campos)
        ├─ HTTP POST → Supabase (/rest/v1/account)
        ├─ DataWeave: monta { CNPJ__c, Id_Supabase__c, Status_Integracao__c }
        └─ Upsert no Salesforce (chave externa CNPJ__c)
```

## O que foi feito

- Flow `Receive Account`: escuta o evento, transforma, grava no Supabase e atualiza a Account no Salesforce.
- Autenticação no Salesforce via OAuth 2.0 Client Credentials.
- Sincronização de volta: `Id_Supabase__c` e `Status_Integracao__c = "Integrado"`, com upsert por CNPJ.
- Tratamento de erro com `on-error-continue` e log estruturado (tipo, descrição e corpo da resposta).
- Endpoint de teste `POST /hellow` para validar o ambiente.
- Rascunho de contrato RAML da `account-api`.
- Credenciais fora do código: tudo via `config.properties` (ignorado pelo git), com modelo em `config.properties.example`.

## Estrutura

```
├── mulesoft/        # aplicação Mule (flows, configs, pom, RAML)
│   └── README.md    # detalhes técnicos, mapeamento de campos e como rodar
└── README.md
```

Veja [mulesoft/README.md](mulesoft/README.md) para o detalhamento.

## Stack

Mule Runtime 4.12.3 · Java 17 · HTTP Connector · Salesforce Connector · DataWeave 2 · Supabase (REST)

## Branches

- `main`: versão estável
- `dev`: desenvolvimento
