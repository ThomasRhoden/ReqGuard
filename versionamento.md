# Versionamento — ReqGuard

| Campo | Valor |
|---|---|
| **Versão do documento** | 1.1 |
| **Pacote documental** | v3.1 |
| **Documentos relacionados** | `README.md`, `especificacao_sistema.md`, `arquitetura_agentes.md`, `schema_projeto.md` |

Define a política de versionamento em todas as camadas que podem mudar com o tempo: documentação, requisitos (dado em runtime), schema de banco, contratos de API, prompts dos agentes de IA, e o código-fonte via Git.

---

## 1. Por que isso importa neste projeto especificamente

O ReqGuard existe para garantir rastreabilidade e auditoria (Módulo C e D, `especificacao_sistema.md`). Isso só é credível se o próprio projeto pratica o que prega: cada camada que pode mudar precisa de um histórico verificável. Sem isso, a trilha de auditoria com hash encadeado (`arquitetura_agentes.md §4`) protege os dados, mas não protege contra um prompt de agente mudar silenciosamente e ninguém saber qual versão gerou qual decisão de conformidade.

---

## 2. Visão Geral das Camadas de Versionamento

| Camada | O que versiona | Mecanismo | Detalhado em |
|---|---|---|---|
| Documentação (`.md`) | Os próprios documentos do projeto | Tabela de metadados no topo do documento (§3) + changelog (§10) | §3 |
| Requisitos (entidade) | Histórico de cada requisito ao longo do ciclo de vida | Tabela `requisitos_versoes` | §4 |
| Schema de banco | Estrutura das tabelas | Alembic (migrations encadeadas) | §5 |
| API | Contratos de rota | Versionamento de URL (`/api/v1`, `/api/v2`) | §6 |
| Prompts dos agentes de IA | System prompt de cada agente | Tabela `prompts_agentes` | §7 |
| Código-fonte | Histórico de commits | Git + Conventional Commits + SemVer | §8–9 |

Além da versão própria de cada documento (MAJOR.MINOR, §3), o projeto usa um número de **pacote documental** (ex: `v3.1`) que amarra um conjunto de mudanças entregues junto. É o mesmo número usado nas linhas do changelog (§10) e estampado no cabeçalho de cada documento — por isso é normal e esperado que documentos individuais tenham versões diferentes entre si (`2.1`, `2.2`, `1.1`...) mesmo pertencendo ao mesmo pacote.

---

## 3. Versionamento da Documentação (`.md`)

**Convenção de cabeçalho:** todo documento abre com uma tabela de metadados:

| Campo | Valor |
|---|---|
| **Versão do documento** | X.Y |
| **Pacote documental** | vN.M |
| **Documentos relacionados** | lista dos `.md` referenciados |

Nenhum documento traz narrativa de "o que mudou nesta versão" no corpo do texto — essa explicação fica centralizada no changelog (§10), para manter cada documento como referência técnica limpa, sem histórico misturado ao conteúdo vigente.

- **MAJOR** (`2.0`, `3.0`...): mudança que quebra compatibilidade com outro documento — ex: renomear uma sigla de agente, mudar um schema JSON que outro documento referencia, ou alterar a semântica de uma coluna do banco que a API expõe.
- **MINOR** (`2.1`, `2.2`...): adição ou reformatação que não quebra nada existente — ex: novo endpoint, nova tabela, nova seção, mudança de formato de cabeçalho.

> Não usamos PATCH em nível de documento — só MAJOR.MINOR.

**Mapa de versão atual:**

| Documento | Versão do documento |
|---|---|
| `README.md` | 2.2 |
| `especificacao_sistema.md` | 2.2 |
| `arquitetura_agentes.md` | 2.2 |
| `schema_projeto.md` | 2.1 |
| `versionamento.md` | 1.1 |

---

## 4. Versionamento de Requisitos (dado em runtime)

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

**Quando é criado:** a cada transição de estado da máquina de estados (`arquitetura_agentes.md §2`) — incluindo transições para `BLOQUEADO_REVISAO_HUMANA`, onde o `motivo_alteracao` registra a justificativa da 3ª reprovação do AC.

**Relação com o `log_auditoria` (hash chain):**

- `log_auditoria` é o backbone criptográfico — prova que a sequência de eventos não foi adulterada.
- `requisitos_versoes` é a visão amigável — responde "como estava o RF-PAG-001 na versão 3?" sem recalcular a cadeia de hash inteira. Exposta via `GET /api/v1/requisitos/{id}/historico` (`schema_projeto.md §4`).

> **Regra de implementação:** o `snapshot` salvo em `requisitos_versoes` deve ser o mesmo payload JSON que é hasheado em `log_auditoria.payload` para aquele evento — uma única fonte de verdade, gravada uma vez, referenciada nos dois lugares.

**Onde implementar:** dentro do código de cada agente (`services/agentes/*.py`), no momento em que ele decide a transição — não via trigger de banco, que não teria acesso fácil ao `motivo_alteracao` legível que o agente já tem em contexto.

---

## 5. Versionamento de Schema de Banco (Alembic)

- Toda mudança de schema = uma migration nova. Nunca editar uma migration já aplicada em produção.
- Migration de correção de uma anterior = nova migration, não edição retroativa — mesma lógica da imutabilidade do `log_auditoria`.
- Mudanças destrutivas (`DROP COLUMN`, `DROP TABLE`) sempre em duas etapas: 1) migration que deixa de usar a coluna/tabela no código, 2) migration seguinte que remove de fato. Exceção aplicável só em produção — a conversão de `elicitacoes_brutas.projeto_id` no pacote v3.0 foi feita direto porque o projeto ainda está em fase de design, sem dados reais.
- Cada migration aplicada deve corresponder a uma versão MINOR ou MAJOR de `schema_projeto.md`, conforme §3.

