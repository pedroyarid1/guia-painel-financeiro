# 05. Design System Material 3 Expressive, Movimento e UX

O painel utiliza um **Design System Material 3 Expressive próprio** (`web/src/design/`), construído sem bibliotecas de componentes prontas (porque nenhuma biblioteca web entrega o M3 Expressive com morph de formas, física de molas e integração nativa com modo privacidade).

---

## 1. As 8 Regras de Ouro do Frontend

1. **Zero valores mágicos de estilo**: nunca escreva cor hexadecimal, raio de borda, sombra ou duração de transição soltos nas telas. Use exclusivamente os tokens CSS (`var(--md-sys-color-*)`, `var(--md-shape-*)`, `var(--md-elevation-*)`, `var(--md-transition-*)`, `var(--space-*)`).
2. **Dinheiro sempre por `<Money>`**: recebe `cents` (inteiro da API), formata em `pt-BR` (`R$ 1.234,56`, `US$ 1.234,56` ou `€ 1.234,56`), anima contagens apenas nos números-herói (`countOnMount`) e respeita o modo privacidade (`Alt+H`). Onde um componente React não cabe (`aria-label`, textos de SVG/eixos), use `useMoneyText()`. Textos da API que contêm valores monetários (como sugestões) vêm em partes (`titleParts`, `bodyParts`) e são renderizados por `<RichText>`.
3. **Toda chamada de API passa por `<AsyncContent>`**: trata estados de carregamento (com `<Skeleton>` do mesmo formato/altura do conteúdo final, nunca spinner solto no meio da página), vazio (`<EmptyState>`) e erro (`<ErrorState>` com botão "Tentar de novo"). Quando está recarregando com dados já na tela, mantém os dados visíveis em crossfade (`Presence mode="popLayout"`).
4. **Cor nunca transmite informação sozinha**: ganho/perda usa `<Delta>` (seta + sinal `+`/`−` + cor); categoria usa `<CategoryIcon>` (ícone Material Symbols + cor derivada com contraste garantido) + nome.
5. **Conteúdo nunca depende do fim de uma animação**: janelas encobertas ou abas em segundo plano param o `requestAnimationFrame`. O vigia de quadros em `web/src/design/motion.ts` detecta quando os quadros param e aplica `<html data-frames="paused">`, forçando todo elemento com `[data-settle]` a aparecer imediatamente no estado final (`opacity: 1 !important; transform: none !important`). Use `<Presence>` em vez de `AnimatePresence` puro e nunca use `mode="wait"` para trocar conteúdo de tela.
6. **Privacidade sem borrão pesado na lista**: no modo privacidade (`Alt+H`), o componente `<Money>` troca o número por `"R$ •••••"` com um fade rápido de 120 ms (`privacy-swap`), evitando `filter: blur()` em centenas de linhas. O blur CSS (`privacy-value`) fica restrito aos rótulos internos de SVG onde o texto não pode mudar de largura.
7. **Links entre telas exclusivamente por `links.*`** (`shared/links.ts` e `web/src/links.ts`). Para ler parâmetros da URL na tela, use `useParam('categoryId')`.
8. **Galeria viva em `/design`**: toda peça do design system deve estar demonstrada em `http://127.0.0.1:5173/design` (com modo claro/escuro lado a lado e filtro `?secao=<id>`).

---

## 2. Cores Dinâmicas e Tokens (`color.ts` e `tokens.css`)

### 2.1. Geração a partir de `SEED_COLOR`
- Em `web/src/design/color.ts`, defina a cor-semente (padrão `#4F46E5`, índigo vibrante; o usuário pode trocar pela cor que quiser e rodar `npm run tokens`).
- Use `@material/material-color-utilities` (`Hct`, `TonalPalette`) para gerar os papéis Material 3 para `light` (`<html data-theme="light">`) e `dark` (`<html data-theme="dark">`):
  - Acentos (`primary`, `secondary`, `tertiary`) com croma vibrante;
  - Superfícies neutras (`surface`, `surface-container-lowest`, `surface-container-low`, `surface-container`, `surface-container-high`, `surface-container-highest`) com croma baixo para não brigarem com as 22 cores de categorias;
  - Papéis semânticos financeiros: `positive` (verde para receitas/ganhos), `warning` (âmbar para alertas) e `error` (vermelho para perdas/erros), cada um com seus pares `on-*`, `*-container` e `on-*-container`.

### 2.2. Tons de categoria com contraste garantido (`categoryTone`)
Nunca use o hexadecimal cru da categoria (`#F4511E`, etc.) diretamente no CSS! A função `categoryTone(hex, theme)` converte o hex para HCT e ajusta o Tone (`T`) conforme o tema (`light` ou `dark`) para devolver `{ color, container, onContainer }` garantindo:
- Contraste de `color` (usado em gráficos e barras) **`>= 3:1`** contra todas as superfícies (`surface` até `surface-container-highest`);
- Contraste de `onContainer` sobre `container` (usado em `<CategoryIcon>` e `<CategoryChip>`) **`>= 4.5:1`**.
- **Teste automatizado obrigatório (`design.test.ts`)**: verifica matematicamente a razão de contraste WCAG de todas as cores de categoria contra todas as superfícies nos temas claro e escuro.

