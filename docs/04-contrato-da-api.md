# 04. Contrato da API REST, Links Compartilhados e Tipos (`shared/`)

O servidor Hono é o único que se comunica com a Pluggy, o Banco Central e o Yahoo Finance. O navegador conversa exclusivamente com a API JSON em `/api` (`http://127.0.0.1:5174`, com proxy do Vite em `:5173` durante o desenvolvimento).

---

## 1. Convenções Gerais da API

| Assunto | Regra |
|---|---|
| **Dinheiro** | **Centavos inteiros** (`number`), campos terminam em `Cents` (`123456` = R$ 1.234,56). Nunca há decimais em dinheiro. |
| **Sinal** | Em `TransactionDto`, `amountCents` segue **"gasto positivo, entrada negativa"** (para conta e cartão). O valor original enviado pela fonte fica em `originalAmountCents`. |
| **Percentual e Participação** | Percentuais em pontos percentuais (`12.5` = 12,5%, sufixo `Pct`). Frações de participação de `0` a `1` (`share`, `usedShare`). |
| **Datas e Meses** | Datas: `AAAA-MM-DD` (horário de Brasília). Meses: `AAAA-MM`. Instantes (`...At`): ISO 8601 UTC. |
| **Critério do mês (`basis`)** | As rotas de gasto (`/overview`, `/spending`, `/spending/monthly`, `/spending/cashflow`) aceitam `basis=bill` (padrão: cartão conta no mês de vencimento da fatura `statementMonth`) ou `basis=purchase` (cartão conta no mês da data da compra `date`). |
| **Erros HTTP** | Status `400`, `404`, `409`, `500` ou `503` com corpo `{ "error": "CODIGO", "message": "Mensagem em português" }`. Códigos: `INVALID_MONTH`, `INVALID_DATE`, `INVALID_PARAM`, `INVALID_JSON`, `INVALID_BODY`, `INVALID_CATEGORY`, `INVALID_PATTERN`, `INVALID_TYPE`, `INVALID_TICKER`, `NOT_FOUND`, `PRICE_UNAVAILABLE` (503), `SYNC_RUNNING` (409), `INTERNAL` (500). |

---

## 2. Construtor de Links entre Telas (`shared/links.ts`)

Tanto o servidor (ao gerar `InsightDto.link`) quanto o frontend (em todos os `<Button to={...}>` e `<Card to={...}>`) usam o mesmo módulo `shared/links.ts`, com os mesmos nomes de parâmetros da API:

