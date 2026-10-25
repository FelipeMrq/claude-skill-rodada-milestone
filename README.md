# rodada-milestone

Skill do [Claude Code](https://claude.ai/code) que executa os itens pendentes de uma milestone **um por vez**, cada um numa sessão própria do Claude Code na nuvem, até todos terminarem com PR aberto. Você aprova a fila no início e revisa os PRs no fim.

## Como uma rodada anda

1. **Você aprova a fila.** A skill lista os itens pendentes e as autorizações da rodada. É o único momento em que ela para antes do fim.
2. **Uma sessão por item.** Cria a sessão do item, que implementa, roda a verificação do projeto (build, lint, testes), abre o PR e avisa a sessão principal.
3. **Revisão antes de seguir.** Confere CI verde, conflito com os outros PRs abertos, versão de migration e achados fora do escopo. Só depois cria a sessão do próximo item.
4. **Relatório no fim.** Tabela de PRs, ordem de merge recomendada, o que você precisa aplicar e o que ficou pendente.

## O que ela já resolve

- **Migrations com a mesma versão.** Antes de cada item, reserva uma versão maior que a de todos os PRs abertos. Rodar em sequência não basta: a sessão nova parte da branch base, que ainda não tem as migrations dos PRs não mergeados.
- **Limite de uso.** O progresso fica num arquivo versionado no git e uma tarefa agendada dispara de hora em hora. Quando a cota volta, a rodada continua de onde parou.
- **Achado de segurança no meio do caminho.** A rodada pausa e pergunta antes de seguir.
- **Nada vai para produção sozinho.** Nenhuma sessão faz merge, aplica migration ou publica deploy.

## Instalar

1. Copie `.claude/skills/rodada-milestone/SKILL.md` para o mesmo caminho no seu repositório e faça commit.
2. Preencha a tabela **Configuração do projeto** no topo do arquivo: quem aprova, pasta dos planos, branch de produção, comandos de verificação e pasta de migrations (apague as partes de migration se o projeto não tiver).
3. Numa sessão nova do Claude Code, peça `/rodada-milestone` ou "executa a milestone 03 inteira".

## Pré-requisitos

- Sessões do Claude Code na nuvem (claude.ai/code) com acesso ao GitHub do repositório. A skill usa as ferramentas de sessões remotas: criar sessão, mandar mensagem entre sessões e tarefas agendadas.
- Um plano em arquivos: uma milestone listando os itens com status (⬜ 🔄 ✅ ⏸️) e um arquivo de plano por item.

## Origem

Nasceu de uma rodada real: quatro itens executados em paralelo geraram duas migrations com a mesma versão. Esta versão roda em sequência e reserva a versão antes de cada item.
