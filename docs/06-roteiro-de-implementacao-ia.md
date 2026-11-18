# 06. Roteiro de Implementação em 6 Fases para Agentes de IA

Este documento é o plano de execução para o Agente de IA construir a aplicação do zero até a versão final.

> **Instrução para o Agente**: Execute **uma fase por vez**. Ao concluir uma fase, rode `npm run typecheck` e `npm test`, atualize a seção "Estado atual" do `AGENTS.md` e faça um commit em inglês no imperativo antes de iniciar a fase seguinte.

---

## Fase 1: Fundação, Banco de Dados, Domínio e Sincronização

### O que construir
1. **Configuração inicial**:
   - `package.json` (conforme `docs/01-arquitetura-e-especificacao.md`), `server/tsconfig.json`, `web/tsconfig.json`, `vitest.config.ts`, `scripts/dev.mjs` e `scripts/backup.mjs`.
   - `shared/types.ts` e `shared/links.ts` (conforme `docs/04-contrato-da-api.md`).
   - `server/config/insights.json` com os limiares das 12 sugestões.
2. **Camada de Domínio Pura (`server/src/domain/`)**:
   - `money.ts` (`toCents`, `normalizeSpend`, `normalizeText`), `dates.ts` (`toBrasiliaDate`, `todayBrasilia`, `addDays`, `addMonths`, `diffDays`, `monthRange`, `dateRange`).
   - `identity.ts` (`baseLocalKey` com `p_` para `providerId` e `f_` para fallback SHA-1 de conta+data+valor+descrição normalizada, `assignLocalKeys` com sufixo `_2`, `_3` para repetidas no mesmo dia).
   - `categories.ts` (`CATEGORY_TREE` com as 22 categorias padrão, `parseCustomCategories`, `flatCategories`, `categoryTree`, `mapPluggyCategory`).
   - `categorize.ts` (`BRAND_RULES`, `KEYWORD_RULES`, `userRuleCategory` onde o padrão mais longo vence e olha também `counterpartName`, `categorize` em camadas, `suggestedRulePattern`).
   - `movements.ts` (`isBillPayment` com proteção para `"compra no debito"`, `isAmbiguousBillPayment`, `isInvestmentMovement`, `isInvestmentIncome`, `pairOwnTransfers`, `mentionsOwner`, `isRefund`, `matchRefunds`, `detectMovement`, `applyMovementOverride`).
   - `flow.ts` (`classifyFlow`: `expense | income | movement | excluded`).
   - `cards.ts` (`cycleFor`, `cycleFromBills`, `statementMonthFor`, `findFullPurchaseDuplicates`, `cardDebtCents`, `projectInstallments`).
   - `institutions.ts` (`INSTITUTION_PATTERNS`, `inferItemInstitution`, `friendlyAccountName`, `titleCase`).
   - `visibility.ts` (`autoHiddenAccounts`).
   - `investments.ts` + `investment-names.ts` + `fx.ts` (`groupPositions` sem vencimento na chave, tratamento de `"Caixinhas do Nubank"` emitidas pela Nu Financeira, cálculo TWR, conversão PTAX).
   - `spending.ts` (`countedParents`, `spendingTotals`, `detectRecurring`, `findDuplicateCharges`, `forecastMonthEnd`).
   - `insights.ts` (geração pura das 12 sugestões com `{{money:N}}` e `splitInsightText`).
3. **Banco SQLite (`server/src/db/`)**:
   - `migrations.ts` (as 8 migrações de `docs/01-arquitetura-e-especificacao.md`).
   - `index.ts` (`openDatabase`, `migrate`, `loadCustomCategoriesFile`, `seedCategories`, `getSetting`, `setSetting`).
   - `rows.ts` (`loadAccounts`, `loadBills`, `loadTransactions`, `loadRules`, `loadOverrides`, `knownCategoryIds`, `ACTIVE_INVESTMENT_SQL`, `COUNTED_SNAPSHOT_SQL`, `VISIBLE_ACCOUNT_SQL`).
4. **Fontes de Dados (`server/src/sources/`)**:
   - `types.ts` (`DataSource` interface), `mapping.ts`.
   - `mock.ts` + `mock-benchmarks.ts`: gerador determinístico (~14 meses desde `2025-09-01` até `today`, semente `20261110`) com 3 instituições fictícias (`Banco Aurora`, `Banco Horizonte` que manda compra cheia duplicando parcelas, e `Vértice Investimentos`), salários, contas, Pix entre contas próprias, compra no débito via carteira que o banco chama de pagamento de cartão, JCP classificado pelo banco em empréstimos, Pix de terceiros, estorno, assinatura que subiu de preço, assinatura nova, despesa fora do padrão, cobrança duplicada e 14 investimentos com snapshots diários retroativos.
   - `pluggy.ts`: cliente real usando `pluggy-sdk` e `fetchAllTransactions` (rota `/v2/transactions` com cursor).
