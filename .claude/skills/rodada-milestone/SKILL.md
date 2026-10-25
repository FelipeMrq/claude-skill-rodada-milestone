---
name: rodada-milestone
description: Executa em sequência os itens pendentes de uma milestone (ou de um plano), uma sessão filha por item, cada uma terminando com um PR aberto, até zerar a lista. A rodada retoma sozinha depois de um limite de uso. Use quando pedirem "rodada", "executar a milestone inteira", "zerar a milestone NN" ou invocarem /rodada-milestone.
---

# Rodada de milestone (orquestração sequencial)

Você é a **sessão orquestradora**. Você não implementa itens: cria uma sessão
filha por item, **uma de cada vez**, revisa o PR que ela abriu e só então passa
ao próximo. Quem revisa e mergeia os PRs é o responsável pelo projeto (o
"dono"). Nada vai para produção sem a aprovação dele.

## Configuração do projeto (ajuste antes de usar)

| Chave | Exemplo | O que é |
|---|---|---|
| `DONO` | Maria | Quem aprova a rodada e revisa os PRs |
| `PASTA_PLANOS` | `PLAN/milestones/<NN>/` | Onde ficam a milestone e as issues |
| `ARQUIVO_MILESTONE` | `milestone.md` | Lista de itens com status (⬜ 🔄 ✅ ⏸️) |
| `PASTA_ISSUES` | `issues/<slug>.md` | Um arquivo de plano por item |
| `BACKLOG` | `PLAN/backlog.md` | Para onde vão achados fora do escopo |
| `BRANCH_BASE` | `main` | Branch de produção |
| `PREFIXO_BRANCH` | `claude/m<NN>-<slug>` | Branch de cada item |
| `VERIFICACAO` | `npm run build`, `npm run lint`, `npm test` | O que cada filha roda antes do PR |
| `PASTA_MIGRATIONS` | `supabase/migrations/` (versão = prefixo numérico) | Apague as partes de migration se o projeto não tiver |
| `REGRAS_EXTRAS` | checklist de segurança, regra de commits | Regras do `CLAUDE.md` que o PR precisa cumprir |

**Pré-requisitos:** sessões do Claude Code na nuvem (claude.ai/code), com as
ferramentas `create_session`, `send_message`, `get_session`, `list_events`,
`create_trigger`/`delete_trigger`, `send_later` e `subscribe_pr_activity`, além
de acesso ao GitHub do repositório.

Argumentos (pergunte o que faltar, **numa única pergunta**):
- `milestone`: `NN` (padrão: a maior em `PASTA_PLANOS`) ou o caminho de um plano.
- `itens`: todos os ⬜, ou uma lista. Ações manuais (sem código) ficam de fora
  por padrão.
- `autorizacoes` (confirmar explicitamente; valem só nesta rodada):
  - [ ] pular a aprovação de plano/UX antes do código (a revisão acontece no PR);
  - [ ] migrations: escrever e deixar prontas, **nunca** aplicar nem fazer deploy;
  - [ ] o que fazer com achados de segurança graves fora do escopo (padrão:
    **pausar a rodada e perguntar**).

## 0. Preparação (uma vez)

1. Leia a milestone e as issues dos itens escolhidos. Monte a fila na ordem da
   milestone, respeitando as dependências declaradas. Item sem código (ação
   manual, decisão de negócio) não entra na fila: liste-o no relatório como
   "pendente do `DONO`".
2. Apresente a fila e as autorizações ao `DONO` e **espere o "pode seguir"**.
   Esse é o único checkpoint da rodada antes do relatório final.
3. Descubra o próprio id de sessão (`get_session` sem argumento).
4. Crie o **arquivo de estado** `PASTA_PLANOS/rodada-<AAAA-MM-DD>.md` no branch
   da orquestradora (formato no fim deste arquivo), faça commit e push. O
   estado vive no git porque o container pode ser reciclado.
5. Arme o **heartbeat**: `create_trigger` ligado a ESTA sessão,
   `cron_expression: "0 * * * *"` (de hora em hora), com o prompt:
   > Heartbeat da rodada-milestone <NN>: siga a skill rodada-milestone, seção "Retomada". Leia o arquivo de estado e continue de onde parou. Se não houver nada a fazer, não escreva nada.

   Anote o `trigger_id` no estado. É ele que traz a rodada de volta depois de
   um limite de uso: os disparos durante o bloqueio falham, e o primeiro depois
   que a cota volta retoma o trabalho.

## 1. Loop por item (estritamente sequencial)

Para o próximo item ⬜ da fila:

1. **Atualize a base:** `git fetch origin <BRANCH_BASE>` e os branches de
   todos os PRs abertos da rodada.
2. **Reserve a versão de migration** (pule se o projeto não tiver migrations).
   Calcule a maior versão em `PASTA_MIGRATIONS` considerando a `BRANCH_BASE`
   **e** todos os PRs abertos. Sequencial não basta: a sessão nova parte da
   base, que não tem as migrations dos PRs ainda não mergeados. O próximo item
   recebe uma versão **maior** que esse máximo.
3. **Crie a sessão filha** (`create_session`) com o repositório,
   `outcome_branch: <PREFIXO_BRANCH>` e o prompt do **Template da sessão
   filha** (abaixo), preenchido.
4. Registre no estado: item 🔄, `session_id`, branch e hora. Faça commit e push.
5. Arme um **watchdog** com `send_later` de 45 min: "Checar a sessão filha
   <id> do item <slug>". Se a filha avisar antes (passo 6), cancele o watchdog
   com `delete_trigger`.
6. **Termine o turno e espere.** A filha avisa por `send_message` quando abrir
   o PR. Não faça polling com `sleep`.

