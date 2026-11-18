# 03. Regras de Negócio, Categorização e Cálculos Contábeis

Este documento especifica as funções puras de `server/src/domain/`. Toda regra descrita aqui deve ser implementada como função pura (sem acesso direto ao banco ou rede) e coberta por testes unitários em `server/test/`.

---

## 1. Dinheiro, Sinais e Datas (`money.ts` e `dates.ts`)

### 1.1. Centavos inteiros (`toCents`)
Todo valor monetário que entra da API da Pluggy (que vem em reais com casas decimais, ex.: `10000.76`) é convertido imediatamente na borda (`mapping.ts`) para centavos inteiros:
```ts
export function toCents(amount: number | null | undefined): number {
  if (amount === null || amount === undefined || Number.isNaN(amount)) return 0;
  return Math.round(amount * 100);
}
```

### 1.2. Normalização de sinal (`normalizeSpend`)
Na Pluggy, uma compra na conta corrente vem com `amount < 0` e `type = 'DEBIT'`, mas uma compra no cartão de crédito vem com `amount > 0` e `type = 'DEBIT'`.
O painel normaliza tudo pela coluna `type` na convenção **"gasto é positivo, entrada/crédito é negativo"**:
```ts
export function normalizeSpend(type: 'DEBIT' | 'CREDIT', amountOriginalCents: number): number {
  const abs = Math.abs(amountOriginalCents);
  return type === 'DEBIT' ? abs : -abs;
}
```
- `spend_cents > 0`: saída / compra / débito.
- `spend_cents < 0`: entrada / receita / estorno / pagamento recebido no cartão.
- `amount_original_cents`: guarda o valor em centavos com o sinal exato que a Pluggy enviou.

### 1.3. Normalização de texto (`normalizeText`)
Usada para comparações de regras, nomes de estabelecimentos, busca e inferência de instituições:
```ts
export function normalizeText(text: string): string {
  return (text ?? '')
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, ' ')
    .replace(/\s+/g, ' ')
    .trim();
}
```

### 1.4. Datas no fuso de Brasília (`toBrasiliaDate`)
A Pluggy envia timestamps em UTC (`2026-11-10T02:00:00.000Z`). Subtraia 3 horas (`UTC-3`) antes de extrair `AAAA-MM-DD`, para que compras feitas à noite no Brasil não caiam no dia seguinte:
```ts
export function toBrasiliaDate(input: string | Date): string {
  if (typeof input === 'string' && /^\d{4}-\d{2}-\d{2}$/.test(input)) return input;
  const ms = typeof input === 'string' ? Date.parse(input) : input.getTime();
  return new Date(ms - 3 * 3600_000).toISOString().slice(0, 10);
}
```

---

## 2. Árvore de Categorias Padrão + Categorias Personalizadas (`categories.ts`)

O painel usa IDs de 8 dígitos no padrão da Pluggy (`NNMM0000`, onde `NN` é o nível 1 e `MM` é o nível 2). A categoria-mãe de qualquer ID é `id.slice(0, 2) + '000000'`.

### 2.1. As 22 categorias-mãe padrão (`CATEGORY_TREE`)
Cada categoria-mãe tem uma cor hexadecimal fixa (escolhida para que os tons derivados pelo Material Color Utilities mantenham contraste `>= 3:1` em marca e `>= 4.5:1` em texto de chip nos temas claro e escuro):