```ts
export type TxType = 'expense' | 'income' | 'movement' | 'excluded';
export type LinkBasis = 'bill' | 'purchase';
export type TxPeriod = 'month' | '30d' | 'all' | 'custom';

export interface TransactionsLink {
  q?: string;
  accountId?: string;
  categoryId?: string;
  billId?: string;
  type?: TxType;
  review?: boolean;
  period?: TxPeriod;
  from?: string;
  to?: string;
  month?: string;
  key?: string;
}

type Params = Record<string, string | boolean | undefined | null>;

function build(path: string, params: Params): string {
  const p = new URLSearchParams();
  for (const [k, v] of Object.entries(params)) {
    if (v === undefined || v === null || v === '' || v === false) continue;
    p.set(k, v === true ? '1' : v);
  }
  const s = p.toString();
  return s ? `${path}?${s}` : path;
}

export const ROUTE = {
  overview: '/',
  spending: '/gastos',
  transactions: '/transacoes',
  cards: '/cartoes',
  investments: '/investimentos',
  insights: '/sugestoes',
  connections: '/conexoes',
} as const;

export const links = {
  overview: (o: { month?: string; basis?: LinkBasis } = {}) =>
    build(ROUTE.overview, { month: o.month, basis: o.basis === 'purchase' ? 'purchase' : undefined }),
  spending: (o: { categoryId?: string; month?: string; basis?: LinkBasis; section?: 'recorrencias' } = {}) =>
    build(ROUTE.spending, { categoryId: o.categoryId, month: o.month, basis: o.basis === 'purchase' ? 'purchase' : undefined }) +
    (o.section ? `#${o.section}` : ''),
  transactions: (o: TransactionsLink = {}) =>
    build(ROUTE.transactions, {
      q: o.q, accountId: o.accountId, categoryId: o.categoryId, billId: o.billId, type: o.type, review: o.review,
      period: o.period, from: o.from, to: o.to, month: o.month, key: o.key,
    }),
  cards: (o: { accountId?: string; month?: string } = {}) => build(ROUTE.cards, o),
  investments: (o: { positionId?: string } = {}) => build(ROUTE.investments, o),
  insights: () => ROUTE.insights,
  connections: () => ROUTE.connections,
};
```

---

## 3. Especificação Completa das Rotas (`/api/*`)

### 3.1. Rotas Gerais
- **`GET /api/health`** → `{ "ok": true }`
- **`GET /api/meta`** → `MetaResponse`
  - Retorna `dataSource` (`mock | pluggy`), `today`, `currentMonth`, `firstMonth`, `lastMonth` (menor e maior `statement_month` ativo), `reviewCount` (fila de revisão do mês atual), `lastSyncAt`, `syncRunning` e `syncHealth` (pior estado entre as conexões: `ok | stale | empty_read | auth_expired | error`).
  - Também dispara em background (`void freshMarket(false)`) a atualização de cotações PTAX e preços de tickers caso tenham mais de 6 horas.
- **`GET /api/categories`** → `CategoryNode[]`
  - Árvore de categorias de 2 níveis lida da tabela `categories` (as 22 categorias padrão + as personalizadas de `local-data/custom-categories.json`).
- **`GET /api/accounts`** → `AccountsResponse` (`{ accounts: AccountDto[] }`)
  - Lista apenas contas e cartões **visíveis** (`is_removed = 0 AND COALESCE(hidden_override, auto_hidden) = 0`), ordenados por `kind, institution, name`.
- **`PATCH /api/accounts/:id/visibility`** → `ConnectionAccountDto`
  - Corpo: `{ "hidden": true | false | null }` (`null` volta para a regra automática `auto_hidden`).

### 3.2. Tela 1 — Visão geral
- **`GET /api/overview?month=AAAA-MM&basis=bill|purchase`** → `OverviewResponse`
  - `netWorth`: `{ totalCents, accountsCents, investmentsCents, cardDebtCents, changeInMonthCents, series }` (série diária dos últimos 90 dias até `today`).
  - `spending`: `{ monthCents, previousMonthCents, average6mCents, averageFrom, averageTo, forecastCents, isCurrentMonth }`.
  - `topCategories`: top 5 categorias de `spendingByCategory(db, month, basis)`.
  - `nextBills`: próxima fatura (fechada a vencer ou aberta estimada) de cada cartão visível.
  - `insights`: até 3 sugestões ativas (no máximo 1 por `type`, ordenadas por relevância/impacto).
  - `sync`: estado de sincronização por conta visível.

### 3.3. Tela 2 — Gastos
- **`GET /api/spending?month=AAAA-MM&basis=bill|purchase`** → `SpendingResponse`
  - `totalCents`: soma das categorias-mãe contadas no gasto do mês (`countedParents`).
  - `categories`: `CategorySpending[]` (ordenadas da maior para a menor), cada uma com `subcategories[]`. Quando uma transação está classificada diretamente no ID da categoria-mãe (ex.: `17000000`), ela aparece como uma subcategoria de mesmo ID e nome da mãe (e no drill-down usa a URL `?categoryId=17000000:own` e `categoryExact=1` em `/api/transactions`).
  - `outside`: `CategorySpending[]` das categorias-mãe de movimentação ou com saldo líquido `<= 0` no mês (`share: 0`).
- **`GET /api/spending/monthly?end=AAAA-MM&months=12&basis=bill|purchase`** → `MonthlySpendingResponse`
  - Série mensal empilhada por categoria-mãe (`1 <= months <= 36`, padrão `12`). Para cada mês `m`, `series[i].totalCents` é idêntico a `GET /api/spending?month=m`.
- **`GET /api/spending/recurring`** → `RecurringResponse`
  - Recorrências e assinaturas ativas detectadas nos últimos 400 dias (`items: RecurringItem[]`, `monthlyTotalCents`).
- **`GET /api/spending/cashflow?month=AAAA-MM&basis=bill|purchase`** → `CashflowResponse`
  - `incomeCents` (total de receitas), `investmentIncomeCents` (rendimentos/dividendos/JCP), `transfersInCents` (Pix/TED recebidos de terceiros), `expenseCents` (igual a `GET /api/spending.totalCents`), `netCents` (`incomeCents - expenseCents`), `movementsInCents`, `movementsOutCents`, e `days` (curva diária de entradas e saídas do mês).

### 3.4. Tela 3 — Transações e Regras
- **`GET /api/transactions`** → `TransactionListResponse`
  - Query params combináveis: `month`, `from`, `to`, `accountId`, `categoryId`, `categoryExact=1`, `billId`, `type` (`expense | income | movement | excluded | all`), `q` (busca textual sem acento + busca numérica por valor em reais), `review=1`, `includeRemoved=1`, `page` (padrão 1), `pageSize` (1 a 500, padrão 50).
  - Retorna `{ page, pageSize, total, totalAmountCents, reviewCount, items }`. `reviewCount` reflete o tamanho da fila de revisão no mesmo período (`month` ou `from`/`to`) da consulta, ignorando os demais filtros.
- **`GET /api/transactions/:key`** → `TransactionDto`
- **`PATCH /api/transactions/:key/category`** → `RecategorizeResponse`
  - Corpo: `{ categoryId, createRule?: boolean, rulePattern?: string }`.
  - Grava o override manual, zera `is_new`, opcionalmente cria/atualiza a regra em `category_rules`, roda `deriveTransactions(db, today)` e retorna `{ transaction, rule, affectedCount, ruleCreated, previousRuleCategoryId }` (tudo o que o frontend precisa para um "Desfazer" exato).
- **`DELETE /api/transactions/:key/category`** → `TransactionDto`
  - Remove a escolha manual de categoria (e, se não sobrar override de movimentação, apaga a linha de `manual_overrides` e restaura `is_new = prev_is_new`).
- **`PATCH /api/transactions/:key/movement`** → `TransactionDto`
  - Corpo: `{ isMovement: true | false | null }`. Força movimentação, força não-movimentação ou (`null`) volta para a detecção automática.
- **`POST /api/transactions/:key/reviewed`** → `TransactionDto`
- **`POST /api/transactions/reviewed`** → `ConfirmReviewResponse`
  - Confirmação em lote (ou unitária com Desfazer) da fila de revisão sem alterar a categoria. Aceita `{ keys: string[] }` ou `{ query: { month?, from?, to?, accountId?, categoryId?, type?, q? } }` (ignorando itens `uncategorized`, que continuam exigindo categoria). Retorna `{ confirmedCount, previous: ReviewSnapshot[] }`.
- **`POST /api/transactions/reviewed/undo`** → `{ restoredCount: number }`
  - Corpo: `{ previous: ReviewSnapshot[] }`. Restaura o estado exato da fila antes da confirmação.
- **`GET /api/rules`** → `{ rules: CategoryRuleDto[] }`
- **`POST /api/rules`** → `CategoryRuleDto` (status `201`)
  - Corpo: `{ pattern, categoryId, id?, createdAt? }` (aceita `id` e `createdAt` opcionais para recriar uma regra apagada com o mesmo ID no Desfazer).
- **`DELETE /api/rules/:id`** → status `204`

### 3.5. Tela 4 — Cartões
- **`GET /api/cards`** → `CardsResponse` (`{ cards: CardDto[] }`)
  - Para cada cartão visível: limites, `currentBill` (fatura aberta estimada com `datesEstimated`), `previousBills` (com `purchasesCents`, `totalCents`, `differenceCents`, `hasMismatch`) e `billCheckAvailable`.
- **`GET /api/cards/commitments?months=12`** → `CommitmentsResponse`
  - Parcelas futuras mês a mês (de 1 a 24 meses, padrão 12), combinando parcelas já enviadas pelo banco (`projected: false`) e parcelas projetadas pelo painel (`projected: true`).

### 3.6. Tela 5 — Investimentos e Investimentos Manuais
- **`GET /api/investments`** → `InvestmentsResponse`
  - Atualiza cotações/preços se tiverem mais de 6 h (aguardando no máximo 3 s), retorna `totalCents`, `historyStartsAt`, `allocation` (`byClass`, `byInstitution`, `byIssuer`), `positions` (agrupadas por `groupPositions` + posições manuais `man_<id>`) e `maturities` (vencimentos futuros de renda fixa, sem Caixinhas do Nubank).
- **`GET /api/investments/:id/movements`** → `InvestmentMovementsResponse`
- **`GET /api/investments/history?from=&to=`** → `InvestmentHistoryResponse`
- **`GET /api/investments/performance?from=&to=`** → `PerformanceResponse` (Carteira vs. CDI vs. IPCA em base 100 via TWR).
- **`GET /api/benchmarks?series=cdi|ipca|selic|usd|eur&from=&to=`** → `BenchmarksResponse`
- **`GET /api/manual-investments`** → `ManualInvestmentsResponse`
- **`GET /api/manual-investments/lookup?ticker=VOO`** → `TickerLookupResponse`
- **`POST /api/manual-investments`** → `ManualInvestmentDto` (status `201`)
- **`PATCH /api/manual-investments/:id`** → `ManualInvestmentDto`
- **`DELETE /api/manual-investments/:id`** → status `204` (soft delete em `deleted_at`)
- **`POST /api/manual-investments/:id/restore`** → `ManualInvestmentDto` (desfaz a exclusão mantendo todo o histórico de snapshots).

### 3.7. Tela 6 — Sugestões
- **`GET /api/insights?includeDismissed=1`** → `InsightsResponse` (`{ items: InsightDto[], mutedTypes: InsightType[] }`)
- **`GET /api/insights/config`** → `InsightThresholds` (conteúdo de `server/config/insights.json`)
- **`POST /api/insights/:id/dismiss`** → `InsightsResponse` (corpo `{ scope: 'one' | 'type' }`)
- **`POST /api/insights/:id/restore`** → `InsightsResponse` (desfaz o `"entendi"`)
- **`DELETE /api/insights/muted/:type`** → `InsightsResponse` (desfaz o silenciamento de um tipo)

### 3.8. Tela 7 — Conexões e Ajustes
- **`GET /api/connections`** → `ConnectionsResponse`
- **`POST /api/sync`** → `SyncRunDto` (409 `SYNC_RUNNING` se já houver sincronização em andamento)
- **`GET /api/sync/log?limit=20`** → `SyncLogResponse`

---

## 4. Tipos Compartilhados Completos (`shared/types.ts`)

Copie ou use esta definição como contrato único entre `server/` e `web/`:

```ts
export type DataSourceKind = 'mock' | 'pluggy';
export type AccountKind = 'BANK' | 'CREDIT';
export type CategorySource = 'manual' | 'rule' | 'pluggy' | 'keyword' | 'refund' | 'none';
export type MovementReason = 'bill_payment' | 'own_transfer' | 'investment' | 'manual';
export type AssetClass = 'Renda fixa' | 'Fundos' | 'Ações' | 'ETFs' | 'FIIs' | 'Previdência' | 'Exterior' | 'Outros';
export const ASSET_CLASSES: AssetClass[] = ['Renda fixa', 'Fundos', 'Ações', 'ETFs', 'FIIs', 'Previdência', 'Exterior', 'Outros'];
export type CurrencyCode = 'BRL' | 'USD' | 'EUR';
export type SpendingBasis = 'bill' | 'purchase';
export type ConnectionState = 'ok' | 'stale' | 'empty_read' | 'auth_expired' | 'error';
export type ReviewReason = 'uncategorized' | 'ambiguous' | 'new' | 'bank_suggested';
export type ManualInvestmentKind = 'asset' | 'value';
export type InsightType =
  | 'spending_outlier'
  | 'duplicate_charge'
  | 'subscription_change'
  | 'idle_cash'
  | 'issuer_concentration'
  | 'maturity_soon'
  | 'bill_above_normal'
  | 'review_pending'
  | 'authorization_expiring'
  | 'account_stale'
  | 'weekly_summary'
  | 'manual_stale';
```

*(Todos os campos de cada interface DTO seguem exatamente as tabelas e descrições da Seção 3 acima.)*
