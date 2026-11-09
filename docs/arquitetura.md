# Arquitetura: Guardian AI

## Visão geral

O sistema tem dois fluxos independentes que se encontram no banco de dados:

1. **Ingestão (contínua):** cada mensagem dos grupos chega por webhook, é classificada pelo remetente (equipe ou cliente) e gravada.
2. **Análise (diária):** um agendamento lê as conversas do dia, envia ao LLM e transforma a resposta em relatório.

```mermaid
flowchart TB
    subgraph Ingestão
        W[API do WhatsApp] -->|webhook: nova mensagem| R[Fluxo de ingestão]
        R --> K{Remetente é da equipe?}
        K -->|sim| E[Marca como equipe]
        K -->|não| C[Marca como cliente]
        E --> DB[(Banco de dados)]
        C --> DB
    end
    subgraph Análise diária
        S[Agendamento ao fim do dia] --> L[Busca conversas do dia por grupo]
        DB --> L
        L --> P[Monta contexto por grupo]
        P --> M[LLM]
        M --> O[Ocorrências: tom rude, sem resposta, em aberto, pendência]
        O --> Q[Ordena por prioridade e urgência]
        Q --> X[E-mail ao gestor]
    end
```

## O que é regra e o que é IA

| Etapa | Tipo |
|---|---|
| Receber e gravar mensagens | Regra (webhook + banco) |
| Separar equipe x cliente | Regra (cadastro de contatos) |
| Detectar tom rude, pedidos em aberto, falta de resposta e pendências | IA (LLM) |
| Ordenar por prioridade e urgência e enviar o e-mail | Regra sobre a saída do LLM |

## Decisões técnicas

- **Análise em lote, uma vez por dia:** o gestor recebe um resumo único em vez de alertas soltos, e o custo de LLM fica previsível.
- **Separação equipe x cliente antes do LLM:** o modelo recebe a conversa já rotulada, o que deixa a análise mais simples de instruir ("o cliente ficou sem resposta?").
- **Mais de um provedor de LLM:** OpenAI e Claude disponíveis no mesmo fluxo.
- **Low-code (Retool):** banco, fluxos e agendamento na mesma plataforma, sem servidor próprio para manter.
