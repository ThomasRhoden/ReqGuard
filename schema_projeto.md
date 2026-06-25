# Schema do Projeto — ReqGuard

| Campo | Valor |
|---|---|
| **Versão do documento** | 2.1 |
| **Pacote documental** | v3.1 |
| **Documentos relacionados** | `README.md`, `arquitetura_agentes.md`, `especificacao_sistema.md`, `versionamento.md` |

Reúne a stack definitiva, a estrutura de pastas, o schema de banco de dados e os contratos de API necessários para desenvolver o backend do ReqGuard.

---

## 1. Stack Tecnológica e Justificativa

| Camada | Tecnologia | Por quê |
|---|---|---|
| Linguagem / Framework | Python 3.12 + FastAPI | Tipagem forte com Pydantic, docs Swagger automáticas. |
| ORM | SQLAlchemy 2.0 (modo async) | Suporta nativamente colunas JSONB do Postgres. |
| Banco de dados: PostgreSQL (não MySQL) | — | Decisão tecnicamente forçada: `arquitetura_agentes.md §4` exige uma trilha de auditoria com hash SHA-256 encadeado sobre colunas JSONB. O tipo `JSONB` binário e indexável do Postgres, junto com a maturidade de suas funções de hashing e constraints, é o que viabiliza essa trilha de forma performática. |
| Migrations | Alembic | Padrão para SQLAlchemy. |
| Autenticação | JWT (python-jose) + RBAC | Consistente com `especificacao_sistema.md` (RN-SEC-003, RBAC §6). |
| Orquestração dos agentes | Chamadas à API Anthropic (ou outra API de LLM), encapsuladas em `services/agentes/` | Cada agente = 1 system prompt (`arquitetura_agentes.md`) + 1 chamada de API + parsing do JSON de saída. |
| Frontend (fora do escopo deste schema) | React + TypeScript | Direção futura — não detalhado aqui. |

---

## 2. Estrutura de Pastas

```
reqguard/
├── app/
│   ├── main.py
│   ├── core/
│   │   ├── config.py          # variáveis de ambiente, settings
│   │   ├── security.py        # JWT, hashing de senha, RBAC
│   │   └── database.py        # engine, sessionmaker async
│   ├── models/                # SQLAlchemy ORM (1 arquivo por entidade)
│   │   ├── usuario.py             # RBAC — autenticação e perfis
│   │   ├── projeto.py             # multi-projeto
│   │   ├── stakeholder.py
│   │   ├── elicitacao.py
│   │   ├── requisito.py
│   │   ├── caso_uso.py
│   │   ├── regra_negocio.py
│   │   ├── caso_teste.py
│   │   ├── user_story.py
│   │   ├── rastreabilidade.py
│   │   ├── conformidade_lgpd.py
│   │   ├── backlog.py
│   │   ├── auditoria.py
│   │   ├── requisito_versao.py    # versionamento.md §4
│   │   └── prompt_agente.py       # versionamento.md §7
│   ├── schemas/                # Pydantic (request/response) — 1 por entidade
│   ├── routers/                # 1 router por módulo
│   │   ├── auth.py
│   │   ├── usuarios.py
│   │   ├── projetos.py
│   │   ├── stakeholders.py
│   │   ├── elicitacoes.py
│   │   ├── requisitos.py
│   │   ├── casos_uso.py
│   │   ├── regras_negocio.py
│   │   ├── casos_teste.py
│   │   ├── dicionario_dados.py
│   │   ├── rastreabilidade.py
│   │   ├── conformidade.py
│   │   ├── solicitacoes_titular.py
│   │   ├── backlog.py
│   │   ├── user_stories.py
│   │   ├── auditoria.py
│   │   └── prompts_agentes.py     # gestão de versões de system prompt (ADMIN)
│   ├── services/
│   │   ├── agentes/             # orquestração de cada agente de IA
│   │   │   ├── aer.py
│   │   │   ├── aes.py
│   │   │   ├── ar.py
│   │   │   ├── ac.py
│   │   │   ├── apb.py
│   │   │   └── aus.py
│   │   ├── anonimizacao.py      # tokenização/regex de PII + vault antes de chamadas LLM
│   │   └── hash_chain.py        # geração e validação do hash SHA-256 encadeado
│   └── utils/
├── alembic/
│   └── versions/
├── sql/
│   └── schema.sql               # DDL completo (seção 3 deste doc)
├── tests/
├── .env.example
└── requirements.txt
```

