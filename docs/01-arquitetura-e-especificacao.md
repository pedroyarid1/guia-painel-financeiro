# 01. Arquitetura, Especificação e Banco de Dados

Este documento define o objetivo, a stack tecnológica, a árvore de arquivos, o schema completo do banco SQLite (com as 8 migrações) e os requisitos funcionais das 7 telas do Painel Financeiro Pessoal.

---

## 1. Objetivo e Princípios do Produto

Construir um painel pessoal, de um único usuário, que roda localmente no computador (`127.0.0.1`), lê contas correntes, cartões de crédito e investimentos dos bancos do usuário pela Pluggy (caminho gratuito **Meu Pluggy**) e entrega controle de gastos por categoria, acompanhamento de faturas/parcelas, evolução e rentabilidade da carteira e sugestões financeiras ativas.

### Quatro princípios inegociáveis
1. **Número confiável antes de número bonito**: todo total na tela mostra exatamente de onde veio (clicável até a lista de transações que o compõem) e a data/hora da última sincronização da conta. Lista de categorias, gráfico de rosca, barra mensal de 12 meses, card da Visão geral e saída do fluxo de caixa batem até o último centavo.
2. **Dados só no computador**: banco SQLite local em `local-data/` (fora do git). Nada sai para serviços externos além das chamadas de leitura para a Pluggy, para o Banco Central (séries SGS de CDI, IPCA, Selic, Dólar e Euro) e, quando o usuário cadastra ações/ETFs no exterior por ticker, para o endpoint público de gráfico do Yahoo Finance (enviando apenas o código do ticker, sem dados pessoais).
3. **Funciona 100% sem a Pluggy (`DATA_SOURCE=mock`)**: sem arquivo `.env`, o app sobe no modo fictício com ~14 meses de dados determinísticos e realistas. É assim que se desenvolve, testa e demonstra o sistema.
4. **Simples de manter**: um único repositório, um único `package.json`, TypeScript de ponta a ponta (Node 24+ executando `.ts` nativamente no backend), e um único comando (`npm start` após o build, ou `npm run dev` em desenvolvimento).

---

## 2. Stack Tecnológica e `package.json`

- **Runtime**: Node.js `>=24` (testado no Node 26; usa remoção nativa de tipos TypeScript e `process.loadEnvFile`).
- **Backend**: `hono` + `@hono/node-server` (servidor HTTP rápido e leve), `better-sqlite3` (SQLite síncrono, rápido, já vem com binário pré-compilado sem exigir compilador C++), `pluggy-sdk`.
- **Frontend**: `react` 19 + `react-dom` 19, `react-router` 7/8, `motion` (Framer Motion / `motion/react` para física de molas e transições), `recharts` 3 (gráficos tematizados pelos tokens CSS).
- **Design System & Cores**: `@material/material-color-utilities` (geração da paleta Material 3 Expressive a partir de uma cor-semente `SEED_COLOR`), `@fontsource-variable/google-sans-flex` e `material-symbols` (fontes e ícones locais, funcionando offline).
- **Build & Testes**: `vite`, `@vitejs/plugin-react`, `typescript`, `vitest`.

### Estrutura exata do `package.json`

```json
{
  "name": "painel-financeiro",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "description": "Personal finance dashboard (spending, cards, investments) fed by Pluggy or mock data",
  "engines": {
    "node": ">=24"
  },
  "scripts": {
    "dev": "node scripts/dev.mjs",
    "dev:server": "node --watch server/src/index.ts",
    "dev:web": "vite --config web/vite.config.ts",
    "build": "tsc -p server/tsconfig.json && tsc -p web/tsconfig.json && vite build --config web/vite.config.ts",
    "start": "node server/src/index.ts",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc -p server/tsconfig.json && tsc -p web/tsconfig.json",
    "tokens": "node web/scripts/generate-tokens.ts",
    "backup": "node scripts/backup.mjs"
  },
  "dependencies": {
    "@fontsource-variable/google-sans-flex": "^5.3.2",
    "@hono/node-server": "^2.1.6",
    "@material/material-color-utilities": "^0.4.0",
    "better-sqlite3": "^13.0.3",
    "hono": "^4.13.17",
    "material-symbols": "^0.48.3",
    "motion": "^14.12.0",
    "pluggy-sdk": "^0.94.2",
    "react": "^19.3.1",
    "react-dom": "^19.3.1",
    "react-router": "^8.5.1",
    "recharts": "^3.10.1"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^9.6.0",
    "@types/node": "^26.6.6",
    "@types/react": "^19.3.0",
    "@types/react-dom": "^19.3.0",
    "@vitejs/plugin-react": "^6.3.0",
    "typescript": "^7.0.2",
    "vite": "^8.3.7",
    "vitest": "^5.1.0"
  }
}
```

