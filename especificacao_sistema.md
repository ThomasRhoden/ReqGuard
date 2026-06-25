# 🛠️ Especificação do Sistema: ReqGuard

> **Versão 2.0** — Cadeia de rastreabilidade alinhada com o schema JSON real do AR (campo de caso de teste adicionado), responsabilidade de validação SMART atribuída a um agente, stack definida, conteúdo de currículo removido (ver nota no final).

Este documento especifica o escopo funcional, os módulos, as regras de negócio e os requisitos não funcionais do **ReqGuard** — plataforma para gestão do ciclo de vida de requisitos de software com conformidade LGPD nativa, consolidando os 12 documentos absorvidos na UC1.

---

## 1. Escopo e Objetivos do Sistema

O ReqGuard é uma ferramenta para equipes multidisciplinares (squads) gerenciarem o ciclo de vida de requisitos de software, garantindo que o sistema final esteja em total conformidade com a LGPD e vinculando as necessidades de negócio à arquitetura técnica.

---

## 2. Stack Tecnológica Definida

Para evitar ambiguidade entre documentos, esta é a stack oficial do projeto (justificativa completa em `schema_projeto.md`):

| Camada | Tecnologia |
|---|---|
| Backend | Python 3.12 + FastAPI |
| ORM / Validação | SQLAlchemy 2.0 + Pydantic v2 |
| Banco de dados | **PostgreSQL** (não MySQL/MongoDB — necessário pelo uso nativo de JSONB na trilha de auditoria, ver `arquitetura_agentes.md §3`) |
| Migrations | Alembic |
| Autenticação | JWT + RBAC |
| Frontend (futuro) | React + TypeScript |

---

## 3. Módulos Principais do Sistema

### Módulo A: Gestão de Stakeholders e Comunicação
- **Mapeamento de Impacto:** cadastro e classificação de stakeholders pela Matriz Poder × Interesse.
- **Matriz RACI Automatizada:** atribuição de Responsible/Accountable/Consulted/Informed por entrega ou requisito.
- **Plano de Comunicação:** canais (E-mail, Slack, Jira) definidos pelo perfil do stakeholder.
- *Agente envolvido:* nenhum agente de IA atua diretamente neste módulo — é gestão manual de cadastro.

### Módulo B: Engenharia de Requisitos Eletrônica
- **Central de Elicitação:** registro de transcrições de entrevistas e questionários estruturados. *(Agente: **AER**)*
- **Especificação de RF e RNF:** RF em Linguagem Natural Controlada; RNF com métricas de desempenho/segurança baseadas na ISO/IEC 25010. *(Agente: **AES**)*
- **Módulo de Casos de Uso:** Ator, Fluxo Principal, Alternativos e Exceções, com `<<include>>` e `<<extend>>`.
- **Fábrica de User Stories:** conversão de Casos de Uso em User Story (Como/Quero/Para que) com critérios Gherkin (Dado/Quando/Então). *(Agente: **AUS**)*
- **Priorização de Backlog:** metodologia MoSCoW. *(Agente: **APB**)*

### Módulo C: Rastreabilidade (Traceability Core)
- **Motor de Rastreabilidade:** relacionamento bidirecional ponta a ponta:
  `Caso de Uso → Requisito Funcional → Regra de Negócio → Caso de Teste → User Story`
  *(Agente: **AR**)* — o schema JSON da matriz de rastreabilidade em `arquitetura_agentes.md` inclui explicitamente o campo `caso_de_teste_id` para refletir essa cadeia completa.
- **Dicionário de Dados Integrado:** definição e padronização de termos e fatos de negócio.

### Módulo D: Privacy by Design & Conformidade (LGPD)
- **Classificação de Dados:** marcação de campos com Dados Pessoais e Dados Pessoais Sensíveis. *(Agente: **AC**)*
- **Gestão de Bases Legais:** vínculo obrigatório a uma das 10 bases legais da LGPD.
- **Trilha de Auditoria (Logs):** registro imutável de acessos/modificações, sem expor dados sensíveis.
- **Módulo do Titular (DPO Desk):** workflow para Acesso, Correção e Eliminação de dados.

---

## 4. Regras de Negócio Críticas

| Código | Regra | Responsável pela validação |
|---|---|---|
| **RN-LGPD-001** (Coleta Mínima) | Todo RF que solicite coleta de dado pessoal deve estar associado a um Princípio Fundamental da LGPD (Finalidade e Necessidade). | **AC** |
| **RN-REQ-002** (Critério SMART) | Todo critério de aceitação de uma User Story deve passar por validação de formato SMART (Specific, Measurable, Achievable, Relevant, Testable) antes de ser aprovado. | **AUS** — stories que falharem retornam com estado `PRECISA_REFINAMENTO` |
| **RN-SEC-003** (Segurança de Dados) | Dados classificados como "Sensíveis" devem ser criptografados em repouso e em trânsito. | **AC** + infraestrutura |

> Nas versões anteriores, o RN-REQ-002 não tinha um responsável explícito. Atribuído ao **AUS**, já que é o agente que gera os critérios de aceitação em Gherkin — faz sentido que ele também valide o formato antes de marcar a story como pronta para dev.

---

## 5. Requisitos Não Funcionais (RNF) — ISO/IEC 25010

- **Confiabilidade & Disponibilidade:** disponibilidade mínima de 99,9% (MTBF elevado).
- **Segurança (LGPD):** RBAC rígido e mascaramento de dados sensíveis na interface para usuários sem autorização adequada.
- **Usabilidade:** interface responsiva aderente ao WCAG 2.1.

---

## Nota sobre conteúdo removido

A versão anterior deste documento incluía uma seção de "Como traduzir isso no seu currículo". Removida daqui porque um documento de especificação técnica e um material de carreira têm públicos diferentes — misturar os dois deixa o spec menos confiável como referência técnica. Se quiser, esse conteúdo pode voltar como um documento próprio (ex: `portfolio.md`), separado da documentação técnica do sistema.