---

## 3. Schema de Banco de Dados (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ===================================================
-- USUÁRIOS E PROJETOS (RBAC)
-- ===================================================
CREATE TABLE projetos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(50) UNIQUE NOT NULL,       -- ex: 'REQ-GUARD-2026'
    nome VARCHAR(150) NOT NULL,
    descricao TEXT,
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE usuarios (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(150) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    senha_hash VARCHAR(255) NOT NULL,
    perfil VARCHAR(30) NOT NULL
        CHECK (perfil IN ('ADMIN','ANALISTA_REQUISITOS','DPO','PRODUCT_OWNER','STAKEHOLDER_VIEWER')),
    -- matriz completa de permissões por perfil em especificacao_sistema.md §6
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_usuarios_email ON usuarios (email);

-- Nota de isolamento (RN-SEC-007, especificacao_sistema.md §4): toda consulta a
-- entidades vinculadas a um requisito deve filtrar implicitamente por projeto,
-- percorrendo requisitos -> elicitacoes_brutas.projeto_id. Aplicado no nível de
-- serviço (services/), não como Row-Level Security nativo do Postgres nesta fase.

-- ===================================================
-- MÓDULO A: STAKEHOLDERS
-- ===================================================
CREATE TABLE stakeholders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nome VARCHAR(150) NOT NULL,
    papel VARCHAR(100),                     -- Cliente Final, Patrocinador, Usuário...
    poder VARCHAR(20) CHECK (poder IN ('ALTO', 'BAIXO')),
    interesse VARCHAR(20) CHECK (interesse IN ('ALTO', 'BAIXO')),
    canal_comunicacao VARCHAR(50),           -- Email, Slack, Jira
    ativo BOOLEAN NOT NULL DEFAULT true,     -- exclusão lógica — RN-AUD-004
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE matriz_raci (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entrega_id UUID NOT NULL,
    entrega_tipo VARCHAR(30) NOT NULL,       -- 'REQUISITO', 'USER_STORY', etc.
    stakeholder_id UUID NOT NULL REFERENCES stakeholders(id),
    papel_raci CHAR(1) NOT NULL CHECK (papel_raci IN ('R','A','C','I'))
);

-- ===================================================
-- MÓDULO B: ENGENHARIA DE REQUISITOS
-- ===================================================
CREATE TABLE elicitacoes_brutas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projeto_id UUID NOT NULL REFERENCES projetos(id),
    fonte_origem VARCHAR(150),
    stakeholder_origem_id UUID REFERENCES stakeholders(id),
    descricao_bruta TEXT NOT NULL,
    categoria_provavel VARCHAR(100),
    processado_por_aes BOOLEAN NOT NULL DEFAULT false,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_elicitacoes_projeto ON elicitacoes_brutas (projeto_id);

CREATE TABLE requisitos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,      -- RF-PAG-001 / RNF-PERF-001
    tipo VARCHAR(15) NOT NULL CHECK (tipo IN ('FUNCIONAL','NAO_FUNCIONAL')),
    categoria VARCHAR(50),                    -- só RNF: Desempenho, Segurança...
    declaracao TEXT NOT NULL,
    elicitacao_origem_id UUID REFERENCES elicitacoes_brutas(id),
    estado VARCHAR(30) NOT NULL DEFAULT 'ESPECIFICADO',
    -- ESPECIFICADO | EM_RASTREIO | EM_ANALISE_LGPD | REQUER_REVISAO
    -- | BLOQUEADO_REVISAO_HUMANA | APROVADO | PRIORIZADO | DESCARTADO
    tentativas_revisao_lgpd INTEGER NOT NULL DEFAULT 0,
    -- incrementado a cada reprovação do AC vinda do ciclo REQUER_REVISAO
    -- ver arquitetura_agentes.md §2 (RN-PROC-006)
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_requisitos_estado ON requisitos (estado);

CREATE TABLE casos_de_uso (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- UC-PAG-010
    titulo VARCHAR(150) NOT NULL,
    ator VARCHAR(100) NOT NULL,
    fluxo_principal TEXT NOT NULL,
    fluxos_alternativos JSONB,
    fluxos_excecao JSONB,
    relacionamento_include UUID[],
    relacionamento_extend UUID[],
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE regras_negocio (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- RN-FIN-002
    descricao TEXT NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE casos_de_teste (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- CT-PAG-010-01
    requisito_id UUID REFERENCES requisitos(id),
    descricao TEXT NOT NULL,
    resultado_esperado TEXT NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_stories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- US-023
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    titulo VARCHAR(150) NOT NULL,
    narrativa TEXT NOT NULL,
    smart_aprovado BOOLEAN,
    smart_justificativa TEXT,
    estado VARCHAR(30) NOT NULL DEFAULT 'EM_USER_STORY',
    -- EM_USER_STORY | PRECISA_REFINAMENTO | PRONTO_PARA_DEV
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cenarios_gherkin (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_story_id UUID NOT NULL REFERENCES user_stories(id) ON DELETE CASCADE,
    cenario VARCHAR(150) NOT NULL,
    dado TEXT NOT NULL,
    quando TEXT NOT NULL,
    entao TEXT NOT NULL
);

CREATE TABLE backlog_priorizado (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    classificacao_moscow VARCHAR(20) NOT NULL
        CHECK (classificacao_moscow IN ('MUST_HAVE','SHOULD_HAVE','COULD_HAVE','WONT_HAVE')),
    justificativa TEXT,
    classificado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_backlog_requisito ON backlog_priorizado (requisito_id);

-- ===================================================
-- MÓDULO C: RASTREABILIDADE
-- ===================================================
CREATE TABLE matriz_rastreabilidade (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    id_vinculo VARCHAR(20) UNIQUE NOT NULL,   -- VINC-001
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    caso_de_uso_id UUID REFERENCES casos_de_uso(id),
    regra_negocio_id UUID REFERENCES regras_negocio(id),
    caso_de_teste_id UUID REFERENCES casos_de_teste(id),
    user_story_id UUID REFERENCES user_stories(id),
    status VARCHAR(30) NOT NULL DEFAULT 'EM_RASTREIO',
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rastreabilidade_requisito ON matriz_rastreabilidade (requisito_id);

CREATE TABLE dicionario_dados (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    termo VARCHAR(100) UNIQUE NOT NULL,
    definicao TEXT NOT NULL,
    dominio VARCHAR(50),
    ativo BOOLEAN NOT NULL DEFAULT true
);

-- ===================================================
-- MÓDULO D: CONFORMIDADE LGPD
-- ===================================================
CREATE TABLE bases_legais_lgpd (
    id SMALLSERIAL PRIMARY KEY,
    nome VARCHAR(80) NOT NULL,
    artigo_lgpd VARCHAR(30) NOT NULL
);

-- Seed: as 10 hipóteses do Art. 7º da LGPD para dados pessoais comuns.
-- Nota: para Dados Pessoais Sensíveis, o Art. 11º define bases adicionais e mais
-- restritas (ex: consentimento específico e destacado) — não cobertas neste seed
-- inicial. Recomenda-se revisão jurídica formal antes de uso em produção.
INSERT INTO bases_legais_lgpd (nome, artigo_lgpd) VALUES
('Consentimento do titular', 'Art. 7º, I'),
('Cumprimento de obrigação legal ou regulatória', 'Art. 7º, II'),
('Execução de políticas públicas pela administração pública', 'Art. 7º, III'),
('Realização de estudos por órgão de pesquisa', 'Art. 7º, IV'),
('Execução de contrato ou procedimentos preliminares', 'Art. 7º, V'),
('Exercício regular de direitos em processo judicial/administrativo/arbitral', 'Art. 7º, VI'),
('Proteção da vida ou da incolumidade física', 'Art. 7º, VII'),
('Tutela da saúde, em procedimento por profissional de saúde', 'Art. 7º, VIII'),
('Interesse legítimo do controlador ou de terceiro', 'Art. 7º, IX'),
('Proteção do crédito', 'Art. 7º, X');

CREATE TABLE conformidade_lgpd (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    coleta_dados_pessoais BOOLEAN NOT NULL DEFAULT false,
    dados_identificados TEXT[],
    classificacao VARCHAR(30)
        CHECK (classificacao IN ('DADO_PESSOAL_COMUM','DADO_PESSOAL_SENSIVEL','NAO_APLICAVEL')),
    base_legal_id SMALLINT REFERENCES bases_legais_lgpd(id),
    principio_aplicado VARCHAR(80),
    status_aprovacao VARCHAR(20) NOT NULL DEFAULT 'EM_ANALISE'
        CHECK (status_aprovacao IN ('EM_ANALISE','APROVADO','REQUER_REVISAO')),
    analisado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_conformidade_requisito ON conformidade_lgpd (requisito_id);

CREATE TABLE solicitacoes_titular (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tipo VARCHAR(20) NOT NULL CHECK (tipo IN ('ACESSO','CORRECAO','ELIMINACAO')),
    titular_identificacao VARCHAR(150) NOT NULL,  -- sempre tokenizado/anonimizado em logs
    status VARCHAR(20) NOT NULL DEFAULT 'ABERTA',
    aberto_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    concluido_em TIMESTAMPTZ
);

-- ===================================================
-- TRILHA DE AUDITORIA IMUTÁVEL (HASH CHAIN)
-- ===================================================
CREATE TABLE log_auditoria (
    id BIGSERIAL PRIMARY KEY,
    agente_origem VARCHAR(10) NOT NULL,        -- AER, AES, AR, AC, APB, AUS
    entidade_tipo VARCHAR(30) NOT NULL,
    entidade_id UUID NOT NULL,
    payload JSONB NOT NULL,
    -- payload NUNCA contém dado pessoal bruto re-identificável — apenas tokens
    -- gerados por services/anonimizacao.py. Ver arquitetura_agentes.md §4.
    hash_anterior CHAR(64),
    hash_atual CHAR(64) NOT NULL,
    prompt_versao VARCHAR(10),
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_log_auditoria_entidade ON log_auditoria (entidade_tipo, entidade_id);

-- ===================================================
-- VERSIONAMENTO (ver versionamento.md)
-- ===================================================
CREATE TABLE requisitos_versoes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requisito_id UUID NOT NULL REFERENCES requisitos(id),
    numero_versao INTEGER NOT NULL,
    estado_anterior VARCHAR(30),
    estado_novo VARCHAR(30) NOT NULL,
    snapshot JSONB NOT NULL,
    alterado_por_agente VARCHAR(10),   -- AER, AES, AR, AC, APB, AUS ou 'MANUAL'
    motivo_alteracao TEXT,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (requisito_id, numero_versao)
);

CREATE TABLE prompts_agentes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agente VARCHAR(10) NOT NULL,
    versao VARCHAR(10) NOT NULL,
    system_prompt TEXT NOT NULL,
    ativo BOOLEAN NOT NULL DEFAULT true,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (agente, versao)
);
```

> **Convenção de exclusão (RN-AUD-004):** nenhuma tabela deste schema usa `DELETE` físico para as entidades de negócio. A coluna `ativo` (ou, quando já existe uma máquina de estados própria como em `requisitos`/`user_stories`, um estado terminal como `DESCARTADO`) é o mecanismo de remoção lógica. `log_auditoria`, `requisitos_versoes` e `prompts_agentes` são, por natureza, *append-only* — nunca recebem update nem delete.

---

## 4. Contratos de API

| Método | Rota | Descrição | Agente disparado |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Autenticação (JWT) | — |
| `POST` | `/api/v1/usuarios` | Cria usuário (perfil `ADMIN`) | — |
| `GET` | `/api/v1/usuarios` | Lista usuários (perfil `ADMIN`) | — |
| `POST` | `/api/v1/projetos` | Cadastra projeto | — |
| `GET` | `/api/v1/projetos` | Lista projetos | — |
| `POST` | `/api/v1/stakeholders` | Cadastra stakeholder | — |
| `DELETE` | `/api/v1/stakeholders/{id}` | Exclusão lógica (`ativo = false`) | — |
| `POST` | `/api/v1/elicitacoes` | Registra transcrição/ata bruta — ponto de entrada da pipeline | AER |
| `GET` | `/api/v1/elicitacoes/{id}` | Consulta elemento elicitado | — |
| `POST` | `/api/v1/elicitacoes/{id}/especificar` | Dispara especificação do requisito | AES |
| `GET` | `/api/v1/requisitos` | Lista requisitos (filtros: `tipo`, `estado`) | — |
| `GET` | `/api/v1/requisitos/{id}` | Detalhe do requisito | — |
| `GET` | `/api/v1/requisitos/{id}/historico` | Linha do tempo de versões (`requisitos_versoes`) do requisito | — |
| `PATCH` | `/api/v1/requisitos/{id}` | Edição manual de um requisito | — |
| `POST` | `/api/v1/requisitos/{id}/rastrear` | Monta a matriz de rastreabilidade | AR |
| `GET` | `/api/v1/rastreabilidade/{requisito_id}` | Retorna cadeia completa rastreada | — |
| `POST`/`GET` | `/api/v1/casos-uso` | Cadastra / lista casos de uso | — |
| `POST`/`GET` | `/api/v1/regras-negocio` | Cadastra / lista regras de negócio | — |
| `POST`/`GET` | `/api/v1/casos-teste` | Cadastra / lista casos de teste | — |
| `POST`/`GET` | `/api/v1/dicionario-dados` | Cadastra / lista termos do dicionário | — |
| `POST` | `/api/v1/conformidade/analisar/{requisito_id}` | Dispara análise LGPD | AC |
| `GET` | `/api/v1/conformidade/{requisito_id}` | Consulta status de conformidade | — |
| `POST` | `/api/v1/backlog/priorizar` | Prioriza requisitos aprovados (MoSCoW) | APB |
| `GET` | `/api/v1/backlog` | Lista backlog priorizado | — |
| `POST` | `/api/v1/user-stories/gerar/{requisito_id}` | Gera user story + critérios Gherkin | AUS |
| `GET` | `/api/v1/user-stories/{id}` | Detalhe da user story | — |
| `POST` | `/api/v1/solicitacoes-titular` | Abre solicitação do titular (Acesso/Correção/Eliminação) | — |
| `GET` | `/api/v1/solicitacoes-titular` | Lista solicitações do titular (perfil `DPO`) | — |
| `PATCH` | `/api/v1/solicitacoes-titular/{id}` | Atualiza status / conclui solicitação | AC (para `ELIMINACAO`) |
| `GET` | `/api/v1/auditoria/{entidade_tipo}/{entidade_id}` | Retorna a cadeia de hash de auditoria de um registro | — |
| `GET` | `/api/v1/agentes/prompts` | Lista versões de system prompt por agente (perfil `ADMIN`) | — |
| `POST` | `/api/v1/agentes/prompts` | Cria nova versão de prompt (marca anterior `ativo = false`) | — |

> Todo verbo `DELETE` acima (e nas rotas equivalentes de `casos-uso`, `regras-negocio`, `casos-teste`, `dicionario-dados`) implementa exclusão lógica, nunca `DELETE` físico.
> Endpoints que disparam um agente de IA retornam `202 Accepted` com um identificador de acompanhamento — são assíncronos por dependerem de latência de API externa.

### Exemplo — `POST /api/v1/elicitacoes`

**Request:**
```json
{
  "projeto_id": "uuid-do-projeto",
  "fonte_origem": "Entrevista com Stakeholder Financiador",
  "descricao_bruta": "O usuário quer conseguir pagar usando Pix e receber a confirmação na tela em menos de 3 segundos.",
  "stakeholder_origem_id": "uuid-do-stakeholder"
}
```

**Response (201):** schema `elementos_elicitados` do AER (ver `arquitetura_agentes.md §3.1`).

### Exemplo — `POST /api/v1/conformidade/analisar/{requisito_id}`

**Response (200):** schema `analise_conformidade_lgpd` do AC (ver `arquitetura_agentes.md §3.4`).

---

## 5. Próximos Passos para Começar a Desenvolver

1. Subir PostgreSQL local (Docker é o mais simples) e rodar `sql/schema.sql`, ou configurar Alembic e gerar a primeira migration a partir dele.
2. Criar o ambiente Python (venv, `requirements.txt`: fastapi, sqlalchemy[asyncio], asyncpg, alembic, pydantic, python-jose).
3. Resolver o *bootstrap* de acesso: criar o primeiro usuário `ADMIN` via seed/script de setup, já que `POST /usuarios` por padrão exige um `ADMIN` autenticado para criar outros usuários.
4. Implementar primeiro o módulo de Elicitação (`POST /elicitacoes`) — é o ponto de entrada de toda a pipeline.
5. Implementar `services/anonimizacao.py` (incluindo o vault de tokens) antes de qualquer chamada real a uma API de LLM.
6. Implementar cada agente em `services/agentes/` como uma função que monta o system prompt, chama a API, trata falhas/retries (`arquitetura_agentes.md §5`) e faz o parsing do JSON de saída para o schema Pydantic correspondente.
7. Implementar `services/hash_chain.py` e testar a trilha de auditoria com 2-3 registros fake antes de plugar nos agentes reais.
8. Seguir a ordem da pipeline implementando rota por rota: AER → AES → AR/AC → APB → AUS.
