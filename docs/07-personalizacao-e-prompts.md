# 07. Guia de Personalização e Prompts Prontos para o seu Agente de IA

O objetivo deste projeto não é apenas te dar um app igual ao de todo mundo, mas sim uma **base sólida que você pode moldar continuamente com a sua IA** conforme a sua vida financeira muda.

Abaixo estão **prompts prontos para copiar e colar** no seu agente de IA (Claude Code, Cursor, Codex, Antigravity, Windsurf) e exemplos de como personalizar tudo sem quebrar os cálculos contábeis.

---

## 1. Prompts para Construir a Aplicação do Zero

### Prompt 1 — Entrevista rápida + Início da Fase 1 (Comece por aqui!)
> Copie e cole no seu agente de IA ao abrir este repositório pela primeira vez:

```text
Leia o arquivo AGENTS.md e todos os documentos da pasta docs/ (01 a 07).
Quero construir o meu painel financeiro pessoal seguindo o roteiro de 6 fases em docs/06-roteiro-de-implementacao-ia.md.

Antes de escrever o código da Fase 1, me faça 3 perguntas curtas para registrar na seção "Decisões" do AGENTS.md:
1. Qual cor-semente (SEED_COLOR em hex) eu quero para o tema do meu painel (ou se mantenho o índigo #4F46E5 padrão);
2. Se eu já quero criar alguma categoria personalizada inicial em local-data/custom-categories.json além das 22 categorias padrão;
3. Quais bancos/corretoras eu pretendo conectar depois (para garantir que estão na tabela INSTITUTION_PATTERNS).

Depois que eu responder, implemente a Fase 1 completa (Fundação, SQLite, Domínio, Fontes mock/Pluggy e Sincronização), rode `npm run typecheck` e `npm test` até todos os testes passarem, atualize o AGENTS.md e faça o commit da Fase 1.
```

---

### Prompt 2 — Fase 2 (Serviços, Investimentos Manuais e API REST)

```text
Seguindo o AGENTS.md e docs/06-roteiro-de-implementacao-ia.md (e o contrato em docs/04-contrato-da-api.md), implemente a Fase 2 completa:
- Todos os serviços em server/src/services/ (overview, spending com basis=bill|purchase, transactions, cards, investments, manual-investments, insights, connections);
- Todas as rotas Hono em server/src/api/app.ts e o entrypoint server/src/index.ts;
- A suíte de testes de integração da API (mock-api, integration-api, real-shape, manual-investments e ux-review, provando que lista === donut === barra mensal === visão geral === fluxo de caixa nos dois critérios de mês).

Rode `npm run typecheck` e `npm test`, atualize o AGENTS.md e faça o commit da Fase 2.
```

---

### Prompt 3 — Fase 3 (Design System Material 3 Expressive e Galeria `/design`)

```text
Seguindo docs/05-design-system-e-ux.md e a Fase 3 de docs/06-roteiro-de-implementacao-ia.md, implemente o Design System Material 3 Expressive completo em web/src/design/:
- Gerador de tokens (npm run tokens -> tokens.css), base.css, color.ts (com categoryTone garantindo contraste WCAG >= 3:1 e >= 4.5:1), motion.ts (molas spatial/effects, springEasing e vigia de requestAnimationFrame data-frames="paused"), sharedTransition.ts e format.ts;
- Providers (ThemeProvider, PrivacyProvider com Alt+H e useMoneyText, SnackbarProvider com showUndo);
- Os 25 componentes base, os componentes financeiros, os 7 gráficos Recharts + ChartReadout, a casca adaptativa Layout.tsx e a galeria viva em /design.

Implemente os testes de design (contraste WCAG de todas as cores nos dois temas, tokens definidos, formatação e gestos), rode `npm run typecheck` e `npm test`, atualize o AGENTS.md e faça o commit da Fase 3.
```

---

### Prompt 4 — Fase 4 e 5 (As 7 Telas + Polimento e Verificação)

```text
Seguindo docs/01-arquitetura-e-especificacao.md, docs/04-contrato-da-api.md, docs/05-design-system-e-ux.md e as Fases 4 e 5 de docs/06-roteiro-de-implementacao-ia.md:
1. Implemente o cliente da API (web/src/api/) e o estado global (web/src/state/);
2. Implemente as 7 telas completas em web/src/pages/ (Visão geral, Gastos com drill-down e edição inline, Transações com fila "Para revisar" e busca por valor, Cartões com histórico pela soma das compras e parcelas futuras, Investimentos com posições agrupadas e lançamentos manuais, Sugestões com RichText e Conexões e ajustes);
3. Se você tiver suporte a subagentes, pode dividir a construção das telas em paralelo e depois integrar e revisar.
4. Rode `npm run typecheck`, `npm test` e `npm run build`, verifique todas as telas em modo fictício e faça o commit.
```

---

## 2. Como Personalizar o Painel para a Sua Vida

Depois que o painel estiver rodando, tudo o que é pessoal seu fica separado do código-fonte (ou pode ser ajustado pedindo para a IA).

### 2.1. Criar suas próprias Categorias (`local-data/custom-categories.json`)
Você não precisa ter as categorias pessoais de outra pessoa e pode criar as que fazem sentido para a sua rotina (ex.: *Surf*, *Terapia*, *Filhos*, *Reforma do Apê*, *Side Projects*, *Cerveja Artesanal*).