### Quando a filha avisar, ou o watchdog/heartbeat disparar

- `get_session` da filha. Se o `status_bucket` for:
  - `completed` com PR: siga para **Revisão do PR**.
  - `working`: re-arme o watchdog e termine o turno.
  - `failed` ou idle sem PR: veja o motivo com `list_events` (kinds
    `assistant` e `result`). Se foi limite de uso ou erro transitório, mande
    `send_message`: "Continue de onde parou e termine com o PR aberto". Se foi
    bloqueio real (falta decisão ou acesso), marque o item ⏸️ com o motivo e
    passe ao próximo.

### Revisão do PR (antes de liberar o próximo item)

1. CI verde no head. Se estiver vermelho, peça a correção à filha via
   `send_message` e espere.
2. Arquivos do PR: confira que não tocou o arquivo da milestone e que a versão
   da migration é a reservada.
3. **Conflitos cruzados:** rode `git merge-tree --write-tree` do PR novo contra
   a base e contra cada PR aberto da rodada. Conflito entre dois PRs abertos
   não é erro: anote no estado e no relatório (a orquestradora resolve no
   merge do primeiro).
4. Leia a seção **"Para o `DONO` aplicar/validar"** e os **achados fora do
   escopo** da descrição do PR.
   - Achado de **segurança grave** (vazamento de dados, escalada de
     privilégio, credencial exposta): confirme no código e **pause a rodada**.
     Avise o `DONO` e proponha um hotfix como próximo item. Não siga até ele
     responder.
   - Demais achados: anote para entrarem no `BACKLOG` no PR de docs.
5. Inscreva-se no PR (`subscribe_pr_activity`) para cuidar de CI e conflitos
   depois.
6. Marque o item como "PR #N aberto", faça commit e push do estado, e volte ao
   **Loop** com o próximo item.

## 2. Encerramento (fila vazia)

1. Abra um **PR de docs** com:
   - a milestone atualizada (🔄 + link do PR por item; ⏸️ com motivo);
   - os achados no `BACKLOG`;
   - a remoção do arquivo de estado (ou mova-o para um histórico).
2. `delete_trigger` no heartbeat e em qualquer watchdog pendente.
3. Relatório final ao `DONO`: uma tabela de item, PR, migration (sim/não),
   deploys pendentes e o que ficou sem validar. Depois:
   - **ordem de merge recomendada**, considerando dependências, conflitos e a
     ordem das migrations;
   - itens pendentes dele (manuais e ⏸️);
   - achados de segurança.

## Retomada (heartbeat, watchdog ou sessão reiniciada)

1. Leia o arquivo de estado (`git fetch` + `git show origin/<branch>:<arquivo>`).
2. Reconstrua a verdade pelo GitHub: para cada item, existe PR do branch
   `<PREFIXO_BRANCH>`? O estado do git perde para o GitHub.
3. Item 🔄 sem PR: veja a filha (bloco "Quando a filha avisar").
4. Nenhum item 🔄: volte ao **Loop**.
5. Fila vazia e heartbeat ainda ativo: vá para **Encerramento**.
6. Se nada mudou desde o último disparo, termine o turno **sem mensagem**.

## Template da sessão filha

```
Implementa o item <N> (`<slug>`) da milestone <NN>: <resumo>. Plano em `<PASTA_ISSUES>`. Siga a issue e não re-analise o que já foi diagnosticado nela. Branch: `<PREFIXO_BRANCH>`.

## Regras desta rodada (autorizadas pelo <DONO>, valem só para esta sessão)
- <autorização de plano/UX>: implemente direto. Se houver tela nova, descreva-a no PR (ou publique um protótipo e coloque o link).
- Migration: use a versão **<VERSAO_RESERVADA>** (ou maior, se precisar de mais de uma). NUNCA aplique migration nem faça deploy. Valide localmente e diga no PR o que foi e o que não foi validado.
- Não edite o arquivo da milestone nem o backlog. A orquestradora cuida deles.
- Nada de merge, auto-merge ou push na `<BRANCH_BASE>`.
- Achado fora do escopo: não corrija. Registre no PR, numa seção "Achados fora do escopo", com gravidade.

## Entrega
1. Rode <VERIFICACAO>. Zero erro novo nos arquivos tocados.
2. Siga <REGRAS_EXTRAS>.
3. Commits, push e PR contra a `<BRANCH_BASE>`, com: o que foi feito; achados fora do escopo; seção "Para o <DONO> aplicar/validar". Não afirme que testou o que não rodou.
4. **Ao terminar, avise a orquestradora:** `send_message` para `session_id: <ID_DA_ORQUESTRADORA>` com "PR #<n> aberto: <link> — <resumo de 1 linha>". Se travar sem conseguir terminar, envie o motivo do mesmo jeito.
```

## Formato do arquivo de estado

```markdown
# Rodada M<NN> — <data>
orquestradora: <session_id> · heartbeat: <trigger_id> · branch: <branch>
autorizacoes: <lista>

| Ordem | Item | Status | Sessão | Branch | PR | Migration |
|---|---|---|---|---|---|---|
| 1 | <slug> | ✅ PR aberto / 🔄 / ⬜ / ⏸️ <motivo> | session_… | <branch> | #N | <versão>/— |

## Conflitos entre PRs
## Achados para o backlog
## Log
- <hora> — <evento>
```

## Nunca

- Criar a próxima filha antes do PR da anterior estar aberto e revisado.
- Aplicar migration, fazer deploy, mergear, ativar auto-merge ou fazer push na `BRANCH_BASE`.
- Seguir a rodada com um achado de segurança grave sem resposta do `DONO`.
- Deixar heartbeat ou watchdog armado depois do encerramento.
