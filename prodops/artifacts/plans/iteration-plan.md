# Iteration Plan — Platform Release v1.0.0

**Workspace:** prodops-workspace
**Business Intent:** BI-01 — Compra de 1 item por Pix via Listagem
**Release:** v1.0.0
**Modo Workspace:** Upstream (orquestração experimental — sem gates de CI-Async)
**Modo Produto:** Downstream — CI-SYNC only (Bootstrap → Hack → Sync → Finish)
**Branch em todos os repos:** `prodops-workspace`
**Objetivo de validação:** Todos os serviços em execução local para validação visual a partir do Webshop
**CI-Async (Ship → Validate → Promote):** Diferido — será executado após validação visual confirmada

---

## Stack e Dependências

```
webshop  (front-end / Value Stream)
  └─ webshop-api  (BFF — Backend for Frontend)
       ├─ search-api   (Catálogo — GET /produtos)
       └─ order-mngt-api  (Pedidos — POST /pedidos · GET /pedidos/:id)
```

A execução segue a cadeia de baixo para cima para garantir que cada serviço
possa ser integrado antes de subir a camada seguinte.

---

## Fase 0 — Readiness (pré-requisitos)

**Objetivo:** Garantir que todos os repos estão prontos para executar CI-SYNC.

| # | Ação | Repo | Responsável |
|---|---|---|---|
| 0.1 | Instalar ProdOps completo na branch `prodops-workspace` | search-api | Workspace Agent |
| 0.2 | Verificar que `prodops/runtime/runtime.yaml` existe e `project-number` está preenchido em todos os repos | todos | Workspace Agent |
| 0.3 | Verificar que Local OBC BI-01 está em `Draft` e rastreado no issue correspondente | todos | Workspace Agent |

**Critério de saída:** `doctor.sh` passa em todos os repos na branch `prodops-workspace`.

---

## Fase 1 — CI-SYNC: search-api + order-mngt-api (paralelo)

Execução paralela — sem dependência entre si.

### 1-A · search-api — `GET /produtos`

**Local OBC:** `prodops/artifacts/obcs/local-bi-01-search-api.md`
**Outcome:** Retornar listagem de produtos disponíveis no catálogo para alimentar o webshop-api.

| Fase CI-SYNC | Objetivo |
|---|---|
| **Bootstrap** | Preparar branch `prodops-workspace`, verificar tooling (Node/Nest), revisar Local OBC e BDD Feature |
| **Hack** | Implementar `GET /produtos` via TDD: endpoint retorna lista de produtos com status disponível; 503 em falha de fonte de dados (nunca 200 vazio silencioso); propagação de `correlationId` em logs |
| **Sync** | Rebase com `master`; alinhar artifacts com implementação (BDD atualizado, SLIs verificados) |
| **Finish** | Done criteria: testes passando, contrato de resposta validado, correlationId propagado, `GET /produtos` responde < 300ms (p95) no teste local |

**Contrato de saída (response):**
```json
{
  "produtos": [
    { "id": "string", "nome": "string", "preco": 0.00, "disponivel": true }
  ]
}
```

**Run local:** `npm run start:dev` — porta `3001`

---

### 1-B · order-mngt-api — `POST /pedidos` · `GET /pedidos/:id`

**Local OBC:** `prodops/artifacts/obcs/local-bi-01-order-mngt-api.md`
**Outcome:** Criar e recuperar pedidos com idempotência garantida por `correlationId`.

| Fase CI-SYNC | Objetivo |
|---|---|
| **Bootstrap** | Preparar branch `prodops-workspace`, verificar tooling e dependências de persistência, revisar Local OBC e BDD Feature |
| **Hack** | Implementar via TDD: `POST /pedidos` idempotente (mesmo correlationId → mesmo pedidoId); `GET /pedidos/:id`; validação de `productId` (422 se inválido); emissão de `pedido.criado` e `pedido.recebido` após persistência confirmada |
| **Sync** | Rebase com base; alinhar BDD Feature com implementação; confirmar eventos emitidos |
| **Finish** | Done criteria: idempotência testada com mesmo correlationId; `POST /pedidos` < 600ms; rejeição de productId inválido com 422; nunca confirmar pedido não persistido |

**Contrato de saída:**
```json
{
  "pedidoId": "string",
  "productId": "string",
  "customerId": "string",
  "correlationId": "string",
  "status": "criado"
}
```

**Run local:** `npm run start:dev` — porta `3002`

---

**Critério de saída da Fase 1:**
- search-api: `GET http://localhost:3001/produtos` retorna lista de produtos
- order-mngt-api: `POST http://localhost:3002/pedidos` cria pedido; segunda chamada com mesmo correlationId retorna o mesmo pedidoId

---

## Fase 2 — CI-SYNC: webshop-api (BFF)

**Pré-requisito:** Fase 1 completa (search-api e order-mngt-api rodando localmente)

**Local OBC:** `prodops/artifacts/obcs/local-bi-01-webshop-api.md`
**Outcome:** Expor ao front-end API unificada e resiliente — listagem de produtos, criação e detalhe de pedido.

| Fase CI-SYNC | Objetivo |
|---|---|
| **Bootstrap** | Preparar branch `prodops-workspace`; configurar URLs dos serviços downstream (`SEARCH_API_URL=http://localhost:3001`, `ORDER_API_URL=http://localhost:3002`); revisar Local OBC e BDD Feature |
| **Hack** | Implementar via TDD: `GET /produtos` → delega ao search-api; `POST /pedidos` → valida productId (422) → delega ao order-mngt-api com correlationId; `GET /pedidos/:id` → delega ao order-mngt-api; resilência: 503 com `Retry-After` em falha transiente do search-api (nunca stack trace) |
| **Sync** | Rebase com base; alinhar BDD Feature; confirmar propagação de correlationId ponta a ponta |
| **Finish** | Done criteria: `GET /produtos` < 500ms; `POST /pedidos` < 800ms; `GET /pedidos/:id` < 300ms; idempotência end-to-end via correlationId; nenhum detalhe técnico exposto ao front-end em erro |

