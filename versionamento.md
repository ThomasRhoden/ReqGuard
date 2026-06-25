# 📦 Versionamento — ReqGuard

> **Versão 1.0** — Documento novo (5º documento da documentação do projeto). Define a política de versionamento em todas as camadas que podem mudar com o tempo: documentação, requisitos (dado em runtime), schema de banco, contratos de API, prompts dos agentes de IA, e o código-fonte via Git.

---

## 1. Por que isso importa neste projeto especificamente

O ReqGuard existe para garantir rastreabilidade e auditoria (Módulo C e D, `especificacao_sistema.md`). Isso só é credível se o próprio projeto pratica o que prega: cada camada que pode mudar precisa de um histórico verificável. Sem isso, a trilha de auditoria com hash encadeado (`arquitetura_agentes.md §4`) protege os *dados*, mas não protege contra um prompt de agente mudar silenciosamente e ninguém saber qual versão gerou qual decisão de conformidade.

---

## 2. Visão Geral das Camadas de Versionamento

| Camada | O que versiona | Mecanismo | Detalhado em |
|---|---|---|---|
| Documentação (`.md`) | Os próprios documentos do projeto | Cabeçalho `> Versão X.Y` + changelog (§10) | §3 |
| Requisitos (entidade) | Histórico de cada requisito ao longo do ciclo de vida | Tabela `requisitos_versoes` | §4 |
| Schema de banco | Estrutura das tabelas | Alembic (migrations encadeadas) | §5 |
| API | Contratos de rota | Versionamento de URL (`/api/v1`, `/api/v2`) | §6 |
| Prompts dos agentes de IA | System prompt de cada agente | Tabela `prompts_agentes` | §7 |
| Código-fonte | Histórico de commits | Git + Conventional Commits + SemVer | §8–9 |

---

## 3. Versionamento da Documentação (`.md`)

**Convenção:** todo documento abre com um blockquote `> **Versão X.Y** — <o que mudou>`. Já em uso desde a revisão anterior.

- **MAJOR** (`2.0`, `3.0`...): mudança que quebra compatibilidade com outro documento — ex: renomear uma sigla de agente, mudar um schema JSON que outro documento referencia.
- **MINOR** (`2.1`, `2.2`...): adição de conteúdo que não quebra nada existente — ex: novo endpoint, nova tabela, nova seção.

> Não usamos PATCH em nível de documento (correção de digitação não merece um número de versão) — só MAJOR.MINOR.

**Mapa de versão atual:**

| Documento | Versão atual |
|---|---|
| `README.md` | 2.0 |
| `especificacao_sistema.md` | 2.0 |
| `arquitetura_agentes.md` | 2.0 |
| `schema_projeto.md` | 1.1 *(bump nesta entrega — ver §10)* |
| `versionamento.md` | 1.0 |

---

## 4. Versionamento de Requisitos (dado em runtime)

O schema atual (`schema_projeto.md`) guarda só o **estado atual** de um requisito (`requisitos.estado`, `atualizado_em`) — o histórico de como ele chegou ali se perde a cada update. Isso é exatamente o item que já estava no roadmap do `README.md`: *"Implementar um sistema de versionamento de requisitos para acompanhar alterações ao longo do tempo."* Aqui está a implementação.

```sql
CREATE TABLE requisitos_versoes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    numero_versao INTEGER NOT NULL,
    estado_anterior VARCHAR(30),
    estado_novo VARCHAR(30) NOT NULL,
    snapshot JSONB NOT NULL,           -- cópia completa do requisito nesse momento
    alterado_por_agente VARCHAR(10),   -- AER, AES, AR, AC, APB, AUS ou 'MANUAL'
    motivo_alteracao TEXT,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (requisito_id, numero_versao)
);
```

**Quando é criado:** a cada transição de estado da máquina de estados (`arquitetura_agentes.md §2`) — ou seja, toda vez que um agente move um requisito de `ESPECIFICADO` → `EM_RASTREIO` → `EM_ANALISE_LGPD` etc.

**Relação com o `log_auditoria` (hash chain):** são duas coisas diferentes que devem nascer juntas, do mesmo evento:

- `log_auditoria` é o **backbone criptográfico** — prova que a sequência de eventos não foi adulterada (hash encadeado).
- `requisitos_versoes` é a **visão amigável** — permite responder rápido "como estava o RF-PAG-001 na versão 3?" sem precisar recalcular a cadeia de hash inteira.

> **Regra de implementação:** o `snapshot` salvo em `requisitos_versoes` deve ser **o mesmo payload JSON** que é hasheado em `log_auditoria.payload` para aquele evento. Uma única fonte de verdade, gravada uma vez, referenciada nos dois lugares — nunca gerar o snapshot duas vezes de formas diferentes, ou os dois registros podem divergir silenciosamente.

**Onde implementar:** dentro do código de cada agente (`services/agentes/*.py`), no momento em que ele decide a transição — não via trigger de banco. Trigger garantiria a gravação, mas não tem acesso fácil ao `motivo_alteracao` legível que o agente já tem em contexto (ex: a justificativa que o AC escreveu ao reprovar um requisito).

---

## 5. Versionamento de Schema de Banco (Alembic)

