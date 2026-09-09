# PZaaS - Serviço 03: Disponibilidade e Estoque

Microsserviço responsável por receber pedidos, calcular os insumos necessários (Bill of Materials) e realizar a validação e baixa atômica no estoque do ecossistema Pizza as a Service (PZaaS)[cite: 2].

## Arquitetura e Tecnologias

* **Orquestrador:** n8n (Gerenciamento de webhooks, lógica de negócio e integrações).
* **Banco de Dados:** PostgreSQL. As tabelas respeitam o isolamento do projeto utilizando o prefixo da dupla (`turmaa28_estoque`)[cite: 2].
* **Resiliência:** Operação em duas instâncias redundantes (**Estoque A** e **Estoque B**) para garantir suporte ao mecanismo de Fallback do sistema[cite: 2]. 
* **Observabilidade:** Envio de logs estruturados de forma assíncrona para o Serviço 09 (Logger), garantindo a rastreabilidade via `x-pedido-id` sem impactar o tempo de resposta da API[cite: 2].
* **Chamador Direto:** API Gateway e Orquestrador de Pedidos (Serviço 05)[cite: 2].

## Lógica de Processamento (Fluxo n8n)

### 1. Mapeamento de Insumos (B.O.M)
O payload recebido do Orquestrador é interceptado por um nó JavaScript que converte as pizzas solicitadas nos ingredientes primários necessários, calculando as quantidades absolutas baseadas na matriz do pedido:

```javascript
const body = $('Payload do Orquestrador').first().json.body || $input.first().json;
const itens = body.itens || [];
const pedidoId = body.pedido_id || "PEDIDO-SEM-ID";

let insumosNecessarios = {};

for (const item of itens) {
    const pizzaId = item.pizza_id;
    const qtdPedida = Number(item.qtd) || 1;
    let receita = [];

    if (pizzaId === 'marguerita') {
        receita = [
            { id: 'massa_tradicional', qtd: 1 * qtdPedida },
            { id: 'queijo_mussarela', qtd: 300 * qtdPedida }
        ];
    } else if (pizzaId === 'calabresa') {
        receita = [
            { id: 'massa_tradicional', qtd: 1 * qtdPedida },
            { id: 'queijo_mussarela', qtd: 150 * qtdPedida },
            { id: 'calabresa_fatiada', qtd: 250 * qtdPedida }
        ];
    } else if (pizzaId === 'mussarela') {
        receita = [
            { id: 'massa_tradicional', qtd: 1 * qtdPedida },
            { id: 'queijo_mussarela', qtd: 300 * qtdPedida }
        ];
    }

    for (const r of receita) {
        insumosNecessarios[r.id] = (insumosNecessarios[r.id] || 0) + r.qtd;
    }
}

return { json: { pedido_id: pedidoId, insumos: insumosNecessarios } };
```

### 2. Transação Atômica no Banco de Dados
Para evitar concorrência e baixas parciais de estoque, a comunicação com o PostgreSQL utiliza *Common Table Expressions* (CTEs). A query valida se todos os insumos possuem saldo antes de executar o `UPDATE`, retornando o status final da operação:

```sql
WITH itens_mapeados AS (
    SELECT key AS id_insumo, value::numeric AS qtd_necessaria
    FROM json_each_text('{{ JSON.stringify($json.insumos) }}'::json)
),
verificacao AS (
    SELECT 
        i.id_insumo,
        COALESCE(e.quantidade_atual, 0) AS disponivel,
        i.qtd_necessaria,
        (COALESCE(e.quantidade_atual, 0) >= i.qtd_necessaria) AS tem_saldo
    FROM itens_mapeados i
    LEFT JOIN turmaa28_estoque e ON e.id_insumo = i.id_insumo
),
analise AS (
    SELECT 
        CASE 
            WHEN COUNT(*) = 0 OR NOT BOOL_AND(tem_saldo) THEN 'sem_estoque'
            ELSE 'disponivel'
        END AS status_estoque
    FROM verificacao
),
atualizacao AS (
    UPDATE turmaa28_estoque e
    SET quantidade_atual = e.quantidade_atual - i.qtd_necessaria
    FROM itens_mapeados i
    WHERE e.id_insumo = i.id_insumo 
      AND (SELECT status_estoque FROM analise) = 'disponivel'
    RETURNING e.id_insumo
)
SELECT status_estoque FROM analise;
```

## Contrato da API

### Headers Globais
Todas as requisições devem incluir obrigatoriamente[cite: 2]:
* `Content-Type: application/json`[cite: 2]
* `x-api-key: turma2026`[cite: 2]
* `x-pedido-id: <id-do-pedido>`[cite: 2]

### Endpoints

#### `GET /health`
Endpoint isolado para monitoramento de disponibilidade da instância[cite: 2].

**Response (200 OK)**
```json
{
  "status": "UP",
  "servico": "estoque-A"
}
```

#### `POST /validar-baixa`
Recebe as pizzas do pedido, valida o estoque e realiza o abatimento atômico.

**Request Payload:**
```json
{
  "pedido_id": "PZ-12345",
  "itens": [
    { "pizza_id": "calabresa", "qtd": 2 }
  ]
}
```

**Respostas e Tratamento de Erros Padronizados:**
O microsserviço cobre integralmente a árvore de falhas definida na especificação da arquitetura[cite: 2].

* **`200 OK` (Baixa Realizada):** Todos os insumos disponíveis, estoque atualizado[cite: 2].
* **`200 OK` (Fora de Estoque):** Pedido negado por regra de negócio. Retorna `"status": "sem_estoque"`.
* **`400 Bad Request`:** Disparado pelo nó de validação inicial do n8n caso o array de `itens` ou o `pedido_id` estejam ausentes[cite: 2].
* **`401 Unauthorized`:** Header `x-api-key` inválido ou ausente[cite: 2]. O fluxo é abortado antes da consulta ao banco de dados.
* **`500 Internal Server Error`:** Capturado pela saída de erro do nó PostgreSQL em caso de falha de conexão ou erro sintático na query, prevenindo interrupção silenciosa[cite: 2].
