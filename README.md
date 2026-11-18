# Blueprint: Construa seu próprio Painel Financeiro Pessoal com IA e Pluggy

Este repositório é um **blueprint completo de engenharia** para que você — junto com um agente de IA (Claude Code, Cursor, Codex, Antigravity, Windsurf, etc.) — construa do zero a sua própria aplicação de controle financeiro pessoal, conectada automaticamente aos seus bancos e corretoras via **Open Finance (Meu Pluggy)**, rodando 100% no seu próprio computador.

Nada de código pronto engessado nem de categorias que só fazem sentido para outra pessoa: aqui estão a arquitetura, as regras contábeis testadas com bancos brasileiros reais, os aprendizados de integração com a API da Pluggy, o contrato de dados e o roteiro em **6 fases** para a sua IA programar o aplicativo sob medida para você.

---

## O que você vai ter ao final

1. **Um aplicativo web local (Node + SQLite + React)** que sobe com um único comando e guarda tudo em um arquivo `.db` na sua máquina. Nenhum dado financeiro seu vai para servidores de terceiros além do que já está no Open Finance.
2. **Conexão gratuita com seus bancos e corretoras** pelo caminho oficial de uso pessoal da Pluggy (**Meu Pluggy + conector MeuPluggy**), sem mensalidade e sem precisar de servidor na internet ou webhooks.
3. **Modo de demonstração com dados fictícios (`DATA_SOURCE=mock`)**: funciona offline desde o primeiro minuto para você desenvolver, testar telas e experimentar mudanças sem tocar nos seus dados reais.
4. **7 telas completas com Design System Material 3 Expressive próprio**:
   - **Visão geral**: patrimônio líquido hoje (contas + investimentos − dívida nos cartões), gasto do mês com previsão de fechamento, últimos 6 meses, top categorias, próximas faturas e alertas.
   - **Gastos**: rosca interativa com drill-down (categoria → subcategoria → transações → edição inline), alternador **"Por fatura / Por data da compra"**, evolução de 12 meses, recorrências/assinaturas e fluxo de caixa (Entrou, Saiu, Saldo do mês).
   - **Transações**: busca instantânea (por texto, categoria, conta ou valor em R$), fila **"Para revisar"** do mês com confirmação em lote e recategorização em 1 toque com criação de regras automáticas. Tudo com botão **"Desfazer"** real.
   - **Cartões**: fatura aberta estimada (calculada pelo ciclo de faturas mesmo quando o banco não envia a data de fechamento), limite usado, histórico de faturas pela soma das compras e projeção de parcelas futuras para 12 meses.
   - **Investimentos**: evolução diária do patrimônio investido, rentabilidade ponderada no tempo (TWR) comparada com **CDI e IPCA** (buscados automaticamente do Banco Central), alocação por classe/instituição/emissor, agrupamento inteligente de caixinhas/CDBs repetidos, linha do tempo de vencimentos e suporte a **investimentos lançados à mão** (como ações/ETFs em dólar no exterior, com cotação automática via Yahoo Finance e PTAX do Banco Central).
   - **Sugestões**: 12 tipos de alertas calculados localmente (gasto fora do padrão no ritmo do mês, possível cobrança duplicada, assinatura mais cara, concentração por emissor/FGC, vencimento próximo, etc.).
   - **Conexões e ajustes**: saúde de cada conexão, validade da autorização anual, ocultação automática e manual de contas sem movimento, gerenciamento de regras de categoria, tema claro/escuro e modo privacidade (`Alt+H`).
5. **100% moldável ao seu jeito**: suas próprias categorias extras em `local-data/custom-categories.json`, suas próprias regras de estabelecimentos, sua cor-semente para o tema e código limpo documentado para você pedir novas telas à sua IA quando quiser.

---

## Como usar este guia com o seu Agente de IA

Você não precisa escrever código manualmente. O repositório foi desenhado para ser lido diretamente pelo seu agente de IA.

### Passo 1: Prepare o ambiente
1. Instale o **Node.js 24+** (recomendado Node 26; no Windows, Mac ou Linux o `better-sqlite3` já baixa binário pré-compilado sem precisar de compilador C++).
2. Clone ou baixe este repositório em uma pasta onde você quer que o seu painel viva (ou crie uma pasta nova para o seu projeto e copie a pasta `docs/` e o arquivo `.env.example` para lá).
3. Abra a pasta no seu agente de IA com acesso ao terminal e aos arquivos (Claude Code, Cursor, Codex, Antigravity, etc.).

### Passo 2: Alinhe suas preferências pessoais com a IA
Antes de mandar a IA construir tudo, leia [docs/07-personalizacao-e-prompts.md](docs/07-personalizacao-e-prompts.md). Lá você encontra **prompts prontos para copiar e colar** em cada etapa e uma lista rápida de escolhas pessoais que você pode definir desde já (ou deixar para depois):
- Quais bancos e corretoras você usa;
- Se você quer alguma categoria própria além das 22 categorias padrão;
- Qual cor-semente (`SEED_COLOR`) você prefere para o tema do app;
- Se você tem investimentos fora do Open Finance (como corretora no exterior).

### Passo 3: Execute o plano em 6 fases
Peça para o seu agente de IA ler `AGENTS.md` e `docs/06-roteiro-de-implementacao-ia.md`, e avançar **uma fase por vez**, rodando os testes automáticos ao final de cada fase:

