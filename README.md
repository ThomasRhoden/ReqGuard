# 🛡️ ReqGuard — Plataforma de Gestão de Requisitos e Conformidade

> **Versão 2.0** — Documento revisado para corrigir colisão de siglas entre agentes, fluxo de comunicação divergente do diagrama oficial e links internos quebrados.

Projeto conceitual desenvolvido a partir do conteúdo absorvido na **UC1 do curso Técnico em Desenvolvimento de Sistemas (SENAC)**, usado como projeto de portfólio para demonstrar Engenharia de Requisitos, Modelagem de Casos de Uso/User Stories (BDD), Rastreabilidade e *Privacy by Design* aplicados a um sistema real, orquestrado por agentes de IA.

O **ReqGuard** ajuda equipes multidisciplinares a gerenciar o ciclo de vida de requisitos de software, do levantamento com stakeholders até o backlog priorizado, garantindo conformidade com a **LGPD (Lei nº 13.709/2018)** em cada etapa.

---

## 📚 Documentação do Projeto

| Documento | Conteúdo |
|---|---|
| [`especificacao_sistema.md`](./especificacao_sistema.md) | Escopo, módulos do sistema, regras de negócio e requisitos não funcionais (ISO/IEC 25010) |
| [`arquitetura_agentes.md`](./arquitetura_agentes.md) | Especificação completa dos 6 agentes de IA, diagrama de fluxo, máquina de estados e protocolo de segurança |
| [`schema_projeto.md`](./schema_projeto.md) | Stack definitiva, estrutura de pastas, schema de banco (PostgreSQL) e contratos de API — ponto de partida para codar |

---

## 🎯 Visão e Objetivo

Consolidar, num único sistema, as práticas de Engenharia de Requisitos (IEEE 830 / ISO 29148), vinculando necessidades de negócio à arquitetura técnica e à conformidade regulatória — sem depender de planilhas soltas ou documentos desconectados.

---

## 🧩 Módulos do Sistema (resumo)

- **A — Gestão de Stakeholders:** mapeamento Poder × Interesse, matriz RACI, plano de comunicação.
- **B — Engenharia de Requisitos:** elicitação, especificação de RF/RNF, casos de uso, user stories (Gherkin), priorização MoSCoW.
- **C — Rastreabilidade:** vínculo bidirecional Caso de Uso ↔ Requisito ↔ Regra de Negócio ↔ Caso de Teste ↔ User Story.
- **D — Privacy by Design / LGPD:** classificação de dados, bases legais, trilha de auditoria imutável, módulo do titular.

Detalhes completos em [`especificacao_sistema.md`](./especificacao_sistema.md).

---

## 🤖 Agentes de Inteligência Artificial

| Sigla | Nome | Responsabilidade principal |
|---|---|---|
| **AER** | Agente de Elicitação de Requisitos | Processa transcrições/atas e extrai elementos brutos de necessidade |
| **AES** | Agente de Especificação de Requisitos | Transforma elementos brutos em RF/RNF estruturados |
| **AR** | Agente de Rastreabilidade | Vincula requisito ↔ caso de uso ↔ regra de negócio ↔ caso de teste ↔ user story |
| **AC** | Agente de Conformidade (LGPD) | Classifica dados pessoais e valida base legal de cada requisito |
| **APB** | Agente de Priorização de Backlog | Classifica requisitos aprovados na matriz MoSCoW |
| **AUS** | Agente de User Stories | Gera user stories e critérios de aceitação em Gherkin, valida formato SMART |

> ⚠️ Nas versões anteriores deste documento, o agente de especificação aparecia incorretamente com a mesma sigla do agente de elicitação (ambos como "AER"). Corrigido para **AES**.

---

## 🔄 Fluxo de Comunicação entre Agentes

```
Stakeholder → AER → AES → (AR ⇄ AC, validação cruzada) → APB → AUS → Repositório / Jira
```

O fluxo **não é um ciclo fechado**: ele nasce no levantamento com o stakeholder e termina na entrega ao repositório/Jira, pronto para desenvolvimento. O único ponto não-linear é entre **AR e AC**, que se validam mutuamente antes de liberar o requisito para priorização — se o AC reprovar por falta de base legal, o requisito retorna para revisão em vez de seguir adiante.

Diagrama completo, system prompts e schemas JSON de cada agente em [`arquitetura_agentes.md`](./arquitetura_agentes.md).

---

## 🔐 Segurança e Privacidade (resumo)

- Qualquer dado potencialmente real de usuário final passa por tokenização/anonimização **antes** de ser enviado a uma API de LLM.
- Toda a cadeia de mensagens entre agentes é armazenada com hash SHA-256 encadeado (estilo blockchain), tornando a trilha de auditoria imutável.
- RBAC e mascaramento de dados sensíveis na interface para usuários sem nível de autorização adequado.
- Criptografia em repouso e em trânsito para dados classificados como sensíveis.

Protocolo técnico completo em [`arquitetura_agentes.md` §3](./arquitetura_agentes.md).

---

## 🛣️ Roadmap / Escopo de Melhorias Futuras

- Sistema de notificações para mudanças em requisitos ou conformidade.
- Módulo de análise de impacto (o que muda na LGPD/rastreabilidade quando um requisito muda).
- Integração com Jira/Trello para sincronização de backlog.
- Dashboard em tempo real (status de requisitos, conformidade, rastreabilidade).
- Suporte a múltiplos idiomas.
- Sugestões de melhoria de requisitos via aprendizado de máquina sobre histórico do próprio projeto.
- Versionamento completo de requisitos para auditoria histórica.

---

## 🧱 Stack Tecnológica

**Backend:** Python 3.12 + FastAPI + SQLAlchemy 2.0 + Pydantic v2
**Banco de dados:** PostgreSQL (suporte nativo a JSONB, necessário para a trilha de auditoria com hash encadeado)
**Migrations:** Alembic · **Auth:** JWT + RBAC

Justificativa da escolha, estrutura de pastas completa e schema de banco em [`schema_projeto.md`](./schema_projeto.md).

---

## ✍️ Sobre este projeto

- **Copiloto:** Thomas Rhoden Gonçalves
- **Piloto:** GitHub Copilot
- **Demais agentes de IA consultados no processo:** ChatGPT, Claude, Gemini, Mistral, LLaMA

Projeto conceitual construído como exercício prático de Engenharia de Requisitos + IA aplicada, no contexto da formação Técnico em Desenvolvimento de Sistemas (SENAC).