### Regras de TypeScript para execução direta no Node
Configure `server/tsconfig.json` e `web/tsconfig.json` com `"allowImportingTsExtensions": true`, `"erasableSyntaxOnly": true` (ou equivalente: proibido usar `enum`, `namespace` e parameter properties como `constructor(private x: number)`), `"noEmit": true` e `"moduleResolution": "bundler"` (ou `"nodenext"`). Todos os imports relativos em `server/`, `shared/` e `web/` devem terminar com `.ts` ou `.tsx`.

---

## 3. Árvore de Diretórios da Aplicação

```text
painel-financeiro/
├── .env.example
├── .gitignore                 # Ignora .env, local-data/, *.db, dist/, node_modules/
├── AGENTS.md                  # Instruções e estado do projeto para agentes de IA
├── README.md                  # Como usar e rodar
├── package.json
├── vitest.config.ts           # Roda server/test/**/*.test.ts e web/src/**/*.test.ts
├── scripts/
│   ├── dev.mjs                # Sobe dev:server (:5174) e dev:web (:5173) juntos
│   └── backup.mjs             # Backup online seguro do SQLite e JSONs de local-data para BACKUP_DIR
├── shared/
│   ├── types.ts               # Todos os DTOs e contratos compartilhados entre servidor e front
│   └── links.ts               # Construtor único de rotas e query params (usado pelo front e por InsightDto.link)
├── server/
│   ├── tsconfig.json
│   ├── config/
│   │   └── insights.json      # Limites configuráveis das 12 sugestões
│   ├── src/
│   │   ├── index.ts           # Entrada do servidor: carrega .env, abre DB, serve API + dist/web e dispara sync inicial
│   │   ├── config.ts          # Leitura e validação das variáveis de ambiente
│   │   ├── db/
│   │   │   ├── index.ts       # openDatabase, migrate, seedCategories (+ custom-categories.json), getSetting/setSetting
│   │   │   ├── migrations.ts  # As 8 migrações SQLite forward-only (PRAGMA user_version)
│   │   │   └── rows.ts        # Loaders SQL -> registros de domínio e filtros SQL reutilizáveis
│   │   ├── domain/            # Funções puras (100% testadas em unidade, sem banco nem rede)
│   │   │   ├── types.ts       # Tipos internos de domínio (AccountRecord, TxRecord, BillRecord, etc.)
│   │   │   ├── money.ts       # toCents, normalizeSpend, normalizeText
│   │   │   ├── dates.ts       # toBrasiliaDate, todayBrasilia, addDays, addMonths, diffDays, monthRange
│   │   │   ├── identity.ts    # baseLocalKey (p_... ou f_...), assignLocalKeys
│   │   │   ├── categories.ts  # CATEGORY_TREE (22 categorias padrão), parseCustomCategories, flatCategories, mapPluggyCategory
│   │   │   ├── categorize.ts  # Categorização em camadas, BRAND_RULES, KEYWORD_RULES, userRuleCategory, suggestedRulePattern
│   │   │   ├── movements.ts   # isBillPayment, isAmbiguousBillPayment, isInvestmentMovement, isInvestmentIncome, pairOwnTransfers, matchRefunds
│   │   │   ├── flow.ts        # classifyFlow (expense | income | movement | excluded)
│   │   │   ├── cards.ts       # cycleFor, cycleFromBills, statementMonthFor, findFullPurchaseDuplicates, cardDebtCents, projectInstallments
│   │   │   ├── institutions.ts# INSTITUTION_PATTERNS, inferItemInstitution, friendlyAccountName, titleCase
│   │   │   ├── visibility.ts  # autoHiddenAccounts (contas sem movimento em 12m ou gêmeas vazias)
│   │   │   ├── investments.ts # mapAssetClass, formatRateLabel, groupPositions (caixinhas do Nubank), computePerformanceTwr
│   │   │   ├── investment-names.ts # Limpeza e padronização de nomes de ativos
│   │   │   ├── fx.ts          # Conversão de moedas pela última PTAX disponível até a data
│   │   │   ├── spending.ts    # countedParents, spendingTotals, detectRecurring, findDuplicateCharges, forecastMonthEnd
│   │   │   └── insights.ts    # Geração pura das 12 sugestões e partes formatadas ({{money:N}})
│   │   ├── sources/
│   │   │   ├── types.ts       # Interface DataSource e tipos Pluggy*Like
│   │   │   ├── mock.ts        # Gerador determinístico de ~14 meses (Banco Aurora, Banco Horizonte, Vértice Investimentos)
│   │   │   ├── mock-benchmarks.ts # Séries sintéticas de CDI, Selic, IPCA, USD, EUR e catálogo de tickers
│   │   │   ├── pluggy.ts      # Implementação real com pluggy-sdk (fetchAllTransactions via v2 cursor)
│   │   │   └── mapping.ts     # Conversão de respostas Pluggy para registros do banco
│   │   ├── sync/
│   │   │   ├── sync.ts        # runSync, storeTransactions (remoção reversível + proteção de leitura vazia), recomputeLocal, refreshMarketData
│   │   │   ├── accounts.ts    # Aplica inferItemInstitution, friendlyAccountName e autoHiddenAccounts
│   │   │   ├── derive.ts      # Recalcula statement_month, duplicadas, movimentações, categorias, flow, in_totals e review_flag
│   │   │   ├── benchmarks.ts  # Busca e cache das séries SGS do Banco Central (12, 433, 11, 1, 21619)
│   │   │   └── prices.ts      # Busca e cache de preços de tickers em dólar (Yahoo Finance / mock)
│   │   ├── services/          # Consultas SQL + montagem dos DTOs da API
│   │   │   ├── common.ts      # REVIEW_REASON_SQL, categoriesById, toTransactionDto, HttpError
│   │   │   ├── overview.ts
│   │   │   ├── spending.ts
│   │   │   ├── transactions.ts
│   │   │   ├── cards.ts
│   │   │   ├── investments.ts
│   │   │   ├── manual-investments.ts
│   │   │   ├── insights.ts
│   │   │   └── connections.ts
│   │   └── api/
│   │       └── app.ts         # Rotas Hono em /api/*
│   └── test/                  # Suíte de testes do servidor (regras, sync, formato real MeuPluggy, API, UX review)
└── web/
    ├── index.html
    ├── vite.config.ts         # Servidor dev em 127.0.0.1:5173 com proxy de /api para 127.0.0.1:5174
    ├── tsconfig.json
    ├── scripts/
    │   └── generate-tokens.ts # Gera web/src/design/tokens.css a partir de SEED_COLOR
    └── src/
        ├── main.tsx
        ├── App.tsx
        ├── routes.tsx         # Lazy loading das 7 telas + galeria /design
        ├── Layout.tsx         # Casca adaptativa (Navigation Rail no desktop, barra inferior no celular, barra superior com MonthPicker)
        ├── links.ts           # Reexporta links.* e hooks useParam / useMonthParam
        ├── texts.ts           # Textos explicativos compartilhados (critério do mês, definição de gasto, média 6m)
        ├── api/
        │   ├── client.ts      # Cliente tipado de todas as rotas /api/*
        │   └── useApi.ts      # Hook com estado de carregamento, erro, recarga sem piscar a tela
        ├── state/
        │   ├── month.tsx      # Mês global selecionado
        │   ├── meta.tsx       # GET /api/meta, versão de sync e disparo de sincronização
        │   ├── basis.ts       # Critério 'bill' | 'purchase' sincronizado entre URL (?basis=) e localStorage
        │   ├── syncAction.ts  # Ação de sincronização com feedback visual e snackbar
        │   └── syncMessage.ts # Resumo amigável do que mudou na sincronização
        ├── design/            # Design System Material 3 Expressive próprio
        │   ├── tokens.css
        │   ├── base.css
        │   ├── color.ts       # SEED_COLOR, geração de esquema M3 e tons de categoria com contraste garantido
        │   ├── motion.ts      # Molas AndroidX, springEasing para Recharts e vigia de requestAnimationFrame
        │   ├── sharedTransition.ts # View Transitions lista -> detalhe
        │   ├── format.ts      # Formatação pt-BR (moeda, data, mês, relativo, prettyDescription)
        │   ├── hooks.ts       # useWindowSizeClass, useIsCompact, useInstantMotion, useScrollRestoration
        │   ├── providers/     # ThemeProvider, PrivacyProvider (Alt+H, useMoneyText), SnackbarProvider
        │   ├── components/    # 25 componentes base M3 Expressive
        │   ├── finance/       # Money, Delta, CategoryIcon/Chip/Dot, StatCard, SectionHeader, TransactionRow, RichText, InfoHint, SyncStatus
        │   ├── charts/        # DonutChart, StackedBars, AreaTrend, Sparkline, TreemapChart, HorizontalBars, TimelineChart, ChartReadout
        │   └── page/          # Page, Grid, GridItem, AsyncContent
        └── pages/
            ├── overview/
            ├── spending/
            ├── transactions/
            ├── cards/
            ├── investments/
            ├── insights/
            ├── connections/
            └── design/        # Galeria viva em /design
```