---

## 6. Versionamento de API

- URL versioning, já adotado: `/api/v1/...`.
- Breaking change (campo removido, tipo de campo mudou, rota removida) → nova versão major na URL (`/api/v2/...`), com a v1 mantida ativa por um período de transição (ex: 90 dias) e header `Deprecation`.
- Non-breaking change (campo novo opcional, rota nova) → mesma versão de URL, só atualiza a documentação.

---

## 7. Versionamento de Prompts dos Agentes de IA

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

ALTER TABLE log_auditoria ADD COLUMN prompt_versao VARCHAR(10);
```

**Regra de uso:** nunca editar o `system_prompt` de uma linha existente. Para mudar o comportamento de um agente: inserir uma nova linha com `versao` incrementada, marcar a antiga como `ativo = false`, e a partir daí toda nova execução do agente referencia a nova versão em `log_auditoria.prompt_versao`. Gerenciado via `GET/POST /api/v1/agentes/prompts` (perfil `ADMIN`). Os prompts completos atuais (versão `1.0` de cada agente) estão em `arquitetura_agentes.md §3`.

---

## 8. Convenção de Commits e Branches (Git)

**Commits — Conventional Commits:**

| Prefixo | Uso |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug |
| `docs:` | mudança só de documentação |
| `refactor:` | mudança de código sem alterar comportamento |
| `chore:` | tarefas de manutenção (deps, config) |
| `test:` | adição/ajuste de testes |

**Branches:** `main` (produção, protegida) · `develop` (integração) · `feature/<nome>` · `fix/<nome>`.

**Releases:** tag `vX.Y.Z` (SemVer, §9) criada na `main` a cada release.

---

## 9. Versionamento Semântico do Sistema (SemVer)

`MAJOR.MINOR.PATCH`

- **MAJOR** — quebra de compatibilidade: schema de banco incompatível, contrato de API quebrado, agente removido ou renomeado.
- **MINOR** — funcionalidade nova e compatível: novo agente, novo módulo, novo endpoint.
- **PATCH** — correção de bug ou ajuste de prompt que não muda schema nem contrato.

---

## 10. Changelog do Projeto

| Pacote | Data | Mudanças |
|---|---|---|
| **v1.0** | — | Versão inicial conceitual: `README.md`, `especificacao_sistema.md` e `arquitetura_agentes.md` criados a partir do conteúdo da UC1. |
| **v2.0** | 2026-06-24 | Revisão completa dos 3 documentos: corrigida colisão de siglas AER/AES, fluxo de comunicação alinhado ao diagrama real (Mermaid), especificação completa dos 6 agentes, matriz de rastreabilidade corrigida com `caso_de_teste_id`, "Zero-Knowledge Logs" renomeado e corrigido, stack definida (PostgreSQL). Criado `schema_projeto.md` (4º documento). |
| **v2.1** | 2026-06-24 | Criado `versionamento.md` (5º documento). Adicionadas as tabelas `requisitos_versoes` e `prompts_agentes`, e a coluna `log_auditoria.prompt_versao`. |
| **v3.0** | 2026-06-25 | RBAC explícito (tabelas `usuarios`/`projetos`, 5 perfis); `elicitacoes_brutas.projeto_id` convertido de texto livre para FK `projetos(id)` (breaking); seed das 10 bases legais do Art. 7º da LGPD; índices e `atualizado_em` padronizados; estado `BLOQUEADO_REVISAO_HUMANA` com circuito de revisão limitado (RN-PROC-006); estratégia de retry/timeout para LLM; resolução da tensão eliminação LGPD × hash chain via *crypto-shredding* (RN-LGPD-005); não-exclusão física (RN-AUD-004); cobertura completa da ISO/IEC 25010; novos endpoints de API; `README.md` referenciando os 5 documentos. |
| **v3.1** | 2026-06-25 | **Novo formato documental:** cabeçalho passa de blockquote narrativo para tabela de metadados (Versão do documento / Pacote documental / Documentos relacionados); introduzido o conceito de pacote documental para amarrar versões individuais diferentes a uma mesma entrega. Removido todo conteúdo não-técnico: emojis em títulos e seções, parágrafo de contexto acadêmico/portfólio e a seção de créditos pessoais do `README.md`. Seções do `README.md` numeradas, igualando a convenção dos demais documentos. Adicionada RN-SEC-007 (isolamento por projeto/multi-tenant) em `especificacao_sistema.md` e nota correspondente em `schema_projeto.md §3`. Adicionada §6 "Observabilidade e Logs Operacionais" em `arquitetura_agentes.md`, distinguindo a trilha de auditoria de logs de debug/performance. |

---

## 11. O que fica pra próxima iteração

- Estratégia de testes/avaliação dos agentes (golden dataset, regressão de prompt a cada nova versão em `prompts_agentes`).
- Controle de custo e rate limit das chamadas a LLM (orçamento por projeto, alertas de consumo).
- Política de retenção dos logs operacionais (§6 de `arquitetura_agentes.md`) — hoje só `log_auditoria` tem regra explícita de retenção (permanente, por design).
- Pipeline de CI/CD e definição formal de ambientes (dev/staging/produção).
- Detalhar expiração e renovação (`refresh token`) da sessão JWT — hoje só "JWT + RBAC" está definido, sem TTL.
- Detalhar o fluxo de onboarding do primeiro usuário `ADMIN` em ambiente novo (problema de *bootstrap*, hoje só um passo manual em `schema_projeto.md §5`).
- Decidir o período de depreciação padrão da API (sugestão: 90 dias) — formalizar quando a primeira mudança breaking de fato acontecer em produção.