| ID | Nome | Cor | Movimentação? | Subcategorias (`[id, nome]`) |
|---|---|---|---|---|
| `01000000` | Renda | `#2E7D32` | não | `01010000` Salário, `01020000` Aposentadoria, `01030000` Atividade autônoma, `01040000` Auxílio do governo, `01050000` Renda não recorrente, `01060000` Rendimentos de investimentos |
| `02000000` | Empréstimos e financiamentos | `#6D4C41` | não | `02010000` Juros e cheque especial, `02020000` Juros cobrados, `02030000` Empréstimos, `02040000` Financiamento imobiliário, `02050000` Financiamento de veículo, `02060000` Financiamento estudantil |
| `03000000` | Investimentos | `#00838F` | **sim** | `03010000` Aplicação automática, `03020000` Renda fixa, `03030000` Fundos, `03040000` Renda variável, `03050000` Margem, `03060000` Proventos e juros, `03070000` Previdência |
| `04000000` | Transferência entre contas próprias | `#78909C` | **sim** | `04010000` Mesma titularidade - dinheiro, `04020000` Mesma titularidade - PIX, `04030000` Mesma titularidade - TED |
| `05000000` | Transferências | `#546E7A` | não | `05010000` Boleto, `05020000` Dinheiro, `05030000` Cheque, `05040000` DOC, `05050000` Câmbio, `05060000` Transferência interna, `05070000` PIX, `05080000` TED, `05090000` Transferência para terceiros, `05100000` Pagamento de cartão |
| `06000000` | Obrigações legais | `#8D6E63` | não | `06010000` Saldo bloqueado, `06020000` Pensão |
| `07000000` | Serviços | `#5E35B1` | não | `07010000` Telecomunicações, `07020000` Educação, `07030000` Academia e bem-estar, `07040000` Ingressos e eventos |
| `08000000` | Compras | `#D81B60` | não | `08010000` Compras online, `08020000` Eletrônicos, `08030000` Pet, `08040000` Roupas, `08050000` Infantil e brinquedos, `08060000` Livraria, `08070000` Artigos esportivos, `08080000` Papelaria, `08090000` Cashback |
| `09000000` | Serviços digitais | `#3949AB` | não | `09010000` Jogos, `09020000` Streaming de vídeo, `09030000` Streaming de música, `09040000` Apps e armazenamento |
| `10000000` | Mercado | `#43A047` | não | `10010000` Supermercado, `10020000` Padaria e hortifrúti |
| `11000000` | Alimentação | `#FB8C00` | não | `11010000` Restaurantes e bares, `11020000` Delivery |
| `12000000` | Viagem | `#039BE5` | não | `12010000` Passagens aéreas, `12020000` Hospedagem, `12030000` Programas de milhas, `12040000` Ônibus |
| `13000000` | Doações | `#AD1457` | não | `13010000` Doações |
| `14000000` | Apostas | `#6A1B9A` | não | `14010000` Loteria, `14020000` Apostas online |
| `15000000` | Impostos | `#795548` | não | `15010000` Imposto de renda, `15020000` Impostos sobre investimentos, `15030000` IOF |
| `16000000` | Tarifas bancárias | `#9E9D24` | não | `16010000` Tarifa de conta, `16020000` Tarifa de transferência e saque, `16030000` Tarifas de cartão |
| `17000000` | Moradia | `#F4511E` | não | `17010000` Aluguel, `17020000` Casa e utensílios, `17030000` IPTU, `17040000` Água, `17050000` Energia, `17060000` Gás, `17070000` Condomínio |
| `18000000` | Saúde | `#E53935` | não | `18010000` Dentista, `18020000` Farmácia, `18030000` Ótica, `18040000` Clínicas e exames |
| `19000000` | Transporte | `#1E88E5` | não | `19010000` Transporte público, `19020000` Aluguel de carro, `19030000` Bicicleta, `19050000` Táxi e aplicativos, `19060000` Combustível, `19070000` Estacionamento, `19080000` Pedágio, `19090000` IPVA e taxas, `19100000` Manutenção do veículo, `19110000` Multas |
| `20000000` | Seguros | `#00897B` | não | `20010000` Seguro de vida, `20020000` Seguro residencial, `20030000` Plano de saúde, `20040000` Seguro de veículo |
| `21000000` | Lazer | `#C0CA33` | não | `21010000` Lazer |
| `99000000` | Sem categoria | `#9E9E9E` | não | *(vazio)* |

### 2.2. Categorias personalizadas (`local-data/custom-categories.json`)
Para que cada usuário possa ter suas próprias categorias sem editar o código-fonte:
- Na abertura do banco (`openDatabase`), o servidor procura `custom-categories.json` dentro de `DATA_DIR` (ou na pasta pai `local-data/`).
- Formato do JSON:
  ```json
  [
    {
      "id": "90000000",
      "name": "Hobbies",
      "color": "#558B2F",
      "icon": "sports_esports",
      "children": [
        { "id": "90010000", "name": "Equipamentos" }
      ]
    }
  ]
  ```