---

## 4. Schema Completo do Banco SQLite (`server/src/db/migrations.ts`)

O banco usa `PRAGMA journal_mode = WAL` e `PRAGMA foreign_keys = ON`. As migrações são forward-only, controladas por `PRAGMA user_version`.

### Migração 1: Schema inicial
```sql
CREATE TABLE items (
  item_id            TEXT PRIMARY KEY,
  institution        TEXT NOT NULL,
  connector_name     TEXT,
  status             TEXT,
  execution_status   TEXT,
  status_detail_json TEXT,
  source_updated_at  TEXT,
  consent_expires_at TEXT,
  last_sync_at       TEXT,
  alert              TEXT,
  raw_json           TEXT
);

CREATE TABLE accounts (
  account_id            TEXT PRIMARY KEY,
  item_id               TEXT NOT NULL REFERENCES items(item_id),
  kind                  TEXT NOT NULL CHECK (kind IN ('BANK','CREDIT')),
  subtype               TEXT,
  name                  TEXT NOT NULL,
  marketing_name        TEXT,
  number                TEXT,
  institution           TEXT NOT NULL,
  owner                 TEXT,
  tax_number            TEXT,
  currency              TEXT NOT NULL DEFAULT 'BRL',
  balance_cents         INTEGER NOT NULL DEFAULT 0,
  credit_limit_cents    INTEGER,
  available_limit_cents INTEGER,
  close_date            TEXT,
  due_date              TEXT,
  brand                 TEXT,
  level                 TEXT,
  last_tx_sync_at       TEXT,
  last_sync_at          TEXT,
  is_removed            INTEGER NOT NULL DEFAULT 0,
  raw_json              TEXT
);

CREATE TABLE transactions (
  local_key             TEXT PRIMARY KEY,
  pluggy_id             TEXT NOT NULL,
  provider_id           TEXT,
  account_id            TEXT NOT NULL REFERENCES accounts(account_id),
  date                  TEXT NOT NULL,
  description           TEXT NOT NULL,
  description_raw       TEXT,
  amount_original_cents INTEGER NOT NULL,
  currency              TEXT NOT NULL DEFAULT 'BRL',
  type                  TEXT NOT NULL CHECK (type IN ('DEBIT','CREDIT')),
  status                TEXT NOT NULL DEFAULT 'POSTED',
  spend_cents           INTEGER NOT NULL,
  operation_type        TEXT,
  pluggy_category_id    TEXT,
  pluggy_category       TEXT,
  bill_id               TEXT,
  bill_forecast_month   TEXT,
  installment_number    INTEGER,
  total_installments    INTEGER,
  purchase_date         TEXT,
  mcc                   TEXT,
  counterpart_name      TEXT,
  counterpart_doc       TEXT,
  statement_month       TEXT NOT NULL,
  category_id           TEXT NOT NULL DEFAULT '99000000',
  category_source       TEXT NOT NULL DEFAULT 'none',
  is_movement           INTEGER NOT NULL DEFAULT 0,
  movement_reason       TEXT,
  is_duplicate          INTEGER NOT NULL DEFAULT 0,
  duplicate_of          TEXT,
  is_removed            INTEGER NOT NULL DEFAULT 0,
  removed_at            TEXT,
  in_totals             INTEGER NOT NULL DEFAULT 1,
  -- expense | income | movement | excluded
  flow                  TEXT NOT NULL DEFAULT 'expense',
  refund_of             TEXT,
  is_new                INTEGER NOT NULL DEFAULT 0,
  first_seen_at         TEXT NOT NULL,
  last_seen_at          TEXT NOT NULL,
  raw_json              TEXT
);
CREATE INDEX tx_account_date ON transactions(account_id, date);
CREATE INDEX tx_statement_month ON transactions(statement_month);
CREATE INDEX tx_pluggy_id ON transactions(pluggy_id);

CREATE TABLE bills (
  bill_id               TEXT PRIMARY KEY,
  account_id            TEXT NOT NULL REFERENCES accounts(account_id),
  due_date              TEXT NOT NULL,
  closing_date          TEXT,
  total_cents           INTEGER NOT NULL,
  minimum_cents         INTEGER,
  finance_charges_cents INTEGER NOT NULL DEFAULT 0,
  payments_json         TEXT,
  raw_json              TEXT
);

CREATE TABLE investments (
  investment_id         TEXT PRIMARY KEY,
  item_id               TEXT NOT NULL REFERENCES items(item_id),
  name                  TEXT NOT NULL,
  type                  TEXT NOT NULL,
  subtype               TEXT,
  asset_class           TEXT NOT NULL,
  code                  TEXT,
  isin                  TEXT,
  issuer                TEXT,
  institution           TEXT NOT NULL,
  balance_cents         INTEGER NOT NULL,
  amount_cents          INTEGER,
  amount_original_cents INTEGER,
  quantity              REAL,
  unit_value            REAL,
  rate                  REAL,
  rate_type             TEXT,
  fixed_annual_rate     REAL,
  due_date              TEXT,
  issue_date            TEXT,
  purchase_date         TEXT,
  tax_exempt            INTEGER,
  status                TEXT,
  reference_date        TEXT,
  is_removed            INTEGER NOT NULL DEFAULT 0,
  last_sync_at          TEXT,
  raw_json              TEXT
);

CREATE TABLE investment_transactions (
  id               TEXT PRIMARY KEY,
  investment_id    TEXT NOT NULL REFERENCES investments(investment_id),
  date             TEXT NOT NULL,
  type             TEXT,
  movement_type    TEXT,
  quantity         REAL,
  unit_value       REAL,
  amount_cents     INTEGER,
  net_amount_cents INTEGER,
  description      TEXT,
  raw_json         TEXT
);
CREATE INDEX invtx_investment ON investment_transactions(investment_id, date);

CREATE TABLE account_snapshots (
  account_id      TEXT NOT NULL,
  date            TEXT NOT NULL,
  balance_cents   INTEGER NOT NULL,
  liability_cents INTEGER,
  PRIMARY KEY (account_id, date)
);

CREATE TABLE investment_snapshots (
  investment_id TEXT NOT NULL,
  date          TEXT NOT NULL,
  balance_cents INTEGER NOT NULL,
  amount_cents  INTEGER,
  quantity      REAL,
  PRIMARY KEY (investment_id, date)
);

CREATE TABLE categories (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  parent_id   TEXT REFERENCES categories(id),
  color       TEXT NOT NULL,
  sort        INTEGER NOT NULL,
  is_movement INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE category_rules (
  id          INTEGER PRIMARY KEY AUTOINCREMENT,
  pattern     TEXT NOT NULL,
  match_type  TEXT NOT NULL DEFAULT 'contains',
  category_id TEXT NOT NULL REFERENCES categories(id),
  created_at  TEXT NOT NULL,
  UNIQUE (pattern, match_type)
);

CREATE TABLE manual_overrides (
  local_key   TEXT PRIMARY KEY,
  category_id TEXT REFERENCES categories(id),
  is_movement INTEGER,
  reviewed_at TEXT,
  updated_at  TEXT NOT NULL
);

CREATE TABLE insights (
  id           TEXT PRIMARY KEY,
  type         TEXT NOT NULL,
  severity     TEXT NOT NULL,
  title        TEXT NOT NULL,
  body         TEXT NOT NULL,
  amount_cents INTEGER,
  relevance    REAL NOT NULL,
  link         TEXT,
  related_json TEXT,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL,
  dismissed_at TEXT,
  active       INTEGER NOT NULL DEFAULT 1
);

CREATE TABLE benchmarks (
  series TEXT NOT NULL,
  date   TEXT NOT NULL,
  value  REAL NOT NULL,
  PRIMARY KEY (series, date)
);

CREATE TABLE sync_log (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  started_at    TEXT NOT NULL,
  finished_at   TEXT,
  trigger       TEXT NOT NULL,
  status        TEXT NOT NULL,
  summary_json  TEXT,
  warnings_json TEXT,
  error         TEXT
);

CREATE TABLE settings (
  key   TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```