- Toda mudança de schema = uma migration nova (`alembic revision --autogenerate -m "descrição clara"`). Nunca editar uma migration que já foi aplicada em produção.
- Migration de correção de uma anterior = **nova migration**, não edição retroativa — mesma lógica da imutabilidade do `log_auditoria`.
- Mudanças destrutivas (`DROP COLUMN`, `DROP TABLE`) sempre em duas etapas: 1) migration que deixa de usar a coluna/tabela no código, 2) migration seguinte (em outro deploy) que remove de fato. Evita perda de dados se algo precisar ser revertido rápido.
- Cada migration aplicada deve corresponder a uma versão MINOR ou MAJOR de `schema_projeto.md`, dependendo se é aditiva ou quebra compatibilidade (mesma régua do §3).

---

## 6. Versionamento de API

- URL versioning, já adotado: `/api/v1/...`.
- **Breaking change** (campo removido, tipo de campo mudou, rota removida) → nova versão major na URL (`/api/v2/...`), com a v1 mantida ativa por um período de transição definido (ex: 90 dias) e um header `Deprecation` nas respostas da v1 avisando a data de desligamento.
- **Non-breaking change** (campo novo opcional, rota nova) → mesma versão de URL, só atualiza a documentação.
- Nunca remover um campo de resposta sem passar por depreciação — quem consome a API (incluindo os próprios agentes internos) precisa de tempo pra se adaptar.

---

## 7. Versionamento de Prompts dos Agentes de IA

Esta é a camada mais fácil de esquecer e a mais crítica pra auditoria: se o system prompt do **AC** mudar o critério de quando aprovar ou reprovar um requisito por LGPD, qualquer decisão antiga precisa continuar explicável com a versão do prompt que estava ativa **naquele momento** — não com a versão atual.

```sql
CREATE TABLE prompts_agentes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agente VARCHAR(10) NOT NULL,        -- AER, AES, AR, AC, APB, AUS
    versao VARCHAR(10) NOT NULL,        -- ex: '1.0', '1.1', '2.0'
    system_prompt TEXT NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agente, versao)
);

-- Extensão da trilha de auditoria para registrar QUAL versão de prompt gerou cada decisão
ALTER TABLE log_auditoria ADD COLUMN prompt_versao VARCHAR(10);
```

**Regra de uso:** nunca editar o `system_prompt` de uma linha existente. Para mudar o comportamento de um agente: inserir uma nova linha com `versao` incrementada, marcar a antiga como `ativo = false`, e a partir daí toda nova execução do agente referencia a nova versão em `log_auditoria.prompt_versao`. Os prompts completos atuais (versão `1.0` de cada um dos 6 agentes) estão documentados em `arquitetura_agentes.md §3`.

---

## 8. Convenção de Commits e Branches (Git)

**Commits — [Conventional Commits](https://www.conventionalcommits.org/):**

| Prefixo | Uso |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug |
| `docs:` | mudança só de documentação |
| `refactor:` | mudança de código sem alterar comportamento |
| `chore:` | tarefas de manutenção (deps, config) |
| `test:` | adição/ajuste de testes |

**Branches:**
- `main` — sempre em estado de produção, protegida.
- `develop` — branch de integração.
- `feature/<nome>` — uma funcionalidade por branch (ex: `feature/agente-aer`).
- `fix/<nome>` — correção de bug isolada.

**Releases:** tag `vX.Y.Z` (SemVer, §9) criada na `main` a cada release.

---

## 9. Versionamento Semântico do Sistema (SemVer)

`MAJOR.MINOR.PATCH`

- **MAJOR** — quebra de compatibilidade: schema de banco incompatível, contrato de API quebrado, agente removido ou renomeado.
- **MINOR** — funcionalidade nova e compatível: novo agente, novo módulo, novo endpoint.
- **PATCH** — correção de bug ou ajuste de prompt que não muda schema nem contrato.

---

## 10. Changelog do Projeto

| Versão | Data | Mudanças |
|---|---|---|
| **v1.0** | — | Versão inicial conceitual: `README.md`, `especificacao_sistema.md` e `arquitetura_agentes.md` criados a partir do conteúdo da UC1. |
| **v2.0** | 2026-06-24 | Revisão completa dos 3 documentos: corrigida colisão de siglas AER/AES, fluxo de comunicação alinhado ao diagrama real (Mermaid), especificação completa dos 6 agentes (antes só o AER tinha system prompt), matriz de rastreabilidade corrigida com `caso_de_teste_id`, "Zero-Knowledge Logs" renomeado e corrigido, stack definida (PostgreSQL). Criado `schema_projeto.md` (4º documento) com estrutura de pastas, DDL completo e contratos de API. |
| **v2.1** | 2026-06-24 | Criado `versionamento.md` (5º documento). Adicionadas as tabelas `requisitos_versoes` e `prompts_agentes`, e a coluna `log_auditoria.prompt_versao` — refletidas em `schema_projeto.md` (bump para v1.1) para manter os documentos sincronizados. |

---

## 11. O que fica pra próxima iteração

- Decidir o período de depreciação padrão da API (sugestão: 90 dias) — colocar isso formalmente em `schema_projeto.md` quando a primeira mudança breaking de fato acontecer.
- Avaliar se `requisitos_versoes` precisa de um endpoint próprio (`GET /requisitos/{id}/historico`) — hoje não está no contrato de API de `schema_projeto.md`.