- Use IDs na faixa `80000000` a `98000000` para evitar colisão com IDs da Pluggy (`01`..`21`) e com Sem categoria (`99000000`).
- `flatCategories(custom)` insere as categorias personalizadas logo antes de `99000000` (Sem categoria), faz upsert na tabela `categories` (incluindo a coluna `icon`) e `GET /api/categories` lê diretamente da tabela `categories`.

---

## 3. Categorização em Camadas (`categorize.ts`)

Cada transação recebe uma categoria (`categoryId`) e a origem da decisão (`categorySource`: `'manual' | 'rule' | 'refund' | 'pluggy' | 'keyword' | 'none'`), avaliadas **exatamente nesta ordem**:

1. **Correção manual do usuário (`manual`)**: se existir `manual_overrides.category_id` para a `local_key` da transação, ela vence tudo.
2. **Regra do usuário (`rule`)**: procura se alguma regra de `category_rules` está contida em `normalizeText(tx.description + ' ' + (tx.counterpartName ?? ''))` (incluir `counterpartName` é essencial porque descrições de Pix e boleto muitas vezes trazem apenas `"Pagamento efetuado"` e o nome real do recebedor fica em `counterpartName`). Se mais de uma regra casar, **vence a regra com o padrão (`pattern`) mais longo**.
3. **Estorno pareado (`refund`)**: se a transação é um estorno pareado com uma compra original (`matchRefunds`), herda a categoria efetiva da compra original.
4. **Rendimentos de investimentos pagos em conta (`keyword` → `01060000`)**: se `isInvestmentIncome(tx)` for verdadeiro numa conta bancária (e não for movimentação), classifica em `01060000` (*Rendimentos de investimentos*), ignorando a categoria do banco.
5. **Regras de marcas conhecidas (`BRAND_RULES`, `keyword`)**: se for um gasto (`spendCents > 0`) e **não** for movimentação, testa `BRAND_RULES` antes da categoria do banco:
   - `uber eats`, `99 food`, `99food` → `11020000` (Delivery)
   - `uber`, `99 app/pop/taxi/tecnologia`, `cabify`, `indriver` → `19050000` (Táxi e aplicativos)
   - `apple music`, `youtube music`, `amazon music`, `spotify`, `deezer`, `tidal` → `09030000` (Streaming de música)
   - `netflix`, `disney`, `hbo max`, `hbo`, `max com`, `prime video`, `globoplay`, `youtube premium`, `paramount`, `crunchyroll` → `09020000` (Streaming de vídeo)
   - `apple com bill`, `itunes` → `09040000` (Apps e armazenamento)
6. **Categoria enviada pela Pluggy (`pluggy`)**: mapeia `tx.pluggyCategoryId` para o nível 2 (`NNMM0000`) ou nível 1 (`NN000000`). **Exceção crítica**: se a Pluggy enviou `05100000` (*Pagamento de cartão*), ela **só** é aceita se o detector de movimentações do painel também confirmou `movementReason === 'bill_payment'`. Caso contrário (ex.: compra no débito via carteira digital), é ignorada!
7. **Categoria padrão de movimentação (`keyword`)**: se a transação foi detectada como movimentação e chegou até aqui sem categoria:
   - `bill_payment` → `05100000` (Pagamento de cartão)
   - `own_transfer` → `04000000` (Transferência entre contas próprias)
   - `investment` → `03000000` (Investimentos)
8. **Regras de palavras-chave embutidas (`KEYWORD_RULES`, `keyword`)**: ~50 expressões regulares cobrindo salário, iFood/Rappi, restaurantes/padarias/mercados, postos de combustível, estacionamento, pedágio, transporte público, farmácias, médicos/exames, planos de saúde, aluguel, condomínio, luz/água/gás/internet, academia, cursos, lojas online, passagens/hotéis, tarifas, IOF/DARF.
9. **Fallback (`none`)**: `99000000` (*Sem categoria*).

---

## 4. O que NÃO é Gasto: Detecção de Movimentações (`movements.ts`)

Transações marcadas como `is_movement = 1` ficam fora dos totais de gasto e receita (`flow = 'movement'`, `in_totals = 0`), a menos que o usuário force o contrário em `manual_overrides.is_movement`.