### Migrações 2 a 8
```sql
-- 2: estado explícito de alerta na conexão; estado de "nova" antes da primeira correção manual (para desfazer exato)
ALTER TABLE items ADD COLUMN alert_kind TEXT;
ALTER TABLE manual_overrides ADD COLUMN prev_is_new INTEGER;

-- 3: contas sem movimento ocultas automaticamente e override manual do usuário (NULL = automático, 1 = oculta, 0 = visível)
ALTER TABLE accounts ADD COLUMN auto_hidden INTEGER NOT NULL DEFAULT 0;
ALTER TABLE accounts ADD COLUMN hidden_override INTEGER;

-- 4: investimentos lançados à mão (ex.: posições em dólar no exterior)
CREATE TABLE manual_investments (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  name           TEXT NOT NULL,
  institution    TEXT NOT NULL,
  asset_class    TEXT NOT NULL,
  currency       TEXT NOT NULL DEFAULT 'USD' CHECK (currency IN ('USD','BRL','EUR')),
  quantity       REAL,
  value_cents    INTEGER NOT NULL,
  invested_cents INTEGER,
  reference_date TEXT NOT NULL,
  note           TEXT,
  start_date     TEXT NOT NULL,
  created_at     TEXT NOT NULL,
  updated_at     TEXT NOT NULL,
  deleted_at     TEXT
);

-- 5: investimentos manuais por ativo (ticker americano x quantidade, preço buscado automaticamente)
ALTER TABLE manual_investments ADD COLUMN kind TEXT NOT NULL DEFAULT 'value';
ALTER TABLE manual_investments ADD COLUMN ticker TEXT;
CREATE TABLE asset_info (
  ticker          TEXT PRIMARY KEY,
  name            TEXT,
  currency        TEXT NOT NULL,
  exchange        TEXT,
  instrument_type TEXT,
  updated_at      TEXT NOT NULL
);

-- 6: fluxos (aportes/resgates) de investimentos manuais para cálculo de rentabilidade TWR
CREATE TABLE manual_investment_flows (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  manual_id  INTEGER NOT NULL REFERENCES manual_investments(id),
  date       TEXT NOT NULL,
  flow_cents INTEGER NOT NULL
);

-- 7: flag de revisão para decisões automáticas duvidosas (ex.: 'ambiguous_bill_payment')
ALTER TABLE transactions ADD COLUMN review_flag TEXT;

-- 8: ícone opcional Material Symbols por categoria (usado por local-data/custom-categories.json)
ALTER TABLE categories ADD COLUMN icon TEXT;
```

