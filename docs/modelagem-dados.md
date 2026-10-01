# Sistema de Controle de Pedidos — Doceria da Marta

## 02 · Modelagem do Sistema


## 1. Modelo conceitual do domínio

O domínio gira em torno do **Pedido**. Um cliente faz pedidos; cada pedido tem itens (produtos), pagamentos, uma possível entrega e um histórico de status.

```mermaid
flowchart TD
    Cliente -->|faz| Pedido
    Pedido -->|contém| ItemPedido
    ItemPedido -->|refere-se a| Produto
    Produto -->|pertence a| Categoria
    Pedido -->|recebe| Pagamento
    Pedido -->|pode ter| Entrega
    Pedido -->|registra| HistoricoStatus
    Cliente -->|possui| Endereco
    Usuario -->|executa ações em| Pedido
```

---

## 2. Modelo Entidade-Relacionamento (ER)

```mermaid
erDiagram
    USUARIO ||--o{ PEDIDO : "registra"
    USUARIO ||--o{ HISTORICO_STATUS : "altera"
    USUARIO ||--o{ PAGAMENTO : "registra"
    CLIENTE ||--o{ ENDERECO : "possui"
    CLIENTE ||--o{ PEDIDO : "faz"
    CATEGORIA ||--o{ PRODUTO : "classifica"
    PRODUTO ||--o{ VARIACAO_PRODUTO : "tem"
    PEDIDO ||--|{ ITEM_PEDIDO : "contem"
    PRODUTO ||--o{ ITEM_PEDIDO : "e vendido em"
    VARIACAO_PRODUTO ||--o{ ITEM_PEDIDO : "especifica"
    PEDIDO ||--o{ PAGAMENTO : "recebe"
    PEDIDO ||--o| ENTREGA : "tem"
    ENDERECO ||--o{ ENTREGA : "destino de"
    PEDIDO ||--|{ HISTORICO_STATUS : "registra"

    USUARIO {
        uuid id PK
        string nome
        string email UK
        string senha_hash
        string perfil "PROPRIETARIA, ATENDENTE, CONFEITEIRO, ENTREGADOR"
        boolean ativo
        timestamp criado_em
    }
    CLIENTE {
        uuid id PK
        string nome
        string telefone
        string email
        text observacoes
        boolean anonimizado
        timestamp criado_em
    }
    ENDERECO {
        uuid id PK
        uuid cliente_id FK
        string apelido
        string logradouro
        string numero
        string complemento
        string bairro
        string cidade
        string uf
        string cep
        text referencia
    }
    CATEGORIA {
        uuid id PK
        string nome UK
        boolean ativa
    }
    PRODUTO {
        uuid id PK
        uuid categoria_id FK
        string nome
        text descricao
        decimal preco_base
        string unidade_venda "UNIDADE, CENTO, KG, FATIA"
        boolean sob_encomenda
        int antecedencia_min_horas
        boolean ativo
    }
    VARIACAO_PRODUTO {
        uuid id PK
        uuid produto_id FK
        string descricao "ex: 1kg, recheio ninho"
        decimal preco
        boolean ativa
    }
    PEDIDO {
        uuid id PK
        int numero UK "sequencial amigavel"
        uuid cliente_id FK
        uuid criado_por FK
        string tipo "RETIRADA, ENTREGA"
        string status
        timestamp data_hora_entrega
        decimal subtotal
        decimal desconto
        decimal taxa_entrega
        decimal total
        string situacao_pagamento "PENDENTE, SINAL_PAGO, PAGO"
        text observacoes
        text motivo_cancelamento
        timestamp criado_em
        timestamp atualizado_em
    }
    ITEM_PEDIDO {
        uuid id PK
        uuid pedido_id FK
        uuid produto_id FK
        uuid variacao_id FK
        string nome_produto_snapshot
        decimal preco_unitario "congelado (RN06)"
        decimal quantidade
        decimal total_item
        text observacao
    }
    PAGAMENTO {
        uuid id PK
        uuid pedido_id FK
        uuid registrado_por FK
        string tipo "SINAL, PARCIAL, SALDO"
        string forma "PIX, DINHEIRO, CARTAO"
        decimal valor
        boolean estornado
        timestamp pago_em
    }
    ENTREGA {
        uuid id PK
        uuid pedido_id FK
        uuid endereco_id FK
        uuid entregador_id FK
        string status "PENDENTE, EM_ROTA, CONCLUIDA"
        timestamp saiu_em
        timestamp entregue_em
        text observacao
    }
    HISTORICO_STATUS {
        uuid id PK
        uuid pedido_id FK
        uuid usuario_id FK
        string status_anterior
        string status_novo
        text observacao
        timestamp alterado_em
    }
```


