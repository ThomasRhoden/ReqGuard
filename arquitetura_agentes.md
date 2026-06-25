# Arquitetura de Agentes de IA — ReqGuard

| Campo | Valor |
|---|---|
| **Versão do documento** | 2.2 |
| **Pacote documental** | v3.1 |
| **Documentos relacionados** | `README.md`, `especificacao_sistema.md`, `schema_projeto.md`, `versionamento.md` |

Este documento especifica a engenharia de prompts, os esquemas de dados (JSON Payloads) e as regras de transição de estado para os 6 agentes de IA integrados na plataforma ReqGuard.

---

## 1. Diagrama de Fluxo entre Agentes

```mermaid
flowchart TD
    ST[Stakeholder] -->|Transcrição / Ata| AER[AER<br/>Elicitação]
    AER -->|JSON: Dados Brutos Elicitados| AES[AES<br/>Especificação]
    AES -->|JSON: RF / RNF Estruturados| AR[AR<br/>Rastreabilidade]
    AR <-->|Validação Cruzada| AC[AC<br/>Conformidade LGPD]
    AR -->|Requisito rastreado e aprovado| APB[APB<br/>Priorização MoSCoW]
    APB -->|Backlog Priorizado| AUS[AUS<br/>User Stories]
    AUS -->|Gherkin Scenarios| REPO[(Repositório / Jira)]
```

O fluxo é majoritariamente linear (AER → AES → AR → APB → AUS), com uma única exceção não-linear: AR e AC se validam mutuamente antes de liberar o requisito para priorização. Se o AC reprovar (ex: dado pessoal sem base legal vinculada), o requisito retorna ao AES com status `REQUER_REVISAO`. Esse ciclo é limitado: na 3ª reprovação consecutiva, o requisito sai do circuito automático e aguarda decisão humana (§2, RN-PROC-006).

---

## 2. Máquina de Estados de um Requisito

| Estado | Descrição | Agente responsável | Transições possíveis |
|---|---|---|---|
| `BRUTO` | Elemento recém-elicitado, ainda não estruturado | AER | → `ESPECIFICADO` |
| `ESPECIFICADO` | RF/RNF redigido e estruturado | AES | → `EM_RASTREIO` |
| `EM_RASTREIO` | Vínculos com caso de uso, regra de negócio e caso de teste sendo montados | AR | → `EM_ANALISE_LGPD` |
| `EM_ANALISE_LGPD` | Em validação cruzada de conformidade | AC | → `APROVADO` ou `REQUER_REVISAO` |
| `REQUER_REVISAO` | Reprovado por falta de base legal ou dado não classificado | AC | → `ESPECIFICADO` — ou → `BLOQUEADO_REVISAO_HUMANA` na 3ª reprovação consecutiva |
| `BLOQUEADO_REVISAO_HUMANA` | Reprovado pelo AC 3 vezes consecutivas; pipeline automático suspenso | AC (detecta) + `DPO`/`ADMIN` (resolve) | → `ESPECIFICADO` (revisão manual) ou `DESCARTADO` |
| `APROVADO` | Conforme à LGPD e rastreado ponta a ponta | AR + AC | → `PRIORIZADO` |
| `PRIORIZADO` | Classificado na matriz MoSCoW | APB | → `EM_USER_STORY` ou `DESCARTADO` |
| `DESCARTADO` | Won't have, ou descartado manualmente após bloqueio | APB / manual | *(estado final)* |
| `EM_USER_STORY` | User story e critérios Gherkin sendo gerados | AUS | → `PRONTO_PARA_DEV` ou `PRECISA_REFINAMENTO` |
| `PRECISA_REFINAMENTO` | Critério de aceitação não passou validação SMART (RN-REQ-002) | AUS | → `EM_USER_STORY` |
| `PRONTO_PARA_DEV` | Enviado ao repositório/Jira | AUS | *(estado final)* |

**Circuito de revisão limitado (RN-PROC-006):** a coluna `requisitos.tentativas_revisao_lgpd` (`schema_projeto.md §3`) é incrementada a cada reprovação do AC vinda do ciclo `REQUER_REVISAO`. Ao atingir 3, o requisito transita automaticamente para `BLOQUEADO_REVISAO_HUMANA` e o pipeline para até um usuário com perfil `DPO` ou `ADMIN` decidir manualmente.

---

## 3. Especificação Técnica dos Agentes