---

## 5. Especificação Funcional das 7 Telas

A casca do app (`Layout.tsx`) possui trilho lateral no desktop (`>= 600px`, com rótulos visíveis `>= 1200px`), barra inferior no celular (`< 600px`, 4 itens principais + botão "Mais"), seletor de mês global no topo, botão de sincronização com indicador de saúde, botão de modo privacidade (`Alt+H`) e seletor de tema claro/escuro.

### Tela 1: Visão geral (`/`)
- **Alternador de critério no cabeçalho**: "Por fatura" (`basis=bill`, padrão) vs. "Por data da compra" (`basis=purchase`).
- **Card Herói — Patrimônio**: mostra `contas + investimentos − dívida nos cartões` (sempre a posição de **hoje**; se o mês selecionado não for o atual, o título muda para "Patrimônio hoje"). Exibe a variação em reais no mês (`<Delta>`), a quebra nas 3 parcelas e um `<Sparkline>` dos últimos 90 dias.
- **Card — Gasto do mês**: total gasto no mês, dica explicativa do critério (fatura vs. data da compra), variação percentual contra o mês anterior (no mês corrente, compara a **previsão de fechamento** contra o mês anterior para que um mês pela metade não pareça uma queda artificial), barra ondulada (`LinearProgress wavy`) da previsão de fechamento e minigráfico horizontal comparando: Até agora (ou Este mês), Previsão, Mês anterior e Média 6m.
- **Card — Gastos nos últimos 6 meses**: gráfico `<StackedBars>` mostrando o mês selecionado e os 6 meses anteriores, com a linha/valor da média dos 6 meses anteriores e leitura no topo ao passar o mouse/dedo (`<ChartReadout>`). Clicar numa barra abre aquele mês na tela de Gastos.
- **Card — Onde o dinheiro foi**: as 5 maiores categorias do mês com barra de participação proporcional e valor. Clicar abre o drill-down daquela categoria em Gastos.
- **Card — Próximas faturas**: próxima fatura de cada cartão (fechada ainda não vencida ou aberta estimada), com aviso visual se vence em até 3 dias.
- **Card — Para você olhar**: até 3 sugestões mais relevantes (no máximo 1 por tipo), clicáveis para a tela de destino.
- **Card — Contas**: estado da última sincronização de cada conta visível.