Existem 3 tipos automáticos de movimentação:

### 4.1. Pagamento de fatura de cartão (`bill_payment`)
- **No cartão (`kind === 'CREDIT'`)**: qualquer transação de crédito (`type === 'CREDIT'`) que tenha `operationType === 'PAGAMENTO_FATURA'`, texto com `pagamento/pagto ... fatura/cartao` ou `pagamento recebido`, **OU** que não seja estorno e cujo valor absoluto seja exatamente igual ao `totalCents` de uma fatura fechada daquele mesmo cartão em uma janela de até 20 dias do vencimento.
- **Na conta bancária (`kind === 'BANK'`)**: qualquer débito (`type === 'DEBIT'`) que:
  1. **NÃO** contenha `"compra no debito"` / `"compras no debito"` no texto (regra de proteção contra carteiras digitais!); e
  2. Contenha texto de pagamento de fatura/cartão, **OU** venha categorizado pelo banco como `05100000` (*Credit card payment*), **OU** tenha valor exatamente igual ao total de uma fatura fechada do painel em até 3 dias da data de vencimento.
- **Fila de revisão para pagamento ambíguo (`isAmbiguousBillPayment`)**: todo débito em conta bancária que o banco classificou como `05100000` (*Credit card payment*), mas cujo texto **não** contém as palavras `"fatura"` ou `"cartao"`, recebe `review_flag = 'ambiguous_bill_payment'`.

### 4.2. Transferência entre contas próprias (`own_transfer`)
Uma transação de conta bancária é transferência própria se:
1. **Pareamento entre contas do painel (`pairOwnTransfers`)**: existe outra transação em **outra** conta bancária do painel com o **mesmo valor absoluto**, **sinais opostos** (`DEBIT` numa conta, `CREDIT` na outra) e diferença de data de **no máximo 2 dias** (pareando primeiro as datas mais próximas, cada transação usada no máximo uma vez); **OU**
2. A categoria da Pluggy começa com `04` (*Same person transfer*); **OU**
3. **Menção ao titular (`mentionsOwner`)**: a descrição ou `counterpartName` contém o nome do titular (extraído de `accounts.owner`, com `>= 5` caracteres), ou a descrição / `counterpartDoc` contém os 11 dígitos do CPF do titular (`accounts.taxNumber`).

### 4.3. Aplicação e resgate de investimento (`investment`)
Na conta bancária, se **não** for rendimento pago em conta (`!isInvestmentIncome(tx)`), é movimentação de investimento quando:
- `operationType === 'RESGATE_APLIC_FINANCEIRA'`; ou
- `pluggyCategoryId` começa com `03` (`Investments`); ou
- O texto contém palavras de aplicação/resgate (`aplicacao`, `resgate`, `tesouro direto`, `investimento`, `cdb`, `lci`, `lca`, `previdencia`).

---

## 5. Classificação de Fluxo (`flow.ts`) e Definição Única de Gasto e Receita

### 5.1. O campo `flow` de cada transação
Avaliado nesta ordem:
1. Se `isRemoved` ou `isDuplicate` → `'excluded'` (`inTotals = false`).
2. Se for conta bancária (`kind === 'BANK'`), `status === 'PENDING'` e `date > today` (agendamento futuro) → `'excluded'` (`inTotals = false`).
3. Se `isMovement` → `'movement'` (`inTotals = false`).
4. Se for estorno (`isRefund`: `type === 'CREDIT'` com `operationType === 'ESTORNO'` ou palavra `estorno/reembolso/devolucao/chargeback`, ou pareado por `matchRefunds`) → `'expense'` (`inTotals = true`, com `spend_cents < 0`, abatendo a categoria da compra!).
5. Se for conta bancária e `type === 'CREDIT'` em uma das categorias de entrada:
   - Categoria-mãe `01000000` (*Renda*, que inclui Salário e Rendimentos de investimentos);
   - Categoria-mãe `05000000` (*Transferências*, ex.: Pix ou TED recebido de terceiros);
   - Sem categoria (`99000000`);
   então → `'income'` (`inTotals = true`).