5. **Orquestrador de Sincronização (`server/src/sync/`)**:
   - `accounts.ts`, `derive.ts`, `benchmarks.ts` (SGS Banco Central com cache), `prices.ts` (Yahoo Finance com cache e modo mock), `sync.ts` (`runSync`, `storeTransactions` com remoção reversível e proteção contra leitura vazia, `recomputeLocal`, `refreshMarketData`).

### Testes obrigatórios da Fase 1 (`npm test`)
- `server/test/rules.test.ts`: normalização de sinal, datas de Brasília, chaves locais (`p_`, `f_`, `_2`), árvore de categorias + `parseCustomCategories`, todas as camadas de categorização (`manual > rule > refund > investmentIncome > brand > pluggy > movement > keyword > none`), detecção de movimentações, pareamento de transferências próprias e estornos.
- `server/test/cards-investments.test.ts` e `server/test/spending-insights.test.ts`: ciclos de cartão, duplicadas de compra cheia, projeção de parcelas, agrupamento de posições, TWR, recorrências, cobranças duplicadas e as 12 sugestões.
- `server/test/sync.test.ts`: sincronização inicial e incremental (janela de 45 dias), remoção reversível e garantia de que uma leitura vazia (`emptyRead`) **nunca** remove dados existentes e gera o alerta correto (`empty_read` ou `auth_expired`).

---

## Fase 2: Serviços, Investimentos Manuais e API REST

### O que construir
1. **Serviços (`server/src/services/`)**:
   - `common.ts` (`REVIEW_REASON_SQL`, `NEEDS_REVIEW_SQL`, `categoriesById`, `toTransactionDto`, `HttpError`).
   - `overview.ts`, `spending.ts` (suportando `basis = 'bill' | 'purchase'`), `transactions.ts` (filtros, busca por texto e por valor numérico, `recategorize`, `clearCategory`, `setMovement`, `confirmReviewed`, `undoConfirm`, CRUD de regras com contagem de matches).
   - `cards.ts`, `investments.ts`, `manual-investments.ts` (CRUD de investimentos manuais por ativo `asset` e por valor `value`, seed de 2 posições manuais no modo mock: `VOO` e `Caixa em dólar`, snapshots manuais diários em reais e registro de fluxos em `manual_investment_flows`).
   - `insights.ts` (listagem, dismiss `one | type`, restore, unmute), `connections.ts`.
2. **Servidor HTTP (`server/src/api/app.ts` e `server/src/index.ts`)**:
   - Todas as rotas documentadas em `docs/04-contrato-da-api.md`.
   - `server/src/index.ts`: serve `/api`, serve `dist/web` (quando compilado), roda `runSync` no boot se a última sincronização tiver mais de 6 h, ou `recomputeLocal` caso contrário.

### Testes obrigatórios da Fase 2 (`npm test`)
- `server/test/mock-api.test.ts` e `server/test/integration-api.test.ts`: testa todas as rotas HTTP em memória, valida códigos de erro `400/404/409` e testa todos os fluxos de **Desfazer** (recategorizar + regra, confirmar fila de revisão e desfazer, dispensar sugestão e restaurar).
- `server/test/real-shape.test.ts`: testa uma fonte no formato exato do conector `MeuPluggy` (conector proxy, contas sem movimento e gêmea vazia, cartão sem data de fechamento, posições `TOTAL_WITHDRAWAL` e 3 CDBs de caixinha do Nubank com vencimentos diferentes).
- `server/test/manual-investments.test.ts`: criar, editar quantidade (gerando fluxo sem distorcer rentabilidade), excluir, restaurar e converter com PTAX.
- `server/test/ux-review.test.ts`: prova a invariante `lista === donut === barra mensal === visão geral === fluxo de caixa` nos últimos 3 meses tanto para `basis=bill` quanto para `basis=purchase`.

---

## Fase 3: Design System Material 3 Expressive e Galeria `/design`

### O que construir
1. **Fundação visual (`web/src/design/`)**:
   - `color.ts` + `web/scripts/generate-tokens.ts` (`npm run tokens` → gera `tokens.css`), `base.css` (incluindo regras para `[data-frames="paused"] [data-settle]` e `::view-transition-group(shared-detail)`).
   - `motion.ts` (molas, `springEasing`, vigia de `requestAnimationFrame`), `sharedTransition.ts`, `format.ts` (`formatMoney`, `prettyDescription`, etc.), `hooks.ts`.
   - Providers: `ThemeProvider`, `PrivacyProvider` (`Alt+H`, `useMoneyText`), `SnackbarProvider` (`showUndo`).
2. **Componentes**:
   - Os 25 componentes base em `web/src/design/components/` (incluindo `gestures.ts` para a folha inferior arrastável e `PullToRefresh`).
   - Os componentes financeiros em `web/src/design/finance/`.
   - Os 7 gráficos Recharts + `ChartReadout` em `web/src/design/charts/`.
   - Layout de página (`Page`, `Grid`, `GridItem`, `AsyncContent`) em `web/src/design/page/Page.tsx` e a casca `web/src/Layout.tsx`.