### Tela 2: Gastos (`/gastos`)
- **Alternador "Por fatura / Por data da compra"** sincronizado com a Visão geral (via URL `?basis=purchase` e `localStorage`).
- **Por categoria (Donut + Drill-down)**:
  - `<DonutChart>` à esquerda e lista segmentada à direita; passar o mouse na lista destaca a fatia na rosca e vice-versa.
  - Navegação em 3 níveis sincronizada com a URL (`?categoryId=`): **Todas as categorias** → **Subcategorias da categoria escolhida** (incluindo a linha `<mãe>:own` quando há gasto classificado direto na categoria-mãe) → **Transações daquele recorte**.
  - Clicar numa transação dentro do drill-down abre a `<TransactionSheet>` **ali mesmo**, permitindo recategorizar, criar regra ou confirmar sem sair da tela de Gastos (e recalculando o gráfico na hora).
  - Seção recolhida **"Fora dos gastos (N)"**: mostra categorias de movimentação ou com líquido negativo no mês (ex.: devoluções maiores que os gastos), explicando que não entram no total.
- **Evolução de 12 meses**: `<StackedBars>` empilhado por categoria-mãe nos últimos 12 meses até o mês selecionado, com scrub no cabeçalho (`<ChartReadout>`). Clicar em uma barra troca o mês global.
- **Entradas e saídas (Fluxo de caixa)**: mostra **Entrou** (salário, rendimentos/dividendos/JCP pagos em conta e Pix de terceiros), **Saiu** (o mesmo Gasto do mês) e **Saldo do mês** (`Entrou − Saiu`), além de gráfico diário e nota das movimentações que ficaram de fora.
- **Recorrências e assinaturas** (`#recorrencias`): lista de cobranças mensais detectadas automaticamente, soma mensal estimada, data da próxima cobrança (ou aviso de "atrasada") e selos "Nova" ou "Subiu X%".

