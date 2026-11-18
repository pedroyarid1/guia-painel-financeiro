# Guia de Construção: Painel Financeiro Pessoal (Instruções para o Agente de IA)

Você está em um repositório de **blueprint de engenharia**. Sua missão como agente de IA é construir (ou evoluir) uma aplicação web local de controle financeiro pessoal e investimentos, integrada à Pluggy (via Meu Pluggy / Open Finance) e ao Banco Central, seguindo rigorosamente as especificações da pasta `docs/`.

---

## 1. Ordem de leitura obrigatória antes de escrever código

Antes de iniciar qualquer fase, leia os documentos de referência abaixo:

1. `docs/06-roteiro-de-implementacao-ia.md` — **Seu plano de execução em 6 fases.** Siga a ordem das fases, implemente os testes pedidos em cada uma e nunca avance para a próxima fase com testes quebrados ou erros de TypeScript.
2. `docs/01-arquitetura-e-especificacao.md` — Stack, configuração do Node/TypeScript/Vite, estrutura de diretórios, as 8 migrações SQLite e requisitos das 7 telas.
3. `docs/02-pluggy-na-pratica.md` — Como a API real da Pluggy e o conector proxy `MeuPluggy` se comportam na prática (paginação v2, inferência de instituição, contas fantasmas, cartões sem data de fechamento, caixinhas do Nubank, leitura vazia quando a autorização expira).
4. `docs/03-regras-de-negocio-e-calculos.md` — Regras puras de domínio (centavos inteiros, normalização de sinal, chave local estável, árvore de 22 categorias + `custom-categories.json`, categorização em camadas, detecção de movimentações, definição única de gasto do mês, fila de revisão, TWR de investimentos, 12 sugestões).
5. `docs/04-contrato-da-api.md` — Contrato completo das rotas REST (`/api/*`), parâmetros, links entre telas (`shared/links.ts`) e tipos compartilhados (`shared/types.ts`).
6. `docs/05-design-system-e-ux.md` — Design System Material 3 Expressive próprio, tokens gerados por `@material/material-color-utilities`, molas físicas, vigia de `requestAnimationFrame` (`data-frames="paused"`), componentes base, componentes financeiros e gráficos Recharts.
7. `docs/07-personalizacao-e-prompts.md` — Como adaptar o painel às preferências pessoais do usuário sem sujar o motor principal.

---

## 2. Regras inegociáveis de arquitetura e código

- **Idioma**: textos da interface, mensagens de erro da API, documentação e conversas com o usuário em **português (pt-BR)**. Código, nomes de variáveis/funções/tipos e mensagens de commit em **inglês** (imperativo, commits pequenos por fase/onda).
- **TypeScript nativo no Node**: o servidor roda TypeScript diretamente pelo Node (`node --watch server/src/index.ts`, Node 24+). Portanto:
  - Todos os imports relativos devem incluir a extensão `.ts` ou `.tsx`;
  - Não use `enum`, `namespace` nem `parameter properties` em construtores (incompatíveis com o strip-types nativo do Node).
- **Dinheiro sempre em centavos inteiros**: em todo o banco SQLite e em todos os payloads JSON da API (`...Cents`). Nunca use ponto flutuante para guardar ou somar dinheiro.
- **Sinal normalizado**: internamente (`spend_cents` no banco e `amountCents` na API), **gasto é sempre positivo (`> 0`) e entrada/receita é sempre negativa (`< 0`)**, tanto para conta corrente quanto para cartão de crédito. O valor bruto original da fonte fica guardado em `amount_original_cents`.
- **Uma definição única de Gasto do Mês**: a soma das categorias-mãe que não são de movimentação (`03000000` Investimentos, `04000000` Transferência entre contas próprias) e cujo saldo líquido no mês é positivo (`> 0`). Categorias com saldo líquido zero/negativo no mês ou de movimentação vão para `outside` ("Fora dos gastos"). Um teste automatizado deve provar que `soma(lista de categorias) === total do donut === barra do mês no gráfico de 12 meses === card da Visão geral === "Saiu" no fluxo de caixa`, tanto no critério por fatura (`basis=bill`) quanto por data da compra (`basis=purchase`).
- **Funciona 100% offline com dados fictícios (`DATA_SOURCE=mock`)**: sem arquivo `.env`, a aplicação sobe com um gerador determinístico (com semente) que simula ~14 meses de transações, cartões, faturas, investimentos, snapshots diários, índices do Banco Central e preços de ações/ETFs.
- **Segurança e privacidade**:
  - O servidor só escuta em `127.0.0.1`.
  - Segredos (`PLUGGY_CLIENT_ID`, `PLUGGY_CLIENT_SECRET`, `PLUGGY_ITEM_IDS`) ficam exclusivamente no `.env` (fora do git). Nunca imprima nem commite segredos.
  - Dados financeiros reais e categorias muito pessoais ficam exclusivamente em `local-data/` (fora do git), inclusive `local-data/custom-categories.json`. Nunca use dados reais do usuário em testes, fixtures ou commits.
- **Desfazer real**: toda ação mutável do usuário (recategorizar, criar/apagar regra, marcar movimentação, confirmar fila de revisão, dispensar sugestão, editar/excluir investimento manual, ocultar conta) deve oferecer um botão "Desfazer" no snackbar que chama uma rota da API capaz de restaurar o estado exato anterior no banco. Nunca simule "Desfazer" com `setTimeout` no frontend.
- **Conteúdo nunca depende do fim de uma animação**: abas em segundo plano ou janelas encobertas pausam `requestAnimationFrame`. Use o vigia de quadros em `web/src/design/motion.ts` (`data-frames="paused"` + `[data-settle]`) e o componente `<Presence>` para que nenhum conteúdo fique invisível ou preso.

---

## 3.Continuidade entre sessões e agentes

Se o usuário trocar de agente de IA no meio da construção (ou quando o limite de contexto/uso acabar), o próximo agente deve conseguir retomar imediatamente apenas lendo os arquivos do projeto:
1. Ao iniciar qualquer sessão, rode `git status`, `git log --oneline -5` e `npm test` (se `package.json` já existir).
2. Mantenha a seção **"Estado atual"** e **"Decisões"** deste `AGENTS.md` atualizadas ao final de cada fase concluída.

---

## 4. Estado atual

- **2026-11-18**: Blueprint completo criado em `docs/` (`01` a `07`), pronto para um agente de IA iniciar a implementação a partir da **Fase 1** de `docs/06-roteiro-de-implementacao-ia.md`.

## 5. Decisões do projeto

- (Registre aqui as escolhas pessoais do usuário ao iniciar a implementação: cor-semente `SEED_COLOR`, categorias extras em `local-data/custom-categories.json`, bancos utilizados e customizações pedidas.)