**Variáveis de ambiente necessárias:**
```bash
SEARCH_API_URL=http://localhost:3001
ORDER_API_URL=http://localhost:3002
PORT=3000
```

**Run local:** `npm run start:dev` — porta `3000`

**Critério de saída da Fase 2:**
- `GET http://localhost:3000/produtos` retorna listagem via search-api
- `POST http://localhost:3000/pedidos` cria pedido via order-mngt-api
- `GET http://localhost:3000/pedidos/:id` retorna detalhe do pedido

---

## Fase 3 — CI-SYNC: webshop (front-end)

**Pré-requisito:** Fase 2 completa (webshop-api rodando localmente na porta 3000)

**Local OBC:** `prodops/artifacts/obcs/local-bi-01-webshop.md`
**Outcome:** Exibir listagem de produtos e conduzir o cliente pelo fluxo completo de compra — seleção → criar pedido → ver detalhe — sem fricção.

| Fase CI-SYNC | Objetivo |
|---|---|
| **Bootstrap** | Preparar branch `prodops-workspace`; configurar `WEBSHOP_API_URL=http://localhost:3000`; revisar Local OBC e BDD Feature |
| **Hack** | Implementar via TDD: tela de listagem de produtos (carrega de `GET /produtos`); botão de compra (desabilitado após primeiro clique até resposta — prevenir duplo clique); tela de detalhe do pedido (após criação via `POST /pedidos`); correlationId gerado no front-end e propagado em todas as chamadas; mensagem de indisponibilidade em falha do webshop-api (sem detalhes técnicos) |
| **Sync** | Rebase com base; alinhar BDD Feature; confirmar fluxo ponta a ponta com webshop-api real |
| **Finish** | Done criteria: listagem carrega < 2s (p95); fluxo completo seleção → pedido confirmado sem erro; detalhe do pedido acessível após criação; nenhum dado de sessão logado no front-end |

**Variáveis de ambiente necessárias:**
```bash
WEBSHOP_API_URL=http://localhost:3000
```

**Run local:** `npm run dev` (ou `npm start`) — porta `8080`

**Critério de saída da Fase 3:**
- Front-end acessível em `http://localhost:8080`
- Listagem de produtos visível
- Fluxo completo executável: selecionar item → criar pedido → ver detalhe

---

## Fase 4 — Stack Up e Validação Visual

**Objetivo:** Subir toda a stack localmente e validar o fluxo completo de ponta a ponta a partir do Webshop.

### Ordem de inicialização

```
1. search-api       → http://localhost:3001
2. order-mngt-api   → http://localhost:3002
3. webshop-api      → http://localhost:3000  (após 1 e 2 healthy)
4. webshop          → http://localhost:8080  (após 3 healthy)
```

### Smoke test do fluxo completo

| # | Ação | Resultado esperado |
|---|---|---|
| 1 | Acessar `http://localhost:8080` | Tela de listagem de produtos carregada |
| 2 | Verificar que produtos aparecem na lista | ≥ 1 produto exibido (vindo do search-api) |
| 3 | Clicar em "Comprar" em um produto | Botão desabilitado durante a requisição |
| 4 | Aguardar resposta | Tela de detalhe do pedido exibida com `pedidoId` |
| 5 | Clicar em "Comprar" novamente no mesmo produto (mesmo correlationId) | Mesmo `pedidoId` retornado (idempotência visível) |
| 6 | Simular indisponibilidade do search-api (parar serviço) | Mensagem de indisponibilidade exibida — sem stack trace |

### Critério de aceitação da Fase 4

- [ ] Fluxo completo executado sem erros visíveis ao usuário
- [ ] Idempotência demonstrada visualmente
- [ ] Falha de serviço exibe mensagem adequada ao usuário
- [ ] Nenhum detalhe técnico exposto ao cliente

---

## Estado final esperado

| Serviço | Porta | Status |
|---|---|---|
| search-api | 3001 | Rodando — CI-SYNC completo |
| order-mngt-api | 3002 | Rodando — CI-SYNC completo |
| webshop-api | 3000 | Rodando — CI-SYNC completo |
| webshop | 8080 | Rodando — CI-SYNC completo |

---

## Próximo passo após validação visual

Se a validação visual confirmar o fluxo ponta a ponta:

1. **Promover no Workspace** — registrar resultado do experimento Upstream
2. **Executar CI-Async** em cada produto — Ship → Validate → Promote
3. **Fechar Global OBC BI-01** como In Delivery → Operational no prodops-portfolio

---

## Rastreabilidade

| Artefato | Link |
|---|---|
| Global OBC BI-01 | `prodops-portfolio/prodops/artifacts/obcs/global-bi-01-compra-pix-listagem.md` |
| Local OBC webshop | `webshop/prodops/artifacts/obcs/local-bi-01-webshop.md` |
| Local OBC webshop-api | `webshop-api/prodops/artifacts/obcs/local-bi-01-webshop-api.md` |
| Local OBC order-mngt-api | `order-mngt-api/prodops/artifacts/obcs/local-bi-01-order-mngt-api.md` |
| Local OBC search-api | `search-api/prodops/artifacts/obcs/local-bi-01-search-api.md` |
| Portfolio Issue | [prodops-portfolio#5](https://github.com/produtoreativo/prodops-portfolio/issues/5) |
