# 📘 Diagnóstico de Performance de Query em Ambiente Produtivo no DB2 for z/OS

---

## 📑 Índice

- [🎯 Objetivo](#-objetivo)
- [🧭 1. Entendimento do Contexto](#-1-entendimento-do-contexto)
- [🛠️ 2. Levantamento Técnico Necessário](#-2-levantamento-técnico-necessário)
  - [2.1 Query Envolvida](#21-query-envolvida)
  - [2.2 Plano de Acesso (EXPLAIN)](#22-plano-de-acesso-explain)
  - [2.3 Estatísticas Atualizadas (RUNSTATS)](#23-estatísticas-atualizadas-runstats)
  - [2.4 Índices Disponíveis](#24-índices-disponíveis)
  - [2.5 Ambiente de Execução](#25-ambiente-de-execução)
  - [2.6 Recursos Consumidos](#26-recursos-consumidos)
- [🔍 3. Ações Já Realizadas](#-3-ações-já-realizadas)
- [🎯 4. Possíveis Direcionamentos (com explicações)](#-4-possíveis-direcionamentos-com-explicações)
- [📌 Exemplo de RUNSTATS Ideal](#-exemplo-de-runstats-ideal)
- [🧾 Fontes Oficiais de Apoio (IBM)](#-fontes-oficiais-de-apoio-ibm)
- [✅ Condução da Reunião](#-condução-da-reunião)

---

## 🎯 Objetivo

A lentidão de queries em ambientes críticos geralmente está relacionada à escolha ineficiente do plano de acesso pelo otimizador do DB2. Este diagnóstico técnico foi construído para ser utilizado em **reuniões urgentes ou preventivas**, com foco em análise prática e precisa, mesmo em situações onde o conhecimento do ambiente é parcial. Aqui unimos análise de contexto, estatísticas, índices, plano de acesso e ações viáveis para reverter degradações de performance.

---

## 🧭 1. Entendimento do Contexto

- Qual sistema ou aplicação está impactado?
- Ocorre em lote, online, ou ambos?
- Ocorre em:
  - ☐ Execução pontual
  - ☐ Batch programado
  - ☐ Alta concorrência (OLTP)
- Desde quando ocorre o problema?
- Alguma alteração recente?
  - Novo deploy?
  - Crescimento de volume?
  - Alteração de índices?
  - Atualização de estatísticas?

---

## 🛠️ 2. Levantamento Técnico Necessário

### 2.1 Query Envolvida

- Código completo da SQL envolvida.
- Identificação de:
  - Funções em colunas de filtro (ex: `DATE()`, `SUBSTR()`).
  - Joins complexos?
  - Subqueries correlacionadas?
- Identificar filtros e ordenações.

---

### 2.2 Plano de Acesso (EXPLAIN)

- Foi executado `EXPLAIN(YES)` no `REBIND`?
- Verificar:
  - Uso de índices (`MATCHCOLS`, `ACCESSNAME`);
  - Tipo de acesso: **Index Only**, **Index Screening**, **Tablespace Scan**;
  - Joins: Nested Loop, Merge Join, Hybrid Join;
  - Uso de **workfile** (indicando SORTs);
  - Uso de `PREFETCH` (seq ou list).

🔎 **Dica:** `MATCHCOLS=0` geralmente indica full table scan.

---

### 2.3 Estatísticas Atualizadas (RUNSTATS)

#### ✅ Diagnóstico com foco em RUNSTATS:

- Qual comando exato foi executado?
  - Usaram `FREQVAL`, `HISTOGRAM`, `KEYCARD`?
  - Coletaram para colunas fora dos índices?
- Foi feito `COLGROUP`?
- O `RUNSTATS` incluiu `REPORT YES` para validação?

📌 **Importância:**
> O otimizador depende dessas estatísticas para estimar custo e volume. Sem elas, pode assumir seleções incorretas, levando a planos ineficientes.

---

### 2.4 Índices Disponíveis

- Quais índices existem nas tabelas principais?
- Há **índice composto** útil para os filtros?
- Algum índice com `INCLUDE`?

#### 🔍 O que é `INCLUDE`?

- O `INCLUDE` permite adicionar colunas **não-chave** a um índice.
- Essas colunas não participam da ordenação do índice, mas tornam possível um **Index Only Access**, evitando que o DB2 tenha que acessar a tabela base (data page).
- Isso **reduz drasticamente o custo I/O** e melhora o tempo de resposta.

📌 **Exemplo**:
Suponha uma query que filtre por `COD_AGENCIA` e `COD_PRODUTO`, mas também selecione `DS_PRODUTO`:

```sql
SELECT DS_PRODUTO 
FROM TB_PRODUTO 
WHERE COD_AGENCIA = '1234' AND COD_PRODUTO = 'XPTO'
```

Se existir um índice:
```sql
CREATE INDEX IDX_PRODUTO_01 
  ON TB_PRODUTO (COD_AGENCIA, COD_PRODUTO)
  INCLUDE (DS_PRODUTO)
```

> O DB2 poderá **resolver a query somente com o índice**, sem acessar a tabela.

---

### 2.5 Ambiente de Execução

- Acesso ocorre por:
  - Stored Procedure?
  - Dynamic SQL via CICS, IMS, Web?
  - Pacote específico?
- É workload OLTP, OLAP ou batch?
- O plano foi gerado com qual nível de otimização?
  - `REOPT(NONE | ONCE | ALWAYS)`?

#### 📌 Sobre o `REOPT`:
- `REOPT(ONCE)` ou `REOPT(ALWAYS)` devem ser usados quando a query depende de parâmetros que variam muito.
- Sem isso, o plano pode ser gerado com base em valores típicos e se tornar ineficiente para parâmetros incomuns.

---

### 2.6 Recursos Consumidos

- **CPU Time** vs **Elapsed Time** — importante para identificar gargalos de I/O.
- Classe de medição (classe 1, 2, 3 — SMF ou Monitor)
- Quantidade de páginas lidas (`GETPAGE`, `READ I/O`)
- Quantidade de registros retornados e descartados (verificar seletividade)

---

## 🔍 3. Ações Já Realizadas

- Foi executado `RUNSTATS`? Com quais parâmetros?
- Houve `REBIND PACKAGE(... EXPLAIN(YES))`?
- O plano de acesso foi alterado após o `RUNSTATS`?
- Foi testado uso de `COLGROUP`, `FREQVAL`, `HISTOGRAM`?
- Tentaram reescrever a query?
- Alguma tentativa de criação ou alteração de índice?

---

## 🎯 4. Possíveis Direcionamentos (com explicações)

### ✅ Reescrita da Query
- Substituir funções por literais (ex: `YEAR(DATA)` por `DATA BETWEEN :INICIO AND :FIM`)
- Remover `DISTINCT`, `ORDER BY` desnecessários
- Transformar subqueries correlacionadas em `JOIN`

### ✅ Atualização de Estatísticas (RUNSTATS)
- Executar `RUNSTATS` com:
  - `FREQVAL NUMCOLS 1 ON COLUMNS(...)`
  - `COLGROUP` para colunas filtradas em conjunto
  - `HISTOGRAM` para valores não uniformes
  - `KEYCARD` sempre!

### ✅ Ajustes no Plano (REBIND)
- Utilizar `REBIND PACKAGE(... EXPLAIN(YES) OPTHINT(...))` para influenciar escolha do plano
- Usar `REOPT(ALWAYS)` quando o valor dos parâmetros afeta drasticamente a seletividade

### ✅ Criação de Índices Estratégicos
- Índice composto com as colunas mais seletivas
- Utilizar `INCLUDE` para evitar acesso à tabela base
- Excluir colunas desnecessárias para manter o índice enxuto

### ✅ Análise de Stage 1 vs Stage 2
- Condições filtradas em **Stage 1** são aplicadas diretamente no acesso
- Filtros em **Stage 2** são aplicados após os dados serem carregados, o que consome muito mais recursos

> 📌 Exemplo de filtro em Stage 2 (evite):
```sql
WHERE YEAR(DATA_VENCIMENTO) = 2024
```
> ✅ Versão Stage 1 (otimizada):
```sql
WHERE DATA_VENCIMENTO BETWEEN '2024-01-01' AND '2024-12-31'
```

---

## 📌 Exemplo de RUNSTATS Ideal

```sql
RUNSTATS TABLESPACE DBX.TSX 
  TABLE(ALL) 
  INDEX(ALL) 
  KEYCARD 
  FREQVAL NUMCOLS 1 
  FREQVAL NUMCOLS 2 ON COLUMNS(COL1, COL2)
  HISTOGRAM ON COLUMNS(COL1, COL2)
  REPORT YES
```

### 📘 Explicações:

- `FREQVAL`: melhora seletividade de filtros comuns
- `HISTOGRAM`: essencial quando há valores muito dominantes ou raros
- `KEYCARD`: atualiza a cardinalidade dos índices
- `COLGROUP`: evita erros de estimativa ao filtrar por múltiplas colunas
- `REPORT YES`: permite revisar as estatísticas geradas

---

## 🧾 Fontes Oficiais de Apoio (IBM)

- [IBM Documentation - Db2 for z/OS 13](https://www.ibm.com/docs/en/db2-for-zos/13)
- [Db2 13 Performance Topics - Redbook](https://www.redbooks.ibm.com/abstracts/sg248551.html)
- [RUNSTATS Utility - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-runstats-utility)
- [Db2 Explain Tables Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=information-explain-table-descriptions)

---

## ✅ Condução da Reunião

1. Inicie confirmando o **impacto real** do problema (tempo, volume, indisponibilidade).
2. Use os tópicos deste manual para **guiar as perguntas técnicas**.
3. Solicite as evidências:
   - Query SQL
   - Plano de acesso (EXPLAIN)
   - Estatísticas (RUNSTATS com REPORT)
4. Reforce que o diagnóstico técnico **pode evitar parada do sistema**, desde que todas as informações estejam disponíveis.

---
