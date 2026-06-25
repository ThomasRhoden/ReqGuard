# ReqGuard — Plataforma de Gestão de Requisitos e Conformidade

| Campo | Valor |
|---|---|
| **Versão do documento** | 2.2 |
| **Pacote documental** | v3.1 |
| **Documentos relacionados** | `especificacao_sistema.md`, `arquitetura_agentes.md`, `schema_projeto.md`, `versionamento.md` |

O ReqGuard é uma plataforma para gestão do ciclo de vida de requisitos de software, do levantamento com stakeholders até o backlog priorizado, orquestrada por agentes de IA e com conformidade à LGPD (Lei nº 13.709/2018) integrada em cada etapa do fluxo.

---

## 1. Documentação do Projeto

| Documento | Conteúdo |
|---|---|
| [`especificacao_sistema.md`](./especificacao_sistema.md) | Escopo, módulos do sistema, perfis de acesso (RBAC), regras de negócio e requisitos não funcionais (ISO/IEC 25010) |
| [`arquitetura_agentes.md`](./arquitetura_agentes.md) | Especificação dos 6 agentes de IA, diagrama de fluxo, máquina de estados, protocolo de segurança, resiliência e observabilidade |
| [`schema_projeto.md`](./schema_projeto.md) | Stack, estrutura de pastas, schema de banco (PostgreSQL) e contratos de API |
| [`versionamento.md`](./versionamento.md) | Política de versionamento de documentação, requisitos, schema, API, prompts de agentes e código-fonte |

---

## 2. Objetivo

Consolidar, num único sistema, as práticas de Engenharia de Requisitos (IEEE 830 / ISO 29148), vinculando necessidades de negócio à arquitetura técnica e à conformidade regulatória.

---

## 3. Módulos do Sistema

| Módulo | Escopo |
|---|---|
| **A — Gestão de Stakeholders** | Mapeamento Poder × Interesse, matriz RACI, plano de comunicação |
| **B — Engenharia de Requisitos** | Elicitação, especificação de RF/RNF, casos de uso, user stories (Gherkin), priorização MoSCoW |
| **C — Rastreabilidade** | Vínculo bidirecional Caso de Uso ↔ Requisito ↔ Regra de Negócio ↔ Caso de Teste ↔ User Story |
| **D — Privacy by Design / LGPD** | Classificação de dados, bases legais, trilha de auditoria imutável, módulo do titular |

Detalhes completos em [`especificacao_sistema.md`](./especificacao_sistema.md).

---

## 4. Agentes de Inteligência Artificial

| Sigla | Nome | Responsabilidade principal |
|---|---|---|
| **AER** | Agente de Elicitação de Requisitos | Processa transcrições/atas e extrai elementos brutos de necessidade |
| **AES** | Agente de Especificação de Requisitos | Transforma elementos brutos em RF/RNF estruturados |
| **AR** | Agente de Rastreabilidade | Vincula requisito ↔ caso de uso ↔ regra de negócio ↔ caso de teste ↔ user story |
| **AC** | Agente de Conformidade (LGPD) | Classifica dados pessoais e valida base legal de cada requisito |
| **APB** | Agente de Priorização de Backlog | Classifica requisitos aprovados na matriz MoSCoW |
| **AUS** | Agente de User Stories | Gera user stories e critérios de aceitação em Gherkin, valida formato SMART |

---

## 5. Fluxo de Comunicação entre Agentes

```
Stakeholder → AER → AES → (AR ⇄ AC, validação cruzada) → APB → AUS → Repositório / Jira
```

O fluxo não é um ciclo fechado: nasce no levantamento com o stakeholder e termina na entrega ao repositório/Jira. O único ponto não-linear é entre AR e AC, que se validam mutuamente antes de liberar o requisito para priorização — se o AC reprovar por falta de base legal, o requisito retorna para revisão. Esse ciclo é limitado a 3 tentativas antes de escalar para revisão humana.

Diagrama completo, system prompts e schemas JSON de cada agente em [`arquitetura_agentes.md`](./arquitetura_agentes.md).

---

## 6. Perfis de Acesso (RBAC)

| Perfil | Acesso principal |
|---|---|
| **ADMIN** | Acesso total — usuários, projetos e versões de prompts dos agentes |
| **ANALISTA_REQUISITOS** | Módulos A, B e C |
| **DPO** | Módulo D — conformidade LGPD e solicitações do titular |
| **PRODUCT_OWNER** | Backlog priorizado (APB) e user stories (AUS) |
| **STAKEHOLDER_VIEWER** | Leitura apenas |

Matriz completa de permissões por módulo em [`especificacao_sistema.md §6`](./especificacao_sistema.md).

---

## 7. Segurança e Privacidade

- Dado potencialmente real de usuário final passa por tokenização/anonimização antes de qualquer chamada a uma API de LLM.
- A cadeia de mensagens entre agentes é armazenada com hash SHA-256 encadeado, tornando a trilha de auditoria imutável.
- O direito à eliminação de dados do titular (LGPD) é resolvido sem violar essa imutabilidade: apaga-se o mapeamento token ↔ valor real no vault, nunca o registro de auditoria.
- RBAC e mascaramento de dados sensíveis na interface para perfis sem autorização adequada.
- Criptografia em repouso e em trânsito para dados classificados como sensíveis.
- Nenhuma entidade rastreável é excluída fisicamente — remoção é sempre uma exclusão lógica.

Protocolo técnico completo em [`arquitetura_agentes.md §4`](./arquitetura_agentes.md).

---

## 8. Stack Tecnológica

**Backend:** Python 3.12 + FastAPI + SQLAlchemy 2.0 + Pydantic v2
**Banco de dados:** PostgreSQL · **Migrations:** Alembic · **Auth:** JWT + RBAC

Justificativa, estrutura de pastas e schema de banco em [`schema_projeto.md`](./schema_projeto.md).

---

## 9. Roadmap

- Sistema de notificações para mudanças em requisitos ou conformidade.
- Módulo de análise de impacto (o que muda na LGPD/rastreabilidade quando um requisito muda).
- Integração com Jira/Trello para sincronização de backlog.
- Dashboard em tempo real (status de requisitos, conformidade, rastreabilidade).
- Suporte a múltiplos idiomas.
- Sugestões de melhoria de requisitos via aprendizado de máquina sobre histórico do próprio projeto.
- Painel administrativo para gestão de usuários, perfis RBAC e projetos.