### 2.3. Paleta para séries sem categoria (`seriesColor`)
Para gráficos cujas séries não são categorias de gasto (ex.: classes de investimento, instituições, emissores, cartões, CDI vs. IPCA), use uma paleta de 8 cores validada para daltonismo (Okabe-Ito adaptada ao tema claro/escuro). A 9ª fatia em diante é agrupada em `"Outros"`.

---

## 3. Sistema de Movimento e Transições (`motion.ts` e `sharedTransition.ts`)

### 3.1. Molas Expressive: `spatial` vs. `effects`
Inspiradas nos parâmetros oficiais do Material 3 Expressive (AndroidX):
- **`spatial` (com leve quique/overshoot)**: usada apenas quando algo **muda de posição, tamanho ou forma** (morph de canto de botão/chip ao clicar, entrada de card, indicador de aba deslizando, gráfico desenhando pela primeira vez).
  - `spatialFast`: `{ type: 'spring', dampingRatio: 0.6, stiffness: 800 }`
  - `spatialDefault`: `{ type: 'spring', dampingRatio: 0.6, stiffness: 380 }`
  - `spatialSlow`: `{ type: 'spring', dampingRatio: 0.6, stiffness: 200 }`
- **`effects` (criticamente amortecida, ZERO quique, `dampingRatio: 1`)**: usada para **opacidade, cor, fade de troca de tela, crossfade de skeleton → conteúdo e saída de itens**. Nunca faça opacidade ou cor "quicarem".
  - `effectsFast`: `{ type: 'spring', dampingRatio: 1, stiffness: 1600 }`
  - `effectsDefault`: `{ type: 'spring', dampingRatio: 1, stiffness: 800 }`
- **`standard` (`dampingRatio: 0.9`)**: para superfícies grandes como `DetailSheet` / `BottomSheet` voltando ao lugar após arrasto.
- **Conversor de mola para Recharts (`springEasing`)**: resolve a equação diferencial da mola subamortecida/amortecida para gerar uma função `easing(t: 0..1) -> number` compatível com o Recharts.

### 3.2. Transição compartilhada Lista → Detalhe (`useSharedTransition`)
Ao clicar numa linha de transação (em Transações ou no drill-down de Gastos), numa posição de investimento ou numa fatura de cartão:
- Usa a **View Transition API** (`document.startViewTransition`) quando disponível e quando `prefers-reduced-motion` não está ativo.
- Marca a linha clicada e o cabeçalho do detalhe com `view-transition-name: shared-detail`.
- Quando `morph` está ativo no `<DetailSheet>`, a folha entra apenas em fade sem deslizar da borda, deixando o `::view-transition-group(shared-detail)` conduzir o movimento fluido da linha até o painel.

### 3.3. Scrubbing nos gráficos (`<ChartReadout>`)
Em vez de depender apenas de tooltips flutuantes que cobrem as barras ou linhas (especialmente ruins no celular):
- `<AreaTrend>` e `<StackedBars>` aceitam `onScrub(point | null)`.
- No topo do card, `<ChartReadout point={scrub} title={...} cents={...} />` atualiza instantaneamente (`<Money instant>`) o valor e a data sob o cursor/dedo, voltando ao total do período quando o cursor sai do gráfico.

---

## 4. Catálogo de Componentes a Implementar