### Tela 3: Transações (`/transacoes`)
- **Abas no topo**: "Todas" e "Para revisar" (com badge contador do período selecionado).
- **Busca e filtros na URL**: `<SearchBar>` com debounce (busca sem acento em descrição, categoria, conta e também **por valor em reais**, ex.: `187,40` ou `187`) + chips de menu (`<MenuChip>`) para Período (Mês selecionado, Últimos 30 dias, Todo o período, Escolher datas), Conta, Categoria e Tipo (Gastos, Receitas, Movimentações, Fora dos totais).
- **Fila "Para revisar"**:
  - Resumo no topo explicando quantos itens estão por qual motivo (`sem categoria`, `para conferir`, `novas`, `sugeridas pelo banco`).
  - Botão **"Confirmar todas"** (confirma em lote todas do filtro atual que já têm categoria, com botão **"Desfazer"** no snackbar).
  - Botão de check rápido em cada linha para confirmar individualmente com 1 clique.
- **Lista agrupada por dia**: cabeçalho sticky por dia com a soma dos gastos do dia.
- **Folha de detalhe (`<TransactionSheet>`)**:
  - Abre com transição compartilhada (`useSharedTransition`, View Transition API).
  - Switch **"Aplicar a todas de `<Estabelecimento>`"** no topo da seção de categoria.
  - Busca rápida de categorias/subcategorias e ordenação inteligente (`orderCategoriesFor`: categoria atual primeiro, depois até 3 categorias mais usadas naquele estabelecimento no histórico, depois categorias do dia a dia).
  - Ao clicar num chip de categoria, salva imediatamente, fecha/atualiza, pulsa a linha na cor da nova categoria e mostra snackbar com **"Desfazer"**.
  - Switch **"Não é gasto nem receita (movimentação)"** com opção de voltar à detecção automática.

