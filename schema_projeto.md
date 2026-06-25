# 🧱 Schema do Projeto — ReqGuard

> **Versão 1.1** — Reúne a stack definitiva, a estrutura de pastas, o schema de banco de dados e os contratos de API necessários para começar a desenvolver o backend do ReqGuard. Atualizado para incluir as tabelas `requisitos_versoes` e `prompts_agentes` e a coluna `log_auditoria.prompt_versao`, definidas em `versionamento.md`.

---

## 1. Stack Tecnológica e Justificativa

| Camada | Tecnologia | Por quê |
|---|---|---|
| Linguagem / Framework | **Python 3.12 + FastAPI** | Mesma stack do seu projeto Início Responsável — curva de aprendizado reaproveitada, tipagem forte com Pydantic, docs Swagger automáticas. |
| ORM | **SQLAlchemy 2.0** (modo async) | Já é o que você está usando; suporta nativamente colunas JSONB do Postgres. |
| **Banco de dados: PostgreSQL** (não MySQL) | — | Esta é a decisão tecnicamente forçada do projeto: `arquitetura_agentes.md §4` exige uma trilha de auditoria com **hash SHA-256 encadeado sobre colunas JSONB**. O MySQL tem suporte a JSON, mas o tipo `JSONB` binário e indexável — junto com a maturidade de funções de hashing e constraints do Postgres — é o que viabiliza essa trilha de auditoria de forma performática. Usar Postgres aqui também é uma boa adição ao portfólio: mostra que você não fica restrito a uma única stack. |
| Migrations | **Alembic** | Padrão para SQLAlchemy. |
| Autenticação | **JWT (python-jose) + RBAC** | Consistente com o que já está em `especificacao_sistema.md` (RN-SEC-003, RBAC). |
| Orquestração dos agentes | Chamadas à **API Anthropic** (ou outra API de LLM), encapsuladas em `services/agentes/` | Cada agente = 1 system prompt (já definido em `arquitetura_agentes.md`) + 1 chamada de API + parsing do JSON de saída. |
| Frontend (fora do escopo deste schema) | React + TypeScript | Mencionado como direção futura — não detalhado aqui. |

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
│   │   ├── stakeholders.py
│   │   ├── elicitacoes.py
│   │   ├── requisitos.py
│   │   ├── casos_uso.py
│   │   ├── rastreabilidade.py
│   │   ├── conformidade.py
│   │   ├── backlog.py
│   │   ├── user_stories.py
│   │   └── auditoria.py
│   ├── services/
│   │   ├── agentes/             # orquestração de cada agente de IA
│   │   │   ├── aer.py
│   │   │   ├── aes.py
│   │   │   ├── ar.py
│   │   │   ├── ac.py
│   │   │   ├── apb.py
│   │   │   └── aus.py
│   │   ├── anonimizacao.py      # tokenização/regex de PII antes de chamadas LLM
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

> Estrutura intencionalmente parecida com a do Início Responsável (`app/{main.py, database.py, models/, schemas/, routers/, services/}`), só que com `services/agentes/` adicionado para a orquestração de IA.

---

## 3. Schema de Banco de Dados (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

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
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
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
    projeto_id VARCHAR(50) NOT NULL,
    fonte_origem VARCHAR(150),
    stakeholder_origem_id UUID REFERENCES stakeholders(id),
    descricao_bruta TEXT NOT NULL,
    categoria_provavel VARCHAR(100),
    processado_por_aes BOOLEAN NOT NULL DEFAULT false,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE requisitos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,      -- RF-PAG-001 / RNF-PERF-001
    tipo VARCHAR(15) NOT NULL CHECK (tipo IN ('FUNCIONAL','NAO_FUNCIONAL')),
    categoria VARCHAR(50),                    -- só RNF: Desempenho, Segurança...
    declaracao TEXT NOT NULL,
    elicitacao_origem_id UUID REFERENCES elicitacoes_brutas(id),
    estado VARCHAR(30) NOT NULL DEFAULT 'ESPECIFICADO',
    -- ESPECIFICADO | EM_RASTREIO | EM_ANALISE_LGPD | REQUER_REVISAO
    -- | APROVADO | PRIORIZADO | DESCARTADO
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE regras_negocio (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- RN-FIN-002
    descricao TEXT NOT NULL,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE casos_de_teste (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    codigo VARCHAR(20) UNIQUE NOT NULL,       -- CT-PAG-010-01
    requisito_id UUID REFERENCES requisitos(id),
    descricao TEXT NOT NULL,
    resultado_esperado TEXT NOT NULL,
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
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
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
    classificado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    status VARCHAR(30) NOT NULL DEFAULT 'EM_RASTREIO'
);

CREATE TABLE dicionario_dados (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    termo VARCHAR(100) UNIQUE NOT NULL,
    definicao TEXT NOT NULL,
    dominio VARCHAR(50)
);

-- ===================================================
-- MÓDULO D: CONFORMIDADE LGPD
-- ===================================================
CREATE TABLE bases_legais_lgpd (
    id SMALLSERIAL PRIMARY KEY,
    nome VARCHAR(80) NOT NULL,
    artigo_lgpd VARCHAR(30) NOT NULL
);
-- seed: as 10 bases legais dos Art. 7º / 11º da LGPD

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
    analisado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);

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
    hash_anterior CHAR(64),
    hash_atual CHAR(64) NOT NULL,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_log_auditoria_entidade ON log_auditoria (entidade_tipo, entidade_id);