Basta criar ou editar o arquivo `local-data/custom-categories.json` (ele fica fora do git e entra automaticamente no `npm run backup`):

```json
[
  {
    "id": "90000000",
    "name": "Reforma do Apê",
    "color": "#558B2F",
    "icon": "construction",
    "children": [
      { "id": "90010000", "name": "Marcenaria" },
      { "id": "90020000", "name": "Material de construção" }
    ]
  },
  {
    "id": "91000000",
    "name": "Corrida e Esportes",
    "color": "#00897B",
    "icon": "directions_run",
    "children": []
  }
]
```
- **IDs recomendados**: use `80000000` até `98000000` para categorias-mãe (`NN000000`) e `NN010000`, `NN020000` para as subcategorias.
- **Ícones**: qualquer nome de ícone do [Material Symbols Rounded](https://fonts.google.com/icons) (ex.: `eco`, `pets`, `child_care`, `sports_esports`, `construction`, `local_cafe`, `self_improvement`).
- Ao reiniciar o servidor, as novas categorias já aparecem no gráfico de rosca, nos filtros e nos chips de recategorização com contraste de cor ajustado automaticamente para claro e escuro!

### 2.2. Ensinar seus estabelecimentos e Pix recorrentes (Regras de Categoria)
No Brasil, muita despesa fixa é paga por Pix para pessoa física ou jurídica (aluguel direto com proprietário, diarista, personal trainer, terapeuta, feira).
- **Pela interface**: abra a transação em **Transações** (ou direto no drill-down de **Gastos**), ligue o switch **"Aplicar a todas de `<Nome>`"** e toque na categoria. O painel cria a regra (que olha tanto a descrição quanto o nome do recebedor do Pix/boleto) e recategoriza todo o seu histórico na hora.
- **Com ajuda da IA**: depois da sua primeira sincronização real, você pode pedir para a sua IA:
  > *"Olhe as transações que estão na fila 'Para revisar' deste mês pela API local (`http://127.0.0.1:5174/api/transactions?review=1`) e me liste os estabelecimentos/Pix que estão sem categoria para eu te dizer em qual categoria cada um entra e você criar as regras via `POST /api/rules`."*

### 2.3. Ajustar a sensibilidade das Sugestões (`server/config/insights.json`)
Acha que o alerta de "Dinheiro parado em conta" com `1.5x` o gasto mensal é agressivo demais e prefere `3x`? Ou quer ser avisado de vencimentos de renda fixa com `60 dias` de antecedência em vez de `30`?
Edite `server/config/insights.json`:
```json
{
  "spendingOutlier": { "ratio": 1.3, "minDifferenceCents": 10000, "minAmountCents": 30000, "minPace": 0.2, "months": 6 },
  "duplicateCharge": { "maxDays": 3, "lookbackDays": 35 },
  "subscriptionChange": { "increasePct": 5 },
  "idleCash": { "multiple": 1.5, "days": 30, "months": 6 },
  "issuerConcentration": { "maxShare": 0.25, "fgcLimitCents": 25000000 },
  "maturitySoon": { "days": 30 },
  "billAboveNormal": { "ratio": 1.2, "months": 6 },
  "reviewPending": { "minCount": 1 },
  "authorizationExpiring": { "warnDays": 30 },
  "accountStale": { "days": 3 },
  "weeklySummary": { "weekday": 1 },
  "manualStale": { "days": 30 }
}
```

### 2.4. Trocar a Cor do Tema (`SEED_COLOR`)
Quer o painel com cara de verde-floresta, azul-oceano, terracota ou roxo?
1. Edite `SEED_COLOR` em `web/src/design/color.ts`;
2. Rode `npm run tokens`;
3. Toda a paleta clara e escura do Material 3 Expressive se regenera mantendo o contraste acessível.

---

## 3. Ideias de Evoluções Futuras para Pedir à sua IA

Quando o seu painel v1 estiver rodando redondo, você pode pedir para a sua IA criar novas funcionalidades sob medida para você. Exemplos de prompts:

- **Metas e Orçamento por Categoria**:
  > *"Quero definir um teto mensal opcional para algumas categorias (ex.: R$ 1.500 em Alimentação, R$ 800 em Lazer). Crie uma tabela `category_budgets`, uma rota na API e mostre uma barra de progresso discreta dentro de cada linha da lista de Gastos quando a categoria tiver orçamento definido."*
- **Divisão de Despesas da Casa (Casal / República)**:
  > *"Moro com minha companheira e dividimos algumas contas da casa. Adicione uma tag opcional 'Compartilhado (50%)' no detalhe da transação e um resumo mensal de acerto de contas."*
- **Importador de CSV / OFX antigo**:
  > *"O Open Finance puxou os últimos 12 meses, mas eu tenho os arquivos OFX/CSV do ano retrasado. Crie um script em `scripts/import-ofx.ts` que importe essas transações antigas usando a mesma função `assignLocalKeys` e `deriveTransactions`."*
- **Relatório Anual para Imposto de Renda**:
  > *"Crie uma aba ou exportação para Imposto de Renda que liste os saldos das contas e posições de investimentos em 31/12 do ano-base (usando os snapshots) e some todos os gastos do ano nas categorias Saúde e Educação."*
