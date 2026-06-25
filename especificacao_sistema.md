# Especificação do Sistema — ReqGuard

| Campo | Valor |
|---|---|
| **Versão do documento** | 2.2 |
| **Pacote documental** | v3.1 |
| **Documentos relacionados** | `README.md`, `arquitetura_agentes.md`, `schema_projeto.md` |

Este documento especifica o escopo funcional, os módulos, as regras de negócio e os requisitos não funcionais do ReqGuard — plataforma para gestão do ciclo de vida de requisitos de software com conformidade LGPD nativa.

---

## 1. Escopo e Objetivos do Sistema

O ReqGuard é uma ferramenta para equipes multidisciplinares (squads) gerenciarem o ciclo de vida de requisitos de software, garantindo que o sistema final esteja em total conformidade com a LGPD e vinculando as necessidades de negócio à arquitetura técnica.

---

## 2. Stack Tecnológica Definida

| Camada | Tecnologia |
|---|---|
| Backend | Python 3.12 + FastAPI |
| ORM / Validação | SQLAlchemy 2.0 + Pydantic v2 |
| Banco de dados | PostgreSQL (uso nativo de JSONB na trilha de auditoria, ver `arquitetura_agentes.md §4`) |
| Migrations | Alembic |
| Autenticação | JWT + RBAC (perfis detalhados em §6) |
| Frontend (futuro) | React + TypeScript |

---

## 3. Módulos Principais do Sistema

### Módulo A: Gestão de Stakeholders e Comunicação
- **Mapeamento de Impacto:** cadastro e classificação de stakeholders pela Matriz Poder × Interesse.
- **Matriz RACI Automatizada:** atribuição de Responsible/Accountable/Consulted/Informed por entrega ou requisito.
- **Plano de Comunicação:** canais (E-mail, Slack, Jira) definidos pelo perfil do stakeholder.
- *Agente envolvido:* nenhum — gestão manual de cadastro.

### Módulo B: Engenharia de Requisitos Eletrônica
- **Central de Elicitação:** registro de transcrições de entrevistas e questionários estruturados. *(Agente: AER)*
- **Especificação de RF e RNF:** RF em Linguagem Natural Controlada; RNF com métricas baseadas na ISO/IEC 25010. *(Agente: AES)*
- **Módulo de Casos de Uso:** Ator, Fluxo Principal, Alternativos e Exceções, com `<<include>>` e `<<extend>>`.
- **Fábrica de User Stories:** conversão de Casos de Uso em User Story (Como/Quero/Para que) com critérios Gherkin. *(Agente: AUS)*
- **Priorização de Backlog:** metodologia MoSCoW. *(Agente: APB)*

### Módulo C: Rastreabilidade (Traceability Core)
- **Motor de Rastreabilidade:** `Caso de Uso → Requisito Funcional → Regra de Negócio → Caso de Teste → User Story`. *(Agente: AR)*
- **Dicionário de Dados Integrado:** definição e padronização de termos e fatos de negócio.

### Módulo D: Privacy by Design & Conformidade (LGPD)
- **Classificação de Dados:** Dados Pessoais e Dados Pessoais Sensíveis. *(Agente: AC)*
- **Gestão de Bases Legais:** vínculo obrigatório a uma das bases legais da LGPD (Art. 7º/11º).
- **Trilha de Auditoria:** registro imutável de acessos/modificações, sem expor dados sensíveis.
- **Módulo do Titular (DPO Desk):** workflow para Acesso, Correção e Eliminação de dados.

---

## 4. Regras de Negócio Críticas

| Código | Regra | Responsável pela validação |
|---|---|---|
| **RN-LGPD-001** (Coleta Mínima) | Todo RF que solicite coleta de dado pessoal deve estar associado a um Princípio Fundamental da LGPD (Finalidade e Necessidade). | **AC** |
| **RN-REQ-002** (Critério SMART) | Todo critério de aceitação de uma User Story deve passar por validação de formato SMART antes de ser aprovado. | **AUS** |
| **RN-SEC-003** (Segurança de Dados) | Dados classificados como "Sensíveis" devem ser criptografados em repouso e em trânsito. | **AC** + infraestrutura |
| **RN-AUD-004** (Não-Exclusão Física) | Nenhuma entidade rastreável é excluída fisicamente. Toda remoção via API é exclusão lógica (`ativo = false` ou estado terminal). | **AR** + infraestrutura |
| **RN-LGPD-005** (Direito à Eliminação) | Ao concluir uma `solicitacao_titular` do tipo `ELIMINACAO`, descarta-se o mapeamento token↔valor real no vault — nunca o registro em `log_auditoria`. | **AC** + vault de anonimização |
| **RN-PROC-006** (Circuito de Revisão Limitado) | Requisito reprovado pelo AC 3 vezes consecutivas transita para `BLOQUEADO_REVISAO_HUMANA`, suspendendo o pipeline até intervenção manual. | **AC** + **AES** |
| **RN-SEC-007** (Isolamento por Projeto) | Toda consulta a entidades vinculadas a um requisito é filtrada implicitamente por `projeto_id` no nível de serviço. Nenhum endpoint permite acesso cross-projeto a um usuário sem vínculo explícito com o projeto consultado. | infraestrutura + **ADMIN** |

---

## 5. Requisitos Não Funcionais (RNF) — ISO/IEC 25010

| Característica | Requisito aplicado ao ReqGuard |
|---|---|
| **Adequação Funcional** | Cada módulo (A–D) cobre completamente seu escopo funcional (§3), sem depender de processos manuais paralelos para as funções centrais. |
| **Eficiência de Desempenho** | Endpoints de leitura (`GET`) respondem em até 500ms sob carga nominal; endpoints que disparam agentes de IA são assíncronos (`202 Accepted`). |
| **Compatibilidade** | API com contratos versionados (`/api/v1`) e payloads em JSON puro, sem acoplamento ao frontend de referência. |
| **Usabilidade** | Interface responsiva aderente ao WCAG 2.1; mascaramento de dados sensíveis por perfil (RBAC, §6). |
| **Confiabilidade** | Disponibilidade mínima de 99,9%; falhas de chamadas a LLM não corrompem o estado de um requisito (`arquitetura_agentes.md §5`). |
| **Segurança** | RBAC rígido (§6); criptografia em repouso e trânsito (RN-SEC-003); tokenização de PII antes de chamadas a LLM (RN-LGPD-001); isolamento por projeto (RN-SEC-007). |
| **Manutenibilidade** | System prompt de cada agente versionado isoladamente (`prompts_agentes`, `versionamento.md §7`), sem exigir redeploy de código. |
| **Portabilidade** | Backend stateless (FastAPI + JWT); dependência de infraestrutura limitada a PostgreSQL. |

---

## 6. Perfis de Acesso (RBAC)

| Perfil | Módulo A (Stakeholders) | Módulo B (Requisitos) | Módulo C (Rastreabilidade) | Módulo D (LGPD) | Administração |
|---|---|---|---|---|---|
| **ADMIN** | RW | RW | RW | RW | RW — usuários, projetos, prompts dos agentes |
| **ANALISTA_REQUISITOS** | RW | RW | RW | R | — |
| **DPO** | R | R | R | RW | — |
| **PRODUCT_OWNER** | R | RW (backlog, user stories) | R | R | — |
| **STAKEHOLDER_VIEWER** | R | R | R | — | — |

`R` = leitura · `RW` = leitura e escrita · `—` = sem acesso. A exclusão lógica (RN-AUD-004) segue a matriz de escrita. Tabelas e contrato de API em `schema_projeto.md §3–4`.