### Tela 4: Cartões (`/cartoes`)
- Seletor de cartão no topo (quando houver mais de um cartão visível).
- **Fatura atual (estimada)**: valor acumulado no ciclo aberto, quantidade de compras, barra de limite usado do cartão (`usedCents / limitCents`), datas de fechamento e vencimento (com selo "estimado" quando calculadas pelo histórico de faturas) e botão para ver as transações do ciclo.
- **Histórico de faturas anteriores**: gráfico e lista usando a **soma das compras da fatura** (`purchasesCents`, o mesmo critério da fatura estimada). Ao clicar numa fatura, abre o detalhe inline abaixo do gráfico (com transição compartilhada) mostrando as transações daquela fatura e o "Total informado pelo banco" (`totalCents`), com aviso neutro caso haja divergência relevante.
- **Compromisso futuro (Parcelas)**: gráfico e lista dos próximos 12 meses somando parcelas já lançadas pelo banco e parcelas **projetadas** pelo painel a partir de `installmentNumber / totalInstallments`.

### Tela 5: Investimentos (`/investimentos`)
- **Resumo e botão "Adicionar investimento"**: total investido (sincronizados + lançados à mão em reais), quantidade de posições e botão para lançar posição manual.
- **Evolução e Rentabilidade**:
  - Quando há histórico (2+ dias de retratos): gráfico de área do **Patrimônio investido** e gráfico de linha de **Rentabilidade (base 100)** comparando Carteira vs. **CDI** vs. **IPCA** (com scrub `<ChartReadout>`).
  - Quando ainda não há histórico (1º dia de uso real): exibe uma faixa única e limpa explicando que os gráficos de evolução e rentabilidade começam a desenhar a partir do segundo dia de sincronização, subindo o bloco de Alocação.
- **Alocação**: abas para alternar entre **Por classe** (`<TreemapChart>` ou rosca), **Por instituição** e **Por emissor** (`<HorizontalBars>`).
- **Lista de posições**:
  - Agrupa aplicações repetidas do mesmo título (como as dezenas de CDBs das **Caixinhas do Nubank**) em uma única linha expansível (`N aplicações`).
  - Mostra selo **"Lançado à mão"** para posições manuais, exibindo o valor na moeda original (ex.: `US$ 2.166,99`) e abaixo o valor convertido em `R$` com a cotação PTAX usada (ou aviso de desatualizado após 30 dias para posições por valor).
  - Clicar numa posição abre o `<DetailSheet>` com os membros do grupo, histórico de movimentações (`BUY`, `SELL`, `INTEREST`, `TAX`) e botões de **Editar / Excluir** (com Desfazer) para investimentos manuais.
- **Linha do tempo de vencimentos**: `<TimelineChart>` para títulos de renda fixa com vencimento futuro (excluindo Caixinhas do Nubank, que têm liquidez diária).

### Tela 6: Sugestões (`/sugestoes`)
- Feed em coluna estreita (`width="narrow"`) com os cartões de sugestões ativas ordenados por relevância.
- Cada cartão exibe ícone com cor de severidade (`info`, `warning`, `alert`), título e corpo renderizados por `<RichText>` (para que valores em reais dentro do texto fiquem ocultos no modo privacidade!), link direto para a tela/filtro que comprova o alerta, botão **"Entendi"** (dispensa aquele alerta) e menu **"Não mostrar mais deste tipo"** (ambos com **"Desfazer"**).
- Rodapé mostrando tipos silenciados (com botão de reativar) e opção de ver sugestões já dispensadas.

### Tela 7: Conexões e ajustes (`/conexoes`)
- **Banner da fonte de dados**: mostra claramente se o painel está em **Dados fictícios (demonstração)** ou **Dados reais, pela Pluggy**.
- **Card — Buscar novidades**: explica que a Pluggy atualiza os bancos ~1x por dia, botão "Buscar novidades" com animação e resumo da última execução (`N transações novas`, avisos, etc.).
- **Card — Validade da autorização**: barra de progresso dos 365 dias da autorização Meu Pluggy e alerta quando faltam 30 dias ou menos.
- **Contas conectadas por instituição**: cartões por banco (com nome real inferido: Nubank, XP, Santander, etc.), status de saúde, saldo de cada conta/cartão, botão de olho (`visibility` / `visibility_off`) para ocultar ou mostrar qualquer conta (com Desfazer) e gaveta expansível **"Contas sem movimento (N)"** para as contas que o painel ocultou sozinho.
- **Regras de categoria**: lista de todas as regras criadas pelo usuário, mostrando quantas transações cada regra classifica hoje (`matchCount`) e botão de apagar (com confirmação e Desfazer).
- **Preferências**: seletor de Tema (Automático, Claro, Escuro) e switch do Modo Privacidade (`Alt+H`).
- **Histórico de buscas**: lista das últimas 10 execuções de sincronização.