-- ===================================================
-- VERSIONAMENTO (ver versionamento.md)
-- ===================================================
ALTER TABLE log_auditoria ADD COLUMN prompt_versao VARCHAR(10);

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

---

## 4. Contratos de API

| Método | Rota | Descrição | Agente disparado |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Autenticação (JWT) | — |
| `POST` | `/api/v1/stakeholders` | Cadastra stakeholder | — |
| `POST` | `/api/v1/elicitacoes` | Registra transcrição/ata bruta — ponto de entrada da pipeline | **AER** |
| `GET` | `/api/v1/elicitacoes/{id}` | Consulta elemento elicitado | — |
| `POST` | `/api/v1/elicitacoes/{id}/especificar` | Dispara especificação do requisito | **AES** |
| `GET` | `/api/v1/requisitos` | Lista requisitos (filtros: `tipo`, `estado`) | — |
| `GET` | `/api/v1/requisitos/{id}` | Detalhe do requisito | — |
| `PATCH` | `/api/v1/requisitos/{id}` | Edição manual de um requisito | — |
| `POST` | `/api/v1/requisitos/{id}/rastrear` | Monta a matriz de rastreabilidade | **AR** |
| `GET` | `/api/v1/rastreabilidade/{requisito_id}` | Retorna cadeia completa rastreada | — |
| `POST` | `/api/v1/conformidade/analisar/{requisito_id}` | Dispara análise LGPD | **AC** |
| `GET` | `/api/v1/conformidade/{requisito_id}` | Consulta status de conformidade | — |
| `POST` | `/api/v1/backlog/priorizar` | Prioriza requisitos aprovados (MoSCoW) | **APB** |
| `GET` | `/api/v1/backlog` | Lista backlog priorizado | — |
| `POST` | `/api/v1/user-stories/gerar/{requisito_id}` | Gera user story + critérios Gherkin | **AUS** |
| `GET` | `/api/v1/user-stories/{id}` | Detalhe da user story | — |
| `POST` | `/api/v1/solicitacoes-titular` | Abre solicitação do titular (Acesso/Correção/Eliminação) | — |
| `GET` | `/api/v1/auditoria/{entidade_tipo}/{entidade_id}` | Retorna a cadeia de hash de auditoria de um registro | — |

### Exemplo — `POST /api/v1/elicitacoes`

**Request:**
```json
{
  "projeto_id": "REQ-GUARD-2026",
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
3. Implementar primeiro o módulo de **Elicitação** (`POST /elicitacoes`) — é o ponto de entrada de toda a pipeline.
4. Implementar `services/anonimizacao.py` **antes** de qualquer chamada real a uma API de LLM — privacy by design desde o dia 1, não depois.
5. Implementar cada agente em `services/agentes/` como uma função que monta o system prompt (de `arquitetura_agentes.md`), chama a API e faz o parsing do JSON de saída para o schema Pydantic correspondente.
6. Implementar `services/hash_chain.py` e testar a trilha de auditoria com 2-3 registros fake antes de plugar nos agentes reais.
7. Seguir a ordem da pipeline implementando rota por rota: AER → AES → AR/AC → APB → AUS.