6. Caso contrário → `'expense'` (`inTotals = true`). (Nota: créditos em cartão de crédito que não são pagamento de fatura sempre entram como `'expense'` negativo para abater o gasto do cartão.)

### 5.2. A Definição Única de "Gasto do Mês" (`countedParents` em `spending.ts`)
Para evitar que números discordem entre telas:
- Agrupe todas as transações com `flow === 'expense'` do mês de referência (`statement_month` quando `basis = 'bill'`, ou `substr(date, 1, 7)` quando `basis = 'purchase'`) pela **categoria-mãe** (`parentOf(category_id)`).
- Uma categoria-mãe **entra no Gasto do Mês (`countedParents`)** se e somente se:
  1. **Não** for uma categoria-mãe de movimentação (`03000000` Investimentos, `04000000` Transferência entre contas próprias); **E**
  2. A soma líquida (`SUM(spend_cents)`) dessa categoria-mãe no mês for **estritamente positiva (`> 0`)**.
- Categorias-mãe com soma líquida `<= 0` no mês (ex.: um mês em que um estorno superou as compras daquela categoria, ou um crédito de empréstimo) ou de movimentação vão para a lista `outside` (*"Fora dos gastos"*) e **não** entram em `totalCents`.
- **Invariante testada automaticamente**: para qualquer mês e qualquer `basis` (`bill` ou `purchase`), `GET /api/spending.totalCents === soma(GET /api/spending.categories) === barra do mês em GET /api/spending/monthly === GET /api/overview.spending.monthCents === GET /api/spending/cashflow.expenseCents`.

### 5.3. Média de 6 meses e Previsão de fechamento do mês
- **Média de 6 meses (`average6mCents`)**: é sempre a média aritmética simples dos **6 meses imediatamente anteriores** ao mês selecionado (`month - 6` até `month - 1`). O próprio mês selecionado nunca entra na média.
- **Previsão de fechamento (`forecastMonthEnd`)**: no mês corrente (no dia `D`), olha para os 6 meses anteriores que tiveram gasto (`totalCents > 0`), calcula quanto cada um desses meses gastou **depois do dia `D`** (`Math.max(0, totalCents - upToDayCents)`), tira a média desse "restante histórico" e soma ao gasto acumulado até hoje (`spentSoFarCents + remainingAvg`).

---

## 6. Fila "Para Revisar" (`REVIEW_REASON_SQL` em `services/common.ts`)

Uma transação não removida (`is_removed = 0`) aparece na fila **"Para revisar"** se tiver um `reviewReason` não nulo, avaliado nesta ordem:
1. `'uncategorized'`: `category_source = 'none'` (está em *Sem categoria*) e `flow IN ('expense', 'income')`. **Importante**: confirmar em lote (`Confirmar todas`) não tira transações sem categoria da fila; elas precisam ganhar uma categoria.
2. Se `manual_overrides.reviewed_at IS NOT NULL`: retorna `NULL` (já foi revisada ou recategorizada pelo usuário).
3. `'ambiguous'`: `review_flag IS NOT NULL` (ex.: compra no débito que o banco chamou de pagamento de cartão).
4. `'new'`: `is_new = 1` (chegou em uma sincronização incremental após a primeira carga inicial do banco).
5. `'bank_suggested'`: `category_source = 'pluggy'` e `flow IN ('expense', 'income')` (está usando a categoria sugerida pelo banco, ainda não confirmada pelo usuário).

---

## 7. Rentabilidade Ponderada pelo Tempo (TWR) de Investimentos (`investments.ts`)

Como o Open Finance não informa rentabilidade histórica, o painel grava retratos diários (`investment_snapshots`) e calcula o retorno ponderado pelo tempo (**Time-Weighted Return — TWR**), descontando aportes e resgates para que colocar dinheiro novo não pareça lucro:
- Em cada dia $t > t_0$ da série de snapshots:
  - $V_{t-1}$ é o patrimônio investido total no dia anterior.
  - $V_t$ é o patrimônio investido total no dia $t$.
  - $F_t$ é o fluxo líquido (`netFlowCents`) do dia $t$: soma de compras/aportes (`BUY` ou fluxos manuais positivos) menos vendas/resgates (`SELL` ou fluxos manuais negativos). **Dividendos e juros (`INTEREST`) têm `flowCents = 0`**, pois são retorno da carteira!
  - O retorno diário da carteira é:
    $$r_t = \frac{V_t - V_{t-1} - F_t}{V_{t-1}}$$
  - O índice acumulado (base 100 em $t_0$) evolui como $I_t = I_{t-1} \times (1 + r_t)$.