| Fase | O que a IA constrói | Como validar |
|---|---|---|
| **Fase 1: Fundação e Domínio** | Estrutura do projeto, banco SQLite com migrações, gerador de dados fictícios (`mock`), cliente Pluggy (`v2`), motor de sincronização, regras contábeis puras e índices do Banco Central / Yahoo Finance. | `npm test` (dezenas de testes unitários de regras contábeis passando). |
| **Fase 2: API REST e Serviços** | Servidor Hono com todas as rotas JSON das 7 telas, ações reversíveis ("Desfazer"), investimentos manuais e testes de integração fim a fim sobre o banco em memória. | `npm test` (testes de API garantindo que lista = rosca = barra = visão geral). |
| **Fase 3: Design System M3 Expressive** | Gerador de tokens CSS a partir da cor-semente, molas físicas (`spatial` e `effects`), vigia de quadros de animação, 25 componentes base, 8 componentes financeiros, 7 gráficos Recharts, casca adaptativa e galeria `/design`. | Abrir `http://127.0.0.1:5173/design` no navegador e rodar `npm test`. |
| **Fase 4: As 7 Telas** | Implementação das 7 páginas consumindo a API e o Design System, com drill-down, filtros na URL, folhas de detalhe com transição compartilhada e modo privacidade (`Alt+H`). | Navegar pelas 7 telas com dados fictícios (`npm run dev`). |
| **Fase 5: Polimento e UX** | Checklist de fluidez, responsividade mobile (375 px) e desktop (1280 px), claro/escuro, scrub nos gráficos e verificação de ponta a ponta. | `npm run typecheck`, `npm test` e `npm run build`. |
| **Fase 6: Conexão Real e Customização** | Você conecta seus bancos no Meu Pluggy (30 min, grátis), preenche o `.env`, roda a primeira sincronização real e ajusta suas categorias e regras com a IA. | Seus dados reais no ar em `http://127.0.0.1:5174`. |

---

## Mapa dos documentos

Todos os detalhes técnicos que a sua IA precisa estão na pasta `docs/`:

1. **[docs/01-arquitetura-e-especificacao.md](docs/01-arquitetura-e-especificacao.md)** — Visão do produto, stack tecnológica, estrutura de pastas, schema completo do SQLite (8 migrações) e requisitos detalhados das 7 telas.
2. **[docs/02-pluggy-na-pratica.md](docs/02-pluggy-na-pratica.md)** — Guia passo a passo para conectar seus bancos de graça via **Meu Pluggy**, e todas as **peculiaridades reais do Open Finance no Brasil** que o código precisa tratar (como inferir a instituição por trás do proxy, lidar com `providerId` nulo, estimar fechamento de cartão sem `balanceCloseDate`, agrupar dezenas de CDBs de caixinhas e evitar a rota v1 depreciada).
3. **[docs/03-regras-de-negocio-e-calculos.md](docs/03-regras-de-negocio-e-calculos.md)** — A matemática e as regras contábeis do painel: normalização de sinais, chave local estável, árvore de 22 categorias + categorias personalizadas (`custom-categories.json`), categorização em camadas, detecção de movimentações (pagamento de fatura, transferências próprias, compra no débito via carteira digital), definição única de gasto e receita, fila "Para revisar", TWR de investimentos e os 12 algoritmos de sugestões.
4. **[docs/04-contrato-da-api.md](docs/04-contrato-da-api.md)** — Todas as rotas REST (`/api/*`), parâmetros de query, formatos de resposta, links compartilhados entre telas (`shared/links.ts`) e tipos TypeScript (`shared/types.ts`).
5. **[docs/05-design-system-e-ux.md](docs/05-design-system-e-ux.md)** — Especificação do Design System Material 3 Expressive, geração dinâmica de cores, contraste garantido por tema, sistema de movimento com molas, vigia de `requestAnimationFrame`, catálogo de componentes e padrões de UX.
6. **[docs/06-roteiro-de-implementacao-ia.md](docs/06-roteiro-de-implementacao-ia.md)** — O plano de execução passo a passo para agentes de IA, dividido em 6 fases com critérios de aceite e suíte de testes obrigatória em cada etapa.
7. **[docs/07-personalizacao-e-prompts.md](docs/07-personalizacao-e-prompts.md)** — Prompts prontos para copiar e colar no seu agente de IA (para construir do zero, para paralelizar telas com subagentes ou para criar novas funcionalidades depois de pronto) e guia de customização pessoal.

---

## Princípios de privacidade e segurança

- **Zero vazamento de dados pessoais**: seus dados reais vivem apenas dentro de `local-data/` (ignorado pelo `.gitignore`). Nunca coloque valores, extratos, nomes ou CPFs reais em testes ou commits.
- **Segredos apenas no `.env`**: `PLUGGY_CLIENT_ID`, `PLUGGY_CLIENT_SECRET` e `PLUGGY_ITEM_IDS` ficam exclusivamente no arquivo `.env` local (nunca no código nem no chat da IA).
- **Apenas leitura**: a conexão via Meu Pluggy / Open Finance é estritamente de leitura. O aplicativo só escuta em `127.0.0.1` (localhost).