### 4.1. Componentes Base (`web/src/design/components/`)
1. `Text`: variantes tipográficas M3 (`displayLarge` até `labelSmall`), `emphasized`, `color`, `tabular`, `truncate`.
2. `Icon`: Material Symbols Rounded local, com animação suave de `FILL` (`0` → `1`) quando `filled` muda.
3. `Pressable`: base acessível para elementos clicáveis com `md-state-layer`.
4. `Button`: variantes `filled`, `tonal`, `outlined`, `text`, `elevated`, `danger`; tamanhos `xs`, `sm`, `md`, `lg`; morph do raio da borda ao pressionar; suporte a `to` (rota do React Router), `icon`, `loading`.
5. `IconButton`: variantes `standard`, `filled`, `tonal`, `outlined`, com `<Tooltip>` automático a partir da prop obrigatória `label`.
6. `Fab`: botão flutuante com recolhimento animado de texto (`collapsed`).
7. `ButtonGroup` e `SegmentedButtons`: grupos conectados onde o botão selecionado vira pílula e navega por setas do teclado.
8. `Chip`, `ChipSet` e `MenuChip`: chips de filtro (`selected` com ícone de check), chips de ação, chips de input (com botão `X` de remover) e `MenuChip` (chip que abre menu dropdown ancorado para filtros de Conta, Categoria, Período e Tipo).
9. `Breadcrumb`: trilha de navegação usada no drill-down de Gastos.
10. `Card`: variantes `elevated`, `filled`, `outlined`, `tonal`, `primary`, `secondary`, `tertiary`; clicável quando recebe `onClick` ou `to`.
11. `List`, `ListItem`, `Divider`: listas simples ou `segmented` (itens com cantos arredondados separados por gap de 2px, padrão Expressive).
12. `TextField`, `SearchBar`, `Select`: campos com validação, prefixo/sufixo (`R$`, `US$`) e barra de busca em formato pílula (tecla `Esc` limpa).
13. `Menu`: menu dropdown acessível por teclado (setas, Home/End, Enter, Esc), posicionado dentro da viewport.
14. `Dialog`: modal de confirmação com focus trap e fechamento por `Esc`.
15. `DetailSheet` (`SidePanel` no desktop `>= 840px`, `BottomSheet` arrastável no celular `< 840px`): folha inferior com parada em meia altura (`sheetStops`), scrim que acompanha o arrasto e `user-select: none` na alça/cabeçalho.
16. `Tabs`: abas com indicador animado por mola e suporte a `badge`.
17. `Switch`: interruptor com ícones internos de check/close e morph do thumb.
18. `Tooltip`: dica posicionada e limitada (`clamped`) dentro da viewport para nunca cortar na borda direita/esquerda.
19. `Badge`: contador numérico sobre ícones (some quando `count === 0`).
20. `LinearProgress` (com modo ondulado `wavy`), `CircularProgress` e `LoadingIndicator`.
21. `Skeleton` e `SkeletonText`: shimmer composto por `transform: translateX`.
22. `EmptyState` e `ErrorState`.
23. `Presence`: wrapper sobre `AnimatePresence` integrado ao vigia de quadros.
24. `PullToRefresh`: puxar para atualizar no topo da tela em dispositivos touch (usado em Visão geral e Transações).

### 4.2. Componentes de Finanças (`web/src/design/finance/`)
1. `Money`: formatação monetária, contagem animada (`countOnMount`), fade curto em trocas de mês e swap no modo privacidade.
2. `Delta`: pílula ou texto inline de variação percentual (`kind="pct"`) ou em reais (`kind="money"`), com `goodWhen="up" | "down" | "neutral"`.
3. `CategoryIcon`, `CategoryChip`, `CategoryDot`: ícone da categoria em container colorido com contraste validado e chip com efeito "pop" ao selecionar.
4. `MonthPicker`: seletor de mês global na barra superior (setas anterior/próximo com slide direcional e popover com grade de meses do ano).
5. `SyncStatus`: pílula do topo e dos cards de instituição mostrando o estado (`ok`, `stale`, `empty_read`, `auth_expired`, `error`) e pulso de confirmação ao terminar de sincronizar.
6. `RichText`: renderiza `InsightTextPart[]` passando cada `{ cents }` por `<Money>`.
7. `PrivateText`: esconde textos sensíveis não-monetários (como número da conta bancária) quando o modo privacidade está ativo.
8. `InfoHint`: texto curto sublinhado com ícone de ajuda que abre explicação detalhada da regra de cálculo (em tooltip no desktop ou dialog/popover no toque).
9. `StatCard`: card de métrica com `hero`, `delta` e `footer` (`Sparkline`).
10. `SectionHeader`: cabeçalho padronizado de seção (`title`, `subtitle`, `action`).
11. `TransactionRow`: linha completa de transação com ícone de categoria, ponto de revisão (`needsReview`), descrição limpa (`prettyDescription`: remove prefixos de adquirente como `IFD*`, `MP *`, `PG *`, formata `"Pix para/de Nome"` em Title Case e tira marcador `PARC 02/10` que já vai para a tag), metadados, valor colorido e tags (`2/10`, `Pendente`, `Movimentação`, `Duplicada`, `Removida`).

### 4.3. Gráficos (`web/src/design/charts/`)
1. `DonutChart`: rosca com total central, fatia ativa destacada e sincronização bidirecional de hover/clique com a lista ao lado.
2. `StackedBars`: barras verticais empilhadas por categoria, destaque na barra do mês selecionado (`selectedKey`), clique para selecionar mês e `onScrub`.
3. `AreaTrend`: gráfico de área/linhas para evolução de patrimônio e comparação de rentabilidade (base 100, com linha de referência em `100` e séries tracejadas para CDI e IPCA), com `onScrub`.
4. `Sparkline`: minigráfico limpo sem eixos para o rodapé do card de Patrimônio.
5. `TreemapChart`: mapa de árvore proporcional para alocação de investimentos por classe.
6. `HorizontalBars`: barras horizontais de ranking para alocação por instituição e por emissor.
7. `TimelineChart`: linha do tempo de vencimentos de renda fixa com bolhas distribuídas em trilhas (`lanes`) a partir de `today`.