## 3. Máquina de estados do Pedido

```mermaid
stateDiagram-v2
    [*] --> NOVO
    NOVO --> CONFIRMADO: sinal registrado (ou pronta-entrega)
    NOVO --> CANCELADO: cancelar
    CONFIRMADO --> EM_PRODUCAO: iniciar produção
    CONFIRMADO --> CANCELADO: cancelar
    EM_PRODUCAO --> PRONTO: finalizar produção
    EM_PRODUCAO --> CANCELADO: cancelar (Proprietária)
    PRONTO --> SAIU_PARA_ENTREGA: tipo = ENTREGA
    PRONTO --> ENTREGUE: tipo = RETIRADA
    SAIU_PARA_ENTREGA --> ENTREGUE: confirmar entrega
    ENTREGUE --> [*]
    CANCELADO --> [*]
```

**Quem pode fazer cada transição:**

| Transição | Perfis autorizados |
|---|---|
| `NOVO → CONFIRMADO` | Atendente, Proprietária |
| `CONFIRMADO → EM_PRODUCAO` | Confeiteiro, Proprietária |
| `EM_PRODUCAO → PRONTO` | Confeiteiro, Proprietária |
| `PRONTO → SAIU_PARA_ENTREGA` | Entregador, Proprietária |
| `→ ENTREGUE` | Entregador (entrega), Atendente (retirada), Proprietária |
| `→ CANCELADO` | Atendente (até `CONFIRMADO`), Proprietária (qualquer estado ≠ `ENTREGUE`) |

### 3.1 Máquina de estados do pagamento (derivada)

```mermaid
stateDiagram-v2
    [*] --> PENDENTE
    PENDENTE --> SINAL_PAGO: soma pagamentos > 0 e < total
    SINAL_PAGO --> PAGO: soma pagamentos = total
    PENDENTE --> PAGO: pagamento integral
    PAGO --> SINAL_PAGO: estorno parcial
    SINAL_PAGO --> PENDENTE: estorno total
```

---

## 4. Diagrama de classes (domínio)

```mermaid
classDiagram
    class Pedido {
        +UUID id
        +int numero
        +TipoPedido tipo
        +StatusPedido status
        +datetime dataHoraEntrega
        +Decimal subtotal
        +Decimal desconto
        +Decimal taxaEntrega
        +Decimal total
        +adicionarItem(produto, qtd, obs)
        +removerItem(itemId)
        +recalcularTotais()
        +transicionarPara(novoStatus, usuario)
        +cancelar(motivo, usuario)
        +saldoDevedor() Decimal
        +situacaoPagamento() SituacaoPagamento
    }
    class ItemPedido {
        +UUID id
        +string nomeProduto
        +Decimal precoUnitario
        +Decimal quantidade
        +Decimal totalItem()
    }
    class Pagamento {
        +UUID id
        +TipoPagamento tipo
        +FormaPagamento forma
        +Decimal valor
        +bool estornado
        +datetime pagoEm
    }
    class Entrega {
        +StatusEntrega status
        +datetime saiuEm
        +datetime entregueEm
        +confirmar(usuario)
    }
    class Cliente {
        +UUID id
        +string nome
        +string telefone
        +anonimizar()
    }
    class Produto {
        +UUID id
        +string nome
        +Decimal precoBase
        +bool sobEncomenda
        +bool ativo
    }
    class HistoricoStatus {
        +StatusPedido anterior
        +StatusPedido novo
        +datetime alteradoEm
    }
    class Usuario {
        +Perfil perfil
        +podeExecutar(acao) bool
    }

    Cliente "1" --> "*" Pedido
    Pedido "1" *-- "1..*" ItemPedido
    Pedido "1" *-- "*" Pagamento
    Pedido "1" *-- "0..1" Entrega
    Pedido "1" *-- "*" HistoricoStatus
    ItemPedido "*" --> "1" Produto
    Usuario "1" --> "*" HistoricoStatus
```