3. **Galeria viva (`web/src/pages/design/DesignPage.tsx`)**:
   - Rota `/design` exibindo todas as cores, tipografia, botões, chips, campos, folhas, componentes financeiros e gráficos, com alternador "Lado a lado" (Claro + Escuro simultâneos).

### Testes obrigatórios da Fase 3 (`npm test`)
- `web/src/design/design.test.ts`: verifica contraste WCAG (`>= 3:1` em marcas e `>= 4.5:1` em texto de chips e superfícies) para todas as cores de categoria nos temas claro e escuro, além de `springEasing`.
- `web/src/design/format.test.ts`, `gestures.test.ts`, `tokens-defined.test.ts` (verifica que nenhuma variável CSS `var(--...)` usada nos arquivos `.css` ficou sem definição em `tokens.css` / `base.css`).

---

## Fase 4: Implementação das 7 Telas

> **Dica para Agentes com Subagentes**: Como o contrato da API (`shared/types.ts`, `shared/links.ts`, `web/src/api/client.ts`) e o Design System (`web/src/design/`) já estão prontos e testados ao fim da Fase 3, as 7 telas são independentes entre si e podem ser construídas em paralelo por subagentes ou sequencialmente.

### O que construir
1. **Cliente e Estado Global (`web/src/api/` e `web/src/state/`)**:
   - `client.ts`, `useApi.ts`, `month.tsx`, `meta.tsx`, `basis.ts`, `syncAction.ts`, `syncMessage.ts`, `texts.ts`.
2. **As 7 Páginas (`web/src/pages/`)**:
   - `overview/OverviewPage.tsx`
   - `spending/SpendingPage.tsx` (`CategoryBreakdown.tsx`, `MonthlyBlock.tsx`, `CashflowBlock.tsx`, `RecurringBlock.tsx`, `BasisToggle.tsx`, `monthlyCap.ts`)
   - `transactions/TransactionsPage.tsx` (`TransactionDetail.tsx`, `useTransactionActions.ts`, `useTransactionList.ts`, `filters.ts`, `categoryOrder.ts`)
   - `cards/CardsPage.tsx`
   - `investments/InvestmentsPage.tsx` (`ManualInvestmentForm.tsx`, `PositionDetail.tsx`, `helpers.ts`)
   - `insights/InsightsPage.tsx` (`insightMeta.ts`)
   - `connections/ConnectionsPage.tsx`
3. **Rotas (`web/src/routes.tsx`)**: lazy loading (`React.lazy`) para cada uma das 7 telas e `/design`.

### Testes obrigatórios da Fase 4 (`npm test`)
- `web/src/pages/transactions/categoryOrder.test.ts` e `filters.test.ts`.
- `web/src/pages/spending/monthlyCap.test.ts` e `web/src/pages/investments/helpers.test.ts`.

---

## Fase 5: Polimento de UX, Responsividade e Verificação

### Checklist de verificação obrigatória
1. Rode `npm run typecheck`, `npm test` e `npm run build`. Tudo deve passar com zero erros e zero warnings de tipos.
2. Suba o servidor (`npm run dev`) e verifique no navegador (ou via ferramenta de inspeção de browser):
   - **Desktop (`1280x800`) e Mobile (`375x812`)**: zero rolagem horizontal acidental no celular; barra inferior funcionando no celular e Navigation Rail no desktop.
   - **Tema Claro e Tema Escuro**: contraste legível em todos os cards, badges e tooltips.
   - **Modo Privacidade (`Alt+H`)**: todos os valores monetários em cards, listas, gráficos e dentro dos textos de sugestões (`<RichText>`), além dos números de conta (`<PrivateText>`), ficam ocultos.
   - **Fluxos interativos**: drill-down em Gastos até abrir a transação; recategorizar com "Desfazer"; confirmar fila de revisão com "Desfazer"; alternar "Por fatura / Por data da compra"; adicionar/editar/excluir investimento manual com "Desfazer"; scrub do mouse nos gráficos de evolução, 12 meses e 6 meses.

---

## Fase 6: Conexão com Dados Reais (Meu Pluggy) e Customização Pessoal

1. Oriente o usuário a seguir o passo a passo da **Seção 2 de `docs/02-pluggy-na-pratica.md`** para conectar os bancos dele no Meu Pluggy e preencher o `.env` (`DATA_SOURCE=pluggy`, `DATA_DIR=local-data/real`, `PLUGGY_CLIENT_ID`, `PLUGGY_CLIENT_SECRET`, `PLUGGY_ITEM_IDS`, `PLUGGY_AUTHORIZED_AT`).
2. Suba o aplicativo com os dados reais e ajude o usuário a:
   - Criar suas categorias personalizadas em `local-data/custom-categories.json` (se desejar);
   - Revisar a fila **"Para revisar"** do mês atual e criar regras para os estabelecimentos e Pix recorrentes dele;
   - Lançar eventuais investimentos no exterior na tela de Investimentos;
   - Configurar o `BACKUP_DIR` no `.env` apontando para uma pasta sincronizada na nuvem (Google Drive / OneDrive / iCloud) e testar `npm run backup`.