### 3.1 AER — Agente de Elicitação de Requisitos

**System Prompt:**
> "Atue como um Engenheiro de Requisitos Sênior especializado em técnicas de elicitação ativa. Seu objetivo é processar transcrições de reuniões, atas, e-mails ou respostas de formulários brutos. Extraia restrições de negócio, dores do usuário, fluxos operacionais e desejos do cliente sem omitir nuances técnicas. Normalize jargões corporativos em declarações de necessidades de negócio puras."

**Disparado quando:** uma nova transcrição/ata é registrada no sistema (`POST /elicitacoes`).

**Entrada:** texto bruto (transcrição, ata, e-mail, formulário).

**Saída (JSON Schema):**
```json
{
  "projeto_id": "uuid-do-projeto",
  "fonte_origem": "Entrevista com Stakeholder Financiador",
  "elementos_elicitados": [
    {
      "id_bruto": "ELIC-001",
      "descricao_bruta": "O usuário quer conseguir pagar usando Pix e receber a confirmação na tela em menos de 3 segundos.",
      "stakeholder_origem": "Cliente Final",
      "categoria_provavel": "Funcional / Desempenho"
    }
  ]
}
```

---

### 3.2 AES — Agente de Especificação de Requisitos

**System Prompt:**
> "Atue como um Analista de Requisitos especializado nas normas IEEE 830 / ISO/IEC/IEEE 29148. Transforme os elementos brutos elicitados pelo AER em Requisitos Funcionais (RF) e Não Funcionais (RNF) redigidos em Linguagem Natural Controlada, classificando RNFs conforme as características da ISO/IEC 25010 quando aplicável. Mantenha rastreabilidade explícita de cada requisito gerado com seu(s) elemento(s) bruto(s) de origem. Não infira requisitos que não estejam implícitos ou explícitos no input recebido. Quando o input vier de um ciclo `REQUER_REVISAO`, reescreva o requisito considerando explicitamente a justificativa de reprovação do AC fornecida no contexto."

**Disparado quando:** o AER finaliza o processamento de uma elicitação — ou o AC reprova um requisito e o devolve para revisão.

**Entrada:** JSON de saída do AER — ou, no ciclo de revisão, o requisito atual + a justificativa de reprovação do AC.

**Saída (JSON Schema):**
```json
{
  "requisitos_especificados": {
    "funcionais": [
      {
        "id": "RF-PAG-001",
        "declaracao": "O sistema deve processar pagamentos via arranjo Pix e exibir a confirmação de recebimento na interface do usuário.",
        "rastreabilidade_origem": ["ELIC-001"],
        "status": "ESPECIFICADO"
      }
    ],
    "nao_funcionais": [
      {
        "id": "RNF-PERF-001",
        "categoria": "Desempenho",
        "subcategoria": "Tempo de resposta",
        "declaracao": "O tempo de resposta do processamento do Pix deve ser inferior a 3 segundos para 95% das requisições sob carga nominal.",
        "rastreabilidade_origem": ["ELIC-001"],
        "status": "ESPECIFICADO"
      }
    ]
  }
}
```

---

### 3.3 AR — Agente de Rastreabilidade

**System Prompt:**
> "Atue como um Engenheiro de Configuração e Rastreabilidade. Vincule cada requisito funcional/não funcional aos seus artefatos relacionados — caso de uso, regra de negócio, caso de teste e user story — construindo e mantendo a matriz de rastreabilidade bidirecional ponta a ponta. Sinalize como `PENDENTE` qualquer requisito que não tenha um artefato vinculado em alguma das pontas da cadeia. Encaminhe o resultado ao AC para validação cruzada de conformidade antes de liberar o requisito para priorização."

**Disparado quando:** o AES gera um novo requisito especificado.

**Entrada:** JSON de saída do AES + IDs de casos de uso/regras de negócio/casos de teste já cadastrados.

**Saída (JSON Schema):**
```json
{
  "matriz_rastreabilidade": [
    {
      "id_vinculo": "VINC-001",
      "requisito_id": "RF-PAG-001",
      "caso_de_uso_id": "UC-PAG-010",
      "regra_negocio_id": "RN-FIN-002",
      "caso_de_teste_id": "CT-PAG-010-01",
      "user_story_id": "US-023",
      "status": "EM_ANALISE_LGPD"
    }
  ]
}
```

---

### 3.4 AC — Agente de Conformidade (LGPD)