- Os índices de **CDI** (série SGS 12, taxa diária em %) e **IPCA** (série SGS 433, taxa mensal distribuída pró-rata pelos dias do mês: $(1 + \text{ipca}/100)^{1/\text{diasNoMes}}$) também começam em `100` na mesma data inicial $t_0$.

---

## 8. As 12 Sugestões Ativas (`domain/insights.ts` e `server/config/insights.json`)

Todas calculadas localmente a cada sincronização. Textos que contêm valores em reais usam marcadores `{{money:18740}}` no banco e saem na API divididos em `titleParts` / `bodyParts` (`Array<{ text: string } | { cents: number }>`) para que o modo privacidade borre os números no frontend:

1. **`spending_outlier` (Gasto fora do padrão no ritmo do mês)**: no mês atual (quando já passou pelo menos `minPace = 20%` dos dias do mês), se uma categoria já gastou `>= R$ 300` (`minAmountCents`), está acima de `130%` (`ratio = 1.3`) da média histórica de 6 meses **proporcional aos dias corridos do mês** (`media6m * (diaAtual / diasNoMes)`) e a diferença passa de `R$ 100`.
2. **`duplicate_charge` (Possível cobrança duplicada)**: duas despesas de consumo (`isConsumptionExpense`: exclui movimentações, investimentos, transferências, impostos, tarifas e compras parceladas) no mesmo estabelecimento e mesmo valor em até `3 dias` (nos últimos `35 dias`). **Filtros anti-falso-positivo**: ignora se houve estorno/crédito do mesmo valor na conta em até 10 dias; ignora se o mesmo valor naquele estabelecimento já ocorreu `3+` vezes no histórico (hábito, como estacionamento ou café fixo); ignora se o estabelecimento tem `15+` compras no histórico (loja muito frequente). No máximo 1 alerta por estabelecimento.
3. **`subscription_change` (Assinatura nova ou mais cara)**: recorrência mensal detectada por `detectRecurring` com `status === 'new'` (2 primeiras cobranças recentes) ou `status === 'increased'` (subiu mais de `5%` em relação ao mês anterior). Linka para `/gastos#recorrencias`.
4. **`idle_cash` (Dinheiro parado em conta)**: saldo em conta corrente manteve-se acima de `1,5x` a média mensal de gastos durante todos os últimos `30 dias` de snapshots.
5. **`issuer_concentration` (Concentração por emissor / FGC)**: mais de `25%` da carteira investida em um único emissor privado (exclui `"Tesouro Nacional"` e `"Sem emissor"`). Se o valor em renda fixa bancária daquele emissor ultrapassar `R$ 250.000` (teto do FGC), a severidade sobe para `'alert'`.
6. **`maturity_soon` (Vencimento próximo)**: renda fixa (exceto Caixinhas do Nubank) vencendo nos próximos `30 dias`.
7. **`bill_above_normal` (Fatura maior que o normal)**: fatura aberta estimada de um cartão (com `billCheckAvailable === true`) acima de `120%` da média das últimas 6 faturas daquele cartão.
8. **`review_pending` (Transações para revisar)**: fila de revisão do mês atual não está vazia (`reviewCount >= 1`). Linka para `/transacoes?review=1`.
9. **`authorization_expiring` (Autorização do Meu Pluggy vencendo)**: faltam `30 dias` ou menos para `consentExpiresAt` (ou já venceu).
10. **`account_stale` (Conexão sem dados novos)**: `sourceUpdatedAt` do item na Pluggy está há `3 dias` ou mais sem atualizar.
11. **`weekly_summary` (Resumo semanal)**: gerado a partir de segunda-feira com o total gasto na semana anterior (segunda a domingo) e a maior categoria da semana.
12. **`manual_stale` (Investimento manual desatualizado)**: posição lançada à mão **por valor** (`kind === 'value'`) cuja `referenceDate` tem mais de `30 dias`.
