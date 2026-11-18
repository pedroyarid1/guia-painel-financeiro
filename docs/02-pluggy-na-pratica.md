# 02. Pluggy e Meu Pluggy na Prática: Guia de Conexão e Armadilhas Reais

Este documento reúne tudo o que foi descoberto ao integrar a API da Pluggy com bancos brasileiros reais (XP, Nubank, Santander, Itaú, Bradesco, BTG, Inter, C6, etc.) usando o caminho gratuito **Meu Pluggy**.

> **Para o Agente de IA**: Não implemente a sincronização baseando-se apenas na documentação genérica da Pluggy. Siga rigorosamente as regras da **Seção 3** abaixo, pois os dados vindos pelo conector proxy `MeuPluggy` têm diferenças importantes em relação aos exemplos da documentação oficial.

---

## 1. Por que usar o caminho "Meu Pluggy"?

A Pluggy oferece dois caminhos para acessar dados bancários reais:
1. **Conectores próprios na conta empresarial**: funciona durante o trial de 15 dias, mas depois exige plano empresarial pago (a partir de R$ 2.500/mês) e processo de due diligence. Não faz sentido para uso pessoal.
2. **Meu Pluggy + conector `MeuPluggy` (oficial para uso pessoal)**:
   - **Gratuito por tempo indeterminado** (o trial não expira para leitura dos seus próprios dados via conector `MeuPluggy`).
   - Até **5 conexões ativas (bancos)** por titular, para uso pessoal e não comercial.
   - Funciona como um **conector proxy**: você conecta seus bancos via Open Finance no app para pessoa física [meu.pluggy.ai](https://meu.pluggy.ai), e depois autoriza a sua aplicação de desenvolvedor no [dashboard.pluggy.ai](https://dashboard.pluggy.ai) a ler uma cópia espelhada desses itens.
   - Não precisa embutir o widget Pluggy Connect no seu app local, nem expor servidor público ou webhooks na internet.

---

## 2. Passo a Passo para o Usuário Conectar seus Bancos (30 minutos)

Esta é a única etapa manual que o dono do painel precisa fazer no navegador:

### Etapa A: Conectar seus bancos no Meu Pluggy
1. Crie uma conta gratuita em **[https://meu.pluggy.ai](https://meu.pluggy.ai)**.
2. Conecte seus bancos e corretoras (até 5 instituições). Cada banco vai pedir autorização via Open Finance (você informa seu CPF, é redirecionado para o app/site do banco, escolhe o prazo — prefira 12 meses/indeterminado — e aprova no celular).
3. *Dica prática*: se o login do `meu.pluggy.ai` ficar preso na tela *"Redirecionando para login seguro..."* no Chrome normal, abra uma **janela anônima**, faça login por lá e conecte.

### Etapa B: Criar sua aplicação no Dashboard e habilitar o proxy
1. Crie uma conta de desenvolvedor em **[https://dashboard.pluggy.ai](https://dashboard.pluggy.ai)** usando **o mesmo e-mail** da sua conta do Meu Pluggy.
2. Mantenha **ambas as abas abertas e logadas no mesmo navegador** (`meu.pluggy.ai` e `dashboard.pluggy.ai`). *Sem a sessão do Meu Pluggy ativa no mesmo navegador, o widget da página Demo falha com "Um erro inesperado ocorreu" e o item dá `USER_INPUT_TIMEOUT`.*
3. No Dashboard, vá em **Applications → Create Application** (ex.: "Painel Financeiro").
4. Copie o **`CLIENT_ID`** e gere/copie o **`CLIENT_SECRET`** (ele só é exibido uma única vez!). Cole ambos no seu arquivo `.env`.
5. No menu lateral do Dashboard, vá em **Customization → Connectors**, mude para seleção personalizada, abra a aba **Direct** e habilite o conector **MeuPluggy** (id `200`).

### Etapa C: Autorizar cada banco na página Demo e copiar os `itemIds`
1. No Dashboard, abra a sua aplicação e clique na aba **Demo** → botão **Connect Account**.
2. No widget que abrir, escolha **MeuPluggy** (⚠️ **NÃO** escolha o conector direto do banco como Nubank ou Itaú; escolha sempre **MeuPluggy**).
3. Clique em Continuar → Verificação de Segurança → **Conectar**. Uma janela do Meu Pluggy vai abrir para você selecionar **um** dos bancos que você conectou na Etapa A e autorizar.
4. **Repita o passo 2 e 3 uma vez para cada banco** (é **um item por banco**). Se algum item falhar durante o clique, apague-o no ícone de lixeira na lista da página Demo antes de tentar de novo (a API bloqueia criar outro item enquanto há um em execução para a mesma credencial).
5. Na seção **Connected Items** da página Demo, copie o **Item ID** de cada banco conectado (ou pegue da URL `?itemId=...` ao clicar nele).
6. Preencha no seu `.env`:
   ```env
   DATA_SOURCE=pluggy
   PLUGGY_CLIENT_ID=seu-client-id
   PLUGGY_CLIENT_SECRET=seu-client-secret
   PLUGGY_ITEM_IDS=uuid-banco-1,uuid-banco-2,uuid-banco-3
   PLUGGY_AUTHORIZED_AT=2026-11-18
   DATA_DIR=local-data/real
   ```
7. **Renovação anual**: a autorização entre o Meu Pluggy e a sua aplicação do Dashboard dura **1 ano** (`consentExpiresAt`). O painel mostra quantos dias faltam e avisa 30 dias antes. Para renovar, basta repetir a Etapa C uma vez por ano.

---

## 3. O que vem nos dados reais (e como o código DEVE tratar)

Ao testar o conector `MeuPluggy` contra dados reais de bancos brasileiros, surgiram 11 comportamentos fundamentais que o motor de sincronização e domínio precisa implementar:

### 3.1. A API v1 de transações foi desligada (`410 Gone`) — Use apenas a v2 com cursor
- `GET /transactions` (v1) responde **410 Gone** (`"This endpoint is deprecated. Use GET /v2/transactions with cursor pagination instead."`).
- No `pluggy-sdk` (Node), use **`client.fetchAllTransactions(accountId, { from })`** (que já usa `/v2/transactions` com paginação por cursor) ou chame `GET /v2/transactions?accountId=...&dateFrom=...` seguindo o cursor `next` até acabar. Não passe `pageSize` na v2.
- `GET /v2/items` (listar itens da aplicação) retorna `403 Forbidden` para contas pessoais. Por isso o servidor lê a lista de IDs diretamente de `PLUGGY_ITEM_IDS` no `.env` e chama `client.fetchItem(itemId)` para cada um.

### 3.2. `connector.name` vem sempre como `"MeuPluggy"` — Infira a instituição real
- Como o conector `MeuPluggy` é um proxy, todo item, conta e investimento chega com `connector.name = "MeuPluggy"`.
- Se você não tratar isso, o usuário verá `"MeuPluggy"` em todas as contas e cartões!
- **Solução (`server/src/domain/institutions.ts`)**:
  1. Se o conector **não** for proxy (`!/\bmeu ?pluggy\b/i.test(connectorName)`), mantenha o nome do conector (isso preserva os bancos fictícios do modo `mock`: Banco Aurora, Banco Horizonte, Vértice Investimentos).
  2. Se for `MeuPluggy`, infira a instituição **por item (conexão)** aplicando uma tabela de padrões RegExp (`INSTITUTION_PATTERNS`) sobre o texto normalizado:
     - **Primeiro**: faça uma votação (`mostVoted`) sobre os nomes das contas (`name` e `marketingName`) daquele item (ex.: `"Nu Pagamentos S.A."` ou `"ultraviolet-black"` → **`Nubank`**; `"Banco Santander Pessoa Física"` ou `"SANTANDER SX VISA      "` → **`Santander`**; `"XP"` → **`XP`**; e padrões para **`Itaú`**, **`Bradesco`**, **`Inter`**, **`BTG`**, **`C6`**, **`Banco do Brasil`**, **`Caixa`**, **`Mercado Pago`**, **`PicPay`**, **`PagBank`**, **`Sicoob`**, **`Sicredi`**, **`Safra`**, **`Neon`**, **`Rico`**, **`Ágora`**, **`Modal`**, **`Original`**, **`Avenue`**).
     - **Segundo** (só se nenhuma conta casar, ex.: corretora com conta genérica): faça votação sobre os emissores (`issuer`) dos investimentos do item (não use emissores antes das contas, porque uma corretora como a XP pode custodiar CDBs emitidos por outros bancos).
     - **Terceiro** (fallback): o primeiro nome de conta limpo.
  3. Aplique a instituição inferida ao item e a **todas** as contas e investimentos daquele item.
  4. **Nomes amigáveis de conta (`friendlyAccountName`)**:
     - Limpe espaços extras (`"SANTANDER SX VISA              "` → `"Santander SX Visa"`).
     - Para instituições reconhecidas na tabela:
       - Conta corrente/poupança cujo nome é genérico ou só repete o banco vira `"Nubank · Conta"`, `"Santander · Poupança"`, `"XP · Conta"`.
       - Slugs conhecidos de cartão são traduzidos (`/\bultraviolet/` → `"Cartão Ultravioleta"`, resultando em `"Nubank · Cartão Ultravioleta"`).
       - Nomes em CAIXA ALTA ou `kebab-case` viram Title Case preservando siglas (`XP`, `BTG`, `C6`, `BB`, `PJ`, `MEI`, `CDB`, `SX`).

### 3.3. Bancos mandam contas "fantasmas" ou duplicadas — Oculte automaticamente sem apagar
- No Open Finance real, os bancos enviam contas que o usuário nunca usa: uma conta-corrente gêmea com o mesmo número e saldo zero (XP), uma conta poupança zerada sem nenhum lançamento (Santander), ou um cartão antigo cancelado/sem uso com limite zero e faturas zeradas.
- **Solução (`server/src/domain/visibility.ts`)**:
  - Nunca apague a conta do banco (para manter integridade referencial).
  - Marque `auto_hidden = 1` em qualquer conta que:
    1. Tenha `balance_cents == 0`, **zero** transações nos últimos 12 meses (`recentTxCount == 0`) e **zero** faturas com valor diferente de zero (`nonZeroBills == 0`); **OU**
    2. Seja a **gêmea sem transações** (`totalTxCount == 0` e `nonZeroBills == 0`) de outra conta ativa que tem o **mesmo número** dentro do mesmo `itemId`.
  - O usuário pode forçar mostrar ou ocultar qualquer conta na tela Conexões e ajustes (`PATCH /api/accounts/:id/visibility`, gravando `hidden_override = 0 | 1 | NULL`). A visibilidade efetiva é `COALESCE(hidden_override, auto_hidden) == 0`.
  - Contas ocultas saem de listas, filtros, cartões, patrimônio e série histórica do patrimônio (mas se tiverem alguma transação antiga, as transações continuam contando nos gastos históricos).

### 3.4. `providerId` vem `null` na prática — Chave local estável com fallback e sufixo de ordem
- Embora a doc do Open Finance diga que `providerId` é o identificador estável do banco, pelo MeuPluggy ele vem `null` em quase todas as transações reais. E o `id` da Pluggy muda quando o banco atualiza a descrição ou o status da transação!
- **Solução (`server/src/domain/identity.ts`)**:
  - Se `providerId` existir: `p_` + SHA-1(`accountId | providerId`).
  - Se `providerId` for nulo: `f_` + SHA-1(`accountId | date (Brasília) | spendCents | normalizeText(description)`), truncado em 24 hex chars.
  - **Transações idênticas no mesmo dia**: duas compras reais de mesmo valor no mesmo estabelecimento e mesmo dia (ou duas aplicações iguais) terão a mesma chave base. Ordene o lote por data/descrição/ordem e atribua sufixos determinísticos: a 1ª fica `<base>`, a 2ª fica `<base>_2`, a 3ª `<base>_3`.
  - Todas as correções manuais (`manual_overrides`) usam essa `local_key`, jamais o `pluggy_id`.

### 3.5. `creditData.balanceCloseDate` vem `null` em todos os cartões reais — Estime o ciclo pelas faturas
- Na API real do Open Finance, nenhum cartão envia `creditData.balanceCloseDate` (vem sempre `null`; só `balanceDueDate` vem). Além disso, a API `/bills` só retorna faturas já fechadas (a fatura aberta nunca vem).
- **Solução (`server/src/domain/cards.ts` — `cycleFromBills` e `statementMonthFor`)**:
  1. **Mês da fatura de cada compra (`statementMonth`)**, nesta ordem de prioridade:
     - Se a transação tem `billId` e essa fatura existe em `bills`: use o mês do `dueDate` dessa fatura (`bill.dueDate.slice(0, 7)`).
     - Senão, se a transação traz `creditCardMetadata.billForecastDate` no formato `AAAA-MM` (o Nubank e o Santander enviam isso, inclusive para parcelas futuras!): use esse mês.
     - Senão, calcule pelo ciclo de faturas do cartão (`cycleFor` se o cartão tiver `closeDate` e `dueDate`, ou `cycleFromBills` usando o histórico de `bills`).
     - Último recurso: o mês da própria data da transação (`date.slice(0, 7)`).
  2. **Como projetar o ciclo pelo histórico de faturas (`cycleFromBills`)**:
     - Se existir uma fatura conhecida em `bills` cuja `closingDate` seja **maior** que a data de referência (`closingDate > refDate`), a compra pertence a essa fatura!
     - Caso contrário, olhe para as últimas 6 faturas com `closingDate` e `dueDate`: descubra o **dia de vencimento mais frequente** (`dueDay`, ex.: dia 9 ou 10) e a **mediana de dias entre o fechamento e o vencimento** (`gap`, ex.: 7 dias). Projete o próximo vencimento e subtraia `gap` dias para achar o próximo fechamento. (Compras feitas no próprio dia do fechamento já caem na fatura do mês seguinte.)
     - Marque `datesEstimated = true` no DTO do cartão para a interface informar que a data de fechamento é estimada.

### 3.6. Totais de faturas do banco vs. Soma das compras (`billCheckAvailable`)
- Em dados reais do Open Finance, o `totalAmount` enviado pelo banco no endpoint `/bills` muitas vezes diverge da soma das transações daquele ciclo (no Nubank o total da fatura fechada às vezes vem menor que as compras; no Santander às vezes vem ~1,5x maior).
- **Solução**:
  - O histórico de faturas na tela de Cartões deve usar **`purchasesCents`** (a soma das compras, encargos e estornos do cartão naquele `statementMonth`, sem pagamentos de fatura, duplicadas ou removidas — exatamente a mesma fórmula da fatura aberta estimada).
  - O `totalCents` vindo do banco aparece apenas ao abrir o detalhe da fatura, rotulado como *"Total informado pelo banco"*.
  - Se um cartão tiver divergência (`> 2%` e `> R$ 20`) na maioria de suas faturas (3+ faturas e `>= 50%` das faturas com valor), defina `billCheckAvailable = false` para esse cartão: a tela mostra um único aviso neutro explicando a limitação do banco no Open Finance e desativa o alerta "Fatura maior que o normal" para ele.

### 3.7. Dívida nos cartões e Parcelas futuras
- No Open Finance, o `balance` da conta de tipo `CREDIT` (ou `creditLimit - availableCreditLimit`) representa o **limite total consumido**, que já inclui todas as parcelas futuras.
- Portanto, no cálculo do **Patrimônio Líquido Hoje**:
  $$\text{Patrimônio} = \text{Saldos em Conta Corrente} + \text{Investimentos Ativos} - \text{Dívida Total nos Cartões (Limite Usado)}$$
- A "Próxima fatura" é mostrada separadamente para planejamento de caixa, mas o patrimônio desconta a dívida inteira do cartão.
- **Parcelas futuras do Nubank vs. Outros bancos**:
  - O Nubank envia parcelas futuras como transações reais com `date` nos meses futuros e `billForecastDate` preenchido.
  - Outros bancos enviam apenas a parcela do mês atual (ex.: `installmentNumber: 2, totalInstallments: 10`). Para a tela de Compromisso Futuro (12 meses), projete as parcelas `3..10` nos meses seguintes, cuidando para não duplicar parcelas futuras que o banco (como o Nubank) já tenha enviado como transação!
  - Alguns bancos (como a XP) enviam a compra parcelada pelo **valor cheio sem `billId`** no dia da compra, além de enviar cada parcela mensal. Detecte e marque a compra cheia como `is_duplicate = 1` (`findFullPurchaseDuplicates`: mesma descrição base, data próxima à compra da 1ª parcela e valor igual à soma das parcelas).

### 3.8. Caixinhas do Nubank e Posições Resgatadas (`TOTAL_WITHDRAWAL`)
- **Posições resgatadas**: a Pluggy mantém investimentos antigos já sacados com `status: "TOTAL_WITHDRAWAL"` e `balance: 0`. Filtre `COALESCE(status, '') != 'TOTAL_WITHDRAWAL'` em todas as listas de posições, totais e alocações (mantendo apenas a rota de movimentações acessível).
- **Caixinhas do Nubank (150+ CDBs separados)**: cada depósito numa Caixinha do Nubank vem no Open Finance como um `FIXED_INCOME / CDB` separado emitido pela `NU FINANCEIRA`, cada um com uma data de vencimento diferente!
  - **Agrupamento (`groupPositions`)**: agrupe posições que tenham o mesmo `(type, subtype, name normalizado, issuer, institution, rate, rateType, fixedAnnualRate)` **SEM incluir a data de vencimento (`dueDate`) na chave de agrupamento**. Assim, 150 depósitos viram 1 única linha na tabela com `count: 150` e a lista individual dentro de `members`.
  - **Tratamento especial de Caixinhas (`server/src/domain/investment-names.ts`)**: renda fixa emitida pela `NU FINANCEIRA` (ou cujo nome contenha `caixinha`) recebe o nome limpo `"Caixinhas do Nubank"`, tem `dueDate = null` e **não entra** na linha do tempo de vencimentos (`maturities`) nem gera alerta de "vencimento próximo", pois funciona como reserva de liquidez diária.

### 3.9. Categorias da Pluggy vêm preenchidas, mas erram em marcas conhecidas e JCP
- Ao contrário do que a página comercial do Meu Pluggy sugere, as transações **vêm sim com `category` e `categoryId`** da Pluggy (em ~99,9% dos casos).
- Porém, o motor dos bancos comete 3 erros clássicos que o painel corrige:
  1. **Juros sobre Capital Próprio (JCP) e Rendimentos**: a Pluggy às vezes classifica crédito de JCP na conta como `"Interests charged"` (`02020000`, categoria de empréstimos!) ou como `"Investments"`. O painel detecta créditos de dividendos/JCP/rendimentos pelo texto e `operationType` (`isInvestmentIncome`) e os força para `"Rendimentos de investimentos"` (`01060000`), contando como **Receita**.
  2. **Compra no débito via carteira digital classificada como "Pagamento de cartão"**: compras pagas no débito (ex.: *"Compra no débito via NuPay iFood"*) às vezes vêm com `pluggyCategoryId = "05100000"` (*Credit card payment*). Se o painel acreditasse nisso, a compra viraria "Movimentação" e sumiria dos gastos! Regra: texto com `"compra no debito"` **nunca** é pagamento de fatura; a categoria `"05100000"` do banco é ignorada, a transação conta como gasto e ganha `review_flag = 'ambiguous_bill_payment'` para o usuário confirmar na fila de revisão.
  3. **Marcas de transporte por app e streaming**: bancos frequentemente classificam Uber/99 como "Transporte público" e Netflix/Spotify/Apple como "Tarifas bancárias". Por isso, uma lista curta de `BRAND_RULES` (com word boundaries `\b`) roda **antes** da categoria sugerida pelo banco (mas sempre **depois** da correção manual e das regras criadas pelo usuário).

### 3.10. Leitura vazia (`empty_read`) quando a autorização de 1 ano expira
- Quando a autorização anual do Meu Pluggy vence, os endpoints de contas, transações e investimentos passam a responder listas vazias `[]` com status HTTP `200`, sem erro imediato!
- **Regra de segurança máxima na sincronização (`storeTransactions` e `syncItem`)**:
  - Se uma leitura de contas, transações ou investimentos voltar vazia (`fetched.length === 0`), mas o banco SQLite local já tiver registros ativos naquela janela para aquela conta/item, **NUNCA marque os registros locais como removidos**.
  - Mantenha todos os dados locais intactos, grave `alert_kind = 'auth_expired'` (se `consentExpiresAt < today`) ou `'empty_read'`, e exiba o alerta claro na interface.

### 3.11. Investimentos no Exterior (fora do Open Finance)
- O Open Finance brasileiro só traz ativos custodiados no Brasil (B3, Tesouro, CDBs, fundos locais). Contas internacionais (Avenue, Nomad, XP Internacional, Monte Bravo Internacional, Interactive Brokers) não vêm pela Pluggy.
- Para cobrir 100% do patrimônio sem esforço manual diário, o painel implementa **Investimentos lançados à mão** em duas modalidades:
  1. **Por ativo (`kind = 'asset'`)**: o usuário informa apenas o `ticker` da bolsa americana (ex.: `VOO`, `VT`, `AAPL`, `BRK.B`), a quantidade de cotas (`quantity`) e a instituição. O servidor busca o histórico e o preço atual no endpoint de gráfico do **Yahoo Finance** (`https://query1.finance.yahoo.com/v8/finance/chart/<TICKER>?range=1y&interval=1d`, que não exige chave de API) e converte para reais usando a cotação **PTAX venda do Banco Central** (série SGS `1` para USD, `21619` para EUR). O preço atualiza sozinho!
  2. **Por valor (`kind = 'value'`)**: para caixa em dólar/euro ou fundos sem ticker público. O usuário informa o valor na moeda e a data; após 30 dias sem atualização, o painel gera a sugestão `"Atualize o valor de <nome>"`.