**System Prompt:**
> "Atue como um Encarregado de Proteção de Dados (DPO) assistente, especializado na Lei nº 13.709/2018 (LGPD). Analise cada requisito que envolva coleta, tratamento ou exposição de dados pessoais: identifique os dados envolvidos, classifique-os (Dado Pessoal Comum ou Dado Pessoal Sensível) e vincule obrigatoriamente uma base legal válida (Art. 7º ou Art. 11º da LGPD) e o princípio de Finalidade e Necessidade. Requisitos que coletem dados pessoais sem base legal vinculada devem ser reprovados e retornados ao AES com status `REQUER_REVISAO`, com uma justificativa específica e acionável do que falta."

**Disparado quando:** o AR encaminha um requisito rastreado para validação cruzada.

**Entrada:** JSON de saída do AR + declaração completa do requisito (AES).

**Saída (JSON Schema):**
```json
{
  "analise_conformidade_lgpd": [
    {
      "requisito_id": "RF-PAG-001",
      "coleta_dados_pessoais": true,
      "dados_identificados": ["Chave Pix", "Nome do Pagador"],
      "classificacao": "Dado Pessoal Comum",
      "base_legal_proposta": "Execução de Contrato (Art. 7º, V, LGPD)",
      "principio_aplicado": "Necessidade e Finalidade",
      "status_aprovacao": "APROVADO",
      "encaminhado_para": "APB"
    }
  ]
}
```

> Quando `status_aprovacao` é `REQUER_REVISAO`, o agente incrementa `requisitos.tentativas_revisao_lgpd` antes de retornar — esse contador alimenta o circuito de revisão limitado (RN-PROC-006, §2).

---

### 3.5 APB — Agente de Priorização de Backlog

**System Prompt:**
> "Atue como um Product Owner experiente em priorização ágil. Classifique cada requisito aprovado (rastreado pelo AR e conforme à LGPD pelo AC) na matriz MoSCoW (Must have, Should have, Could have, Won't have), considerando valor de negócio, urgência declarada pelo stakeholder e dependências técnicas mapeadas pelo AR. Nunca classifique um requisito que esteja com status `REQUER_REVISAO` ou `BLOQUEADO_REVISAO_HUMANA` — ele deve permanecer fora do backlog até resolução."

**Disparado quando:** o AC aprova um requisito (`status_aprovacao: APROVADO`).

**Entrada:** JSON de saída do AC + matriz de rastreabilidade (AR).

**Saída (JSON Schema):**
```json
{
  "backlog_priorizado": {
    "must_have": ["RF-PAG-001", "RNF-PERF-001"],
    "should_have": [],
    "could_have": [],
    "wont_have": []
  }
}
```

---

### 3.6 AUS — Agente de User Stories

**System Prompt:**
> "Atue como um especialista em BDD (Behavior-Driven Development). Converta requisitos funcionais priorizados em User Stories no formato Como/Quero/Para que, com critérios de aceitação em Gherkin (Dado/Quando/Então). Antes de marcar uma story como `PRONTO_PARA_DEV`, valide cada critério de aceitação contra o formato SMART (Specific, Measurable, Achievable, Relevant, Testable — RN-REQ-002). Stories que não atendam ao SMART devem retornar com status `PRECISA_REFINAMENTO` e a justificativa do que falhou."

**Disparado quando:** o APB classifica um requisito no backlog priorizado.

**Entrada:** JSON de saída do APB + requisito completo.

**Saída (JSON Schema):**
```json
{
  "user_story_detalhada": {
    "id": "US-023",
    "titulo": "Pagamento Instantâneo via Pix",
    "narrativa": "Como Cliente do sistema, quero realizar pagamentos via Pix, para que eu conclua minha compra instantaneamente.",
    "cenarios_gherkin": [
      {
        "cenario": "Sucesso no pagamento Pix dentro do tempo limite",
        "estrutura": [
          "Dado que o usuário selecionou Pix como forma de pagamento",
          "Quando ele ler o QR Code e confirmar a transação no banco",
          "Então o sistema deve exibir a tela de sucesso em até 3 segundos"
        ]
      }
    ],
    "validacao_smart": {
      "aprovado": true,
      "justificativa": "Critério mensurável (≤3s), específico ao fluxo Pix e testável via caso de teste CT-PAG-010-01."
    },
    "status": "PRONTO_PARA_DEV",
    "destino": "Repositório / Jira"
  }
}
```

---

## 4. Protocolo de Segurança entre Agentes (Múltiplas IAs)

**Anonimização e Tokenização de Dados Sensíveis (PII Masking):**

Qualquer payload que carregue dados potencialmente reais de usuários finais — o que normalmente entra na pipeline pelo AER, na transcrição bruta — deve passar por uma camada de tokenização/anonimização de strings via regex antes de ser processado por qualquer chamada a uma API de LLM, em qualquer agente da cadeia (AER, AES, AR, AC, APB, AUS). O valor real só é re-hidratado dentro de um cofre seguro (vault), isolado do restante do sistema — nunca dentro das mensagens trocadas entre agentes, nem dentro do `payload` gravado em `log_auditoria`.

**Imutabilidade de Histórico:**

Toda a cadeia de JSONs gerada pelos agentes é armazenada em formato estruturado (PostgreSQL JSONB) com hash SHA-256 vinculando a mensagem anterior à nova mensagem:

```
hash_atual = SHA256(hash_anterior + payload_jsonb_serializado)
```

Isso impede adulteração de baselines de auditoria técnica — qualquer alteração retroativa em um registro quebra a cadeia de hash de todos os registros posteriores, tornando a violação detectável.

**Direito à Eliminação vs. Imutabilidade:**

A LGPD garante ao titular o direito à eliminação de seus dados (Art. 18, VI). O conflito com a imutabilidade da trilha de auditoria é só aparente:

- O `payload` em `log_auditoria` nunca contém dado pessoal bruto re-identificável — só o token gerado pela camada de anonimização.
- O vault guarda, separadamente, o mapeamento token ↔ valor real.
- Ao concluir uma `solicitacao_titular` do tipo `ELIMINACAO`, apaga-se a entrada do vault, não o registro em `log_auditoria`. O token permanece na cadeia de hash, mas deixa de ser re-hidratável — efeito equivalente a um *crypto-shredding*.

**Não-Exclusão Física (RN-AUD-004):**

Nenhuma entidade rastreável do sistema é excluída fisicamente do banco. Toda rota de API com verbo `DELETE` implementa exclusão lógica (`ativo = false`, ou transição para estado terminal), preservando a cadeia de rastreabilidade e a trilha de auditoria.

---

## 5. Tratamento de Falhas e Resiliência

Cada agente depende de uma chamada de API externa a um modelo de LLM — sem uma estratégia explícita de falha, qualquer instabilidade da API trava ou corrompe a cadeia de eventos.

| Cenário | Estratégia |
|---|---|
| Timeout da API do LLM | Timeout de 30s por chamada; retry automático com backoff exponencial (até 3 tentativas) |
| Falha definitiva após retries | O requisito permanece no estado anterior; é marcado em uma fila de reprocessamento; nenhum log de auditoria é gravado para uma execução que não retornou um JSON válido |
| Resposta do LLM fora do schema esperado | Validação via Pydantic antes de qualquer escrita no banco; resposta inválida é descartada e tratada como falha |
| Reprocessamento (retry manual ou automático) | Cada chamada de agente carrega um `id_evento` idempotente — reprocessar o mesmo evento não duplica entradas em `log_auditoria` nem em `requisitos_versoes` |
| 3ª reprovação consecutiva do AC | Transição automática para `BLOQUEADO_REVISAO_HUMANA` (RN-PROC-006, §2) — parada intencional do fluxo, não falha técnica |

> Esta camada é de infraestrutura/orquestração — não faz parte do *system prompt* de nenhum agente, e sim do código em `services/agentes/*.py` que invoca o agente e trata o retorno.

---

## 6. Observabilidade e Logs Operacionais

A trilha de auditoria (`log_auditoria`, §4) tem propósito de conformidade e prova de integridade — não deve ser usada como fonte de observabilidade operacional. São duas camadas independentes:

| Camada | Propósito | Onde vive |
|---|---|---|
| `log_auditoria` (hash chain) | Prova de integridade, conformidade, auditoria externa | PostgreSQL JSONB, *append-only* |
| Logs operacionais | Debug, latência por chamada de agente, taxa de erro/retry, alertas | Stack de observabilidade própria (logs estruturados + métricas), fora do escopo deste documento |

Nenhum dado pessoal, mesmo tokenizado, deve aparecer em logs operacionais de nível `INFO`/`DEBUG` — eles não têm a mesma rigidez de controle de acesso que `log_auditoria`.