## 5. Diagramas de sequência

### 5.1 Registrar pedido com sinal

```mermaid
sequenceDiagram
    actor At as Atendente
    participant UI as Front-end (SPA)
    participant API as API (FastAPI)
    participant S as PedidoService
    participant DB as PostgreSQL

    At->>UI: Busca cliente e monta pedido
    UI->>API: POST /pedidos {cliente, itens, data, tipo}
    API->>API: Valida token e perfil
    API->>S: criar_pedido(dados, usuario)
    S->>DB: Busca produtos/preços ativos
    S->>S: Congela preços, calcula totais (RN02, RN06)
    S->>DB: Verifica capacidade do dia (RF24)
    S->>DB: INSERT pedido + itens + historico(NOVO)
    API-->>UI: 201 Created {pedido, alerta_capacidade?}
    At->>UI: Registra sinal de 50%
    UI->>API: POST /pedidos/{id}/pagamentos
    API->>S: registrar_pagamento()
    S->>S: Valida RN09, recalcula situação de pagamento
    S->>S: Aplica RN04 → transiciona para CONFIRMADO
    S->>DB: INSERT pagamento + historico(CONFIRMADO)
    API-->>UI: 201 Created {pedido atualizado}
```

### 5.2 Entrega e quitação do saldo

```mermaid
sequenceDiagram
    actor En as Entregador
    participant UI as Front-end
    participant API as API
    participant S as PedidoService
    participant DB as PostgreSQL

    En->>UI: Abre "Entregas do dia"
    UI->>API: GET /entregas?data=hoje
    API-->>UI: Lista (endereço, contato, saldo)
    En->>UI: Recebe o saldo e confirma
    UI->>API: POST /pedidos/{id}/pagamentos {SALDO}
    UI->>API: POST /pedidos/{id}/transicoes {ENTREGUE}
    API->>S: transicionar(ENTREGUE)
    S->>S: Valida RN08 (saldo = 0 ou autorização)
    S->>DB: UPDATE pedido, entrega; INSERT historico
    API-->>UI: 200 OK
```

---

## 6. Modelo de permissões (RBAC)

| Recurso / Ação | Proprietária | Atendente | Confeiteiro | Entregador |
|---|:-:|:-:|:-:|:-:|
| Gerenciar usuários | ✅ | ❌ | ❌ | ❌ |
| Gerenciar catálogo/preços | ✅ | ❌ | ❌ | ❌ |
| Gerenciar clientes | ✅ | ✅ | ❌ | ❌ |
| Criar/editar pedido | ✅ | ✅ | ❌ | ❌ |
| Aplicar desconto | ✅ | ❌ | ❌ | ❌ |
| Registrar pagamento | ✅ | ✅ | ❌ | ✅ (saldo na entrega) |
| Ver fila de produção | ✅ | ✅ | ✅ | ❌ |
| Atualizar produção | ✅ | ❌ | ✅ | ❌ |
| Ver/confirmar entregas | ✅ | ✅ (ver) | ❌ | ✅ |
| Cancelar pedido | ✅ | ✅ (até CONFIRMADO) | ❌ | ❌ |
| Relatórios financeiros | ✅ | ❌ | ❌ | ❌ |

---

## 7. Extensão futura (v2): estoque e ficha técnica

```mermaid
erDiagram
    PRODUTO ||--o{ FICHA_TECNICA : "possui"
    INSUMO ||--o{ FICHA_TECNICA : "compoe"
    INSUMO ||--o{ MOVIMENTACAO_ESTOQUE : "movimenta"
    PEDIDO ||--o{ MOVIMENTACAO_ESTOQUE : "consome"

    INSUMO {
        uuid id PK
        string nome
        string unidade
        decimal quantidade_atual
        decimal estoque_minimo
        decimal custo_medio
    }
    FICHA_TECNICA {
        uuid id PK
        uuid produto_id FK
        uuid insumo_id FK
        decimal quantidade_por_unidade
    }
    MOVIMENTACAO_ESTOQUE {
        uuid id PK
        uuid insumo_id FK
        uuid pedido_id FK
        string tipo "ENTRADA, SAIDA, AJUSTE"
        decimal quantidade
        timestamp registrado_em
    }
```

Com isso será possível calcular **custo e margem por produto** e gerar **lista de compras** a partir da produção planejada.