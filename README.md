# CNH Verify — Documentação do Workflow (n8n)

> Este arquivo explica **o que** o fluxo faz, **por que** cada nó existe e **como** testar. Onde houver expressão/código, o essencial para entender a ideia.

---

## Sumário

1. [Visão Geral](#visão-geral)
2. [Arquitetura por Componentes](#arquitetura-por-componentes)

   * [1) Entrada & Triagem](#1-entrada--triagem)
   * [2) Sanidade de URLs (HEAD Checks)](#2-sanidade-de-urls-head-checks)
   * [3) Extração (IDP/VIO) & Normalização](#3-extração-idpvio--normalização)
   * [4) Merge & Decisão](#4-merge--decisão)
   * [5) Resposta HTTP](#5-resposta-http)
3. [Contratos de Entrada e Saída](#contratos-de-entrada-e-saída)
4. [Guia de Testes](#guia-de-testes)
5. [Solução de Problemas Comuns](#solução-de-problemas-comuns)
6. [Notas de Projeto](#notas-de-projeto)

---

## Visão Geral

![](fluxoGeral.png)

O workflow **valida** duas URLs de imagens (frente/verso da CNH), **extrai** dados em dois motores (IDP e VIO), **consolida** os campos, **calcula** uma pontuação com pesos e **decide** o status final:

* **match** (aprovado)
* **review** (revisão manual)
* **fail** (reprovado)
* **hard-fail** (reprovação imediata por regra dura)
* **unknown** (fallback — não esperado)

A resposta volta em JSON, com score, campos que bateram, possíveis divergências e, quando aprovado, um **resumo da pessoa** (nome, CPF, datas etc.).

---

## Arquitetura por Componentes

> Abaixo, cada bloco do diagrama corresponde a um “componente” lógico. Em cada componente descrevo **nó a nó** (finalidade e como funciona).

### 1) Entrada & Triagem

![](entradaTriagem.png)

**Nós:**

* **Webhook**

  * Porta de entrada HTTP (POST). Recebe o corpo com `imageFrontUrl` e `imageBackUrl`.
  * Responde via nós “Respond to Webhook” no final.

* **validate-input (Set)**

  * Verifica **presença** dos campos, **formato https** e **diferença** entre frente/verso.
  * Se algo não bate, gera `input_errors` (objeto) com `field`, `code` e `message`.

* **input-ok (IF)**

  * Roteia:

    * **true**: segue o fluxo normal.
    * **false**: vai para a construção de erro 400.

* **pick-urls (Set)**

  * Normaliza nomes para uso interno:

    * `imageFrontUrl` → `front_url`
    * `imageBackUrl`  → `back_url`
  * Ajuda a manter consistência nas próximas etapas.

* **guard-urls (IF)**

  * Garante **existência + https + urls diferentes** novamente (último “gate”).
  * Se falhar, explica exatamente o porquê (campo ausente, `http://`, iguais, etc.).

* **classify-input-error (Set)**

  * Monta a “causa raiz” para 400: lista de `reasons` com `field`, `code` e `msg`.

* **build:bad-request (Set) & respond:bad-request (Respond to Webhook)**

  * Resposta **HTTP 400**.

---

### 2) Sanidade de URLs (HEAD Checks)

![](sanidadeURLs.png)

**Por quê existe:** Antes de gastar processamento de extração, checamos se **as URLs existem** e se são mesmo imagens (Content-Type).

**Nós:**

* **head-cnh-front (HTTP Request – HEAD)**

  * Faz um **HEAD** em `front_url`. Traz status, `Content-Type` e tamanho (`Content-Length`).

* **check-front (Set)**

  * Marca `front_ok` = `true/false` e embute motivo (`front_reason`) se falhar.

* **head-cnh-back (HTTP Request – HEAD)**

  * Idem para `back_url`.

* **check-back (Set)**

  * `back_ok` = `true/false`, com `back_reason` se falhar.

* **merge-head-checks (Merge → Position)**

  * Junta os resultados front/back.

* **validate-headers (Set)**

  * Regras de sanidade:

    * `status == 200`
    * `content-type` começa com `image/`
    * tamanho > 0
  * Produz lista de `reasons` caso algum lado falhe.

* **heads-ok? (IF)**

  * **true**: URLs válidas → segue.
  * **false**: monta 422.

* **payload:images (Set)**

  * Prepara um payload enxuto para extração (URLs + metadados úteis).

* **build:headers-bad-request (Set) & respond:bad-heads (Respond)**

  * **HTTP 422** com explicação do que deu errado *no acesso às imagens*.

---

### 3) Extração (IDP/VIO) & Normalização

![](extracao.png)

**Objetivo:** Extrair dados **independentes** em dois motores, validar saída e normalizar o formato para merge.

**Ramos paralelos:**

#### Ramo IDP

* **auth:idp-jwt (Sub-workflow)**

  * Obtém um JWT (token) para o serviço IDP.

* **ctx:attach-jwt-idp (Set)**

  * Gera `authHeader = "Bearer <jwt>"`.

* **idp:content-extraction (HTTP Request – POST)**

  * Envia o payload de imagens. Recebe `result[0].fields[]`.

* **idp-ok? (IF)**

  * Verifica se veio `result` com `fields`.
  * **false** → `build:idp-extract-error` (mensagem) + `respond:idp-extract-error (422)`.

* **store:idp (Set)**

  * Guarda o **JSON completo** do IDP (`idp_raw`) para auditoria.

* **normalize:idp (Set)**

  * Mapeia os campos em nomes **comuns** (ex.: `name`, `cpf`, `birth_date`, `issue_date`, `expiry_date`, `license_category`, `license_type`, `renach`, `security_code`).
  * Aplica normalizações leves (upper-case opcional, trim, etc.).

* **validate:idp (IF)**

  * **Regras essenciais** (ex.: `cpf` não vazio, datas coerentes).
  * **false** → `build:idp-validate-error` + `respond:idp-validate-error (422)`.

#### Ramo VIO

* **auth:vio-jwt → ctx:attach-jwt-vio → vio:extraction (POST) → vio-ok? (IF) → store:vio → normalize:vio → validate:vio (IF)**

  * Idêntico ao ramo IDP, mas com o motor VIO.
  * Mesmas rotas de erro 422 específicas para VIO.

---

### 4) Merge & Decisão

![](mergeDecisao.png)

**Ideia:** Consolidar IDP+VIO, gerar um “consenso” de dados, *pontuar* e *decidir*.

**Nós:**

* **merge:sources (Merge → Position)**

  * Junta `normalize:idp` e `normalize:vio` em um só item.
  * Mantém ambos os conjuntos de campos para comparação e “consensus”.

* **compare:fields (Set)** *(se habilitado no seu fluxo)*

  * Compara pares chave (nome, cpf, datas, categoria, tipo_hab).
  * Produz flags `match_nome`, `match_cpf`, etc., e uma versão normalizada do conjunto (`__norm`).

* **config:scoring (Set)**

  * Traz configuração de pesos/limiares **no próprio item**:

    * `weights`: `{ cpf: 0.40, nome: 0.20, nasc: 0.15, emissao: 0.10, validade: 0.10, categoria: 0.03, tipo_hab: 0.02 }`
    * `thresholds`: `{ approve_min: 0.90, review_min: 0.75 }`
  * Vantagem: ajustar política **sem** mexer nas expressões dos outros nós.

* **calc:score (Set)**

  * Soma ponderada das `match_*` com os **pesos**.
  * Gera: `score_final` (0–1), `score_breakdown` e ecoa `approve_min`/`review_min`.

* **policy:thresholds (Set)**

  * Converte `score_final` para `score_status`:

    * `>= approve_min` → `"match"`
    * `>= review_min && < approve_min` → `"review"`
    * `< review_min` → `"fail"`

* **hardchecks (Set)**

  * **Regras duras**:

    * CPF divergente (`!match_cpf`) ⇒ **hard_fail**.
    * Expiry inválido/ausente (`!match_validade`) ⇒ **hard_fail**.
  * Sinaliza `hard_block = true/false` e `hard_reason`.

* **route:hard (IF)**

  * Se `hard_block == true` → **hard-fail** imediato (monta resposta e 200/JSON, ou 409/422 conforme política).
  * Senão, segue o fluxo normal.

* **decide:status (Set)**

  * Aplica a política: se não for hard, usa `score_status` como `status`.
  * Monta `matched_fields`, `mismatches` e `status_reason`.

* **build:response (Set)**

  * Empacota o resultado padrão: `status`, `score`, `score_threshold`, `score_status`, `matched_fields`, `mismatches`, `reason`.

* **Switch (mode: Expression)**

  * Roteia por status:

    * **match** → sucesso
    * **review** → revisão
    * **fail** → reprovação
    * (default) **unknown** → fallback seguro

---

### 5) Resposta HTTP

![](resposta.png)

**Nós:**

* **build:success-response (Set)**

  * Enriquecimento da resposta de sucesso:

    * `person`: agrega os **principais dados** (nome, cpf, datas, categoria, tipo).

      * Dica: buscamos do `merge:sources`/`normalize:*` priorizando onde a fonte é mais confiável (ex.: VIO → `renach`, `security_code`; IDP → `name`, `birth_date`).
    * Mantém `status`, `score`, `matched_fields` etc.

* **respond:success (Respond to Webhook — JSON, 200)**

* **build:review-response / respond:review**

  * Idêntico ao sucesso, mas `status: "review"` e razão apropriada.

* **build:fail-response / respond:fail**

  * `status: "fail"`, com `mismatches` e `status_reason` explicando a causa.

* **build:unknown-response / application/json (Respond)**

  * Fallback não esperado; loga e retorna uma mensagem amigável.

* **hard:fail-response / respond:hard-fail** *(se route:hard for true)*

  * Resposta direta de reprovação por regra dura (ex.: CPF divergente).

---

## Guia de Testes

> Use Insomnia/Postman. Sempre preferir imagens reais acessíveis via `https://`.

### Casos Felizes

* **Ambas as URLs válidas, dados consistentes → `match`**

  ```json
  {
    "imageFrontUrl": "https://.../frente_cnh.jpg",
    "imageBackUrl":  "https://.../verso_cnh.jpg"
  }
  ```
* No Insomnia, cole o conteúdo acima mudando a o endereço para um válido e use o método  `POST`
* Teste usando a URL: `https://desafiomost.app.n8n.cloud/webhook/cnh-verify`
---

### Fim

 Qualquer ajuste de regras, comece pelo nó **config:scoring** (pesos/limiares) e **hardchecks** (bloqueios).
