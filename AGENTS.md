# guia-painel-financeiro

Blueprint completo para construir um painel financeiro pessoal com Pluggy e agentes de IA

- **Área:** pessoal. Criado em 2026-11-18.
- **Para quem:** (Pedro, ou o parente/colega)
- **Tipo:** (automação, site, design, outro)

## Continuidade entre IAs
O Pedro alterna entre várias IAs (Claude, Codex e outras). Qualquer uma precisa conseguir retomar o trabalho só pelos arquivos, sem o histórico de conversa:
- Mantenha "Estado atual" e "Decisões" abaixo sempre em dia, atualizando ao fim de cada bloco de trabalho (não só no fim do projeto).
- Quando o projeto crescer, crie `docs/continuar-com-outra-ia.md` (ordem de leitura, primeiro passo da sessão, regras inegociáveis, o que está em andamento, backlog, operação do dia a dia e armadilhas) e aponte para ele aqui.
- Decisões tomadas em conversa vão para este arquivo, não ficam só na conversa.
- Ao começar uma sessão, rode `git status`: mudanças sem commit podem ser trabalho interrompido de outra IA.

## Como rodar
(comandos para instalar e rodar; atualize quando mudar)

## Estado atual
(o que já funciona, o que falta, próximos passos)

## Decisões
(escolhas importantes e o porquê, com data)

## Regras do projeto
- Conversa e documentação em português; código, nomes e commits em inglês.
- Segredos só no `.env`, fora do git. Mantenha o `.env.example` atualizado.
- Prefira a solução mais simples de manter.
- Dados pessoais do Pedro ficam em C:\Users\Pedro\IA\assistente e entram por um caminho no `.env`, nunca no repositório.
