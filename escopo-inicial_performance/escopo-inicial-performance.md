# 📘 Diagnóstico de Performance de Query em Ambiente Produtivo no DB2 for z/OS

> **Seção do Manual**: `04.03 - Diagnóstico e Otimização de Queries com Impacto em Produção`

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
- [🎯 4. Possíveis Direcionamentos (a Confirmar)](#-4-possíveis-direcionamentos-a-confirmar)
- [🧾 5. Fontes Oficiais de Apoio (IBM)](#-5-fontes-oficiais-de-apoio-ibm)
- [✅ Condução da Reunião](#-condução-da-reunião)

---

## 🎯 Objetivo

Este roteiro visa apoiar o DBA durante reuniões críticas, quando é identificado que uma query está degradando a performance de um sistema produtivo, a ponto de se considerar sua retirada do ar. O objetivo é realizar um diagnóstico técnico estruturado, mesmo que o ambiente ainda seja desconhecido.

---

## 🧭 1. Entendimento do Contexto

- Qual é a aplicação ou sistema impactado?
- Há impacto em tempo real ou é processamento batch?
- O problema ocorre em:
  - ☐ Execução pontual
  - ☐ Lote recorrente
  - ☐ Alta concorrência
- Quando começou a apresentar lentidão?
- Alguma mudança recente no ambiente (deploy, statistics, rebind, volume de dados)?

---

## 🛠️ 2. Levantamento Técnico Necessário

### 2.1 Query Envolvida
- Query completa ou trecho principal
- Existem subqueries, funções escalares ou filtros complexos?

### 2.2 Plano de Acesso (EXPLAIN)
- Acesso por índice ou scan?
- Quantidade de joins?
- Estimativas do otimizador compatíveis com o real?

### 2.3 Estatísticas Atualizadas (RUNSTATS)
- Foi feito `RUNSTATS` recentemente?
- Usaram `DISTRIBUTION`, `FREQVAL`, `HISTOGRAM`?
- Estatísticas coerentes com o volume atual de dados?

### 2.4 Índices Disponíveis
- Quais índices existem nas tabelas envolvidas?
- Algum índice não utilizado com potencial?
- Existe necessidade de índice composto ou INCLUDE?

### 2.5 Ambiente de Execução
- Workload é OLTP, OLAP, batch?
- Acesso ocorre via pacote ou SP?
- Grau de paralelismo (`DEGREE`), `REOPT` usado?

### 2.6 Recursos Consumidos
- CPU total vs Elapsed Time (Classe 2 / Classe 3)
- I/O (quantidade de páginas lidas)
- Uso de workfile (SORTs, joins, etc)

---

## 🔍 3. Ações Já Realizadas

- Foi feito REBIND? Com `EXPLAIN(YES)`?
- Algum ajuste já foi testado ou descartado?
- Foi analisado `FILTER FACTOR`, `MATCHCOLS`, `ACCESSNAME`, `PREFETCH`?
- Houve análise de Stage 1 vs Stage 2?
- Alguma tentativa de reescrita da query ou uso de hints?

---

## 🎯 4. Possíveis Direcionamentos (a Confirmar)

- Reescrita da query (filtros, joins, funções)
- Criação ou ajuste de índices (INCLUDE, COMPOSTO)
- Atualização de estatísticas com:
  ```sql
  RUNSTATS TABLESPACE ... 
    TABLE(ALL) 
    INDEX(ALL) 
    KEYCARD 
    FREQVAL NUMCOLS 1
  ```
- Ajustes no REBIND (REOPT, DEGREE, OPTHINT)
- Redução de SORTs ou uso excessivo de ORDER BY
- Materialização parcial (CGTT, views, etc.)

---

## 🧾 5. Fontes Oficiais de Apoio (IBM)

- [IBM Documentation - Db2 for z/OS 13](https://www.ibm.com/docs/en/db2-for-zos/13)
- [Db2 13 Performance Topics - Redbook](https://www.redbooks.ibm.com/abstracts/sg248551.html)
- [Db2 Explain Tables Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=information-explain-table-descriptions)

---

## ✅ Condução da Reunião

Durante a reunião:
1. Comece pelo impacto percebido.
2. Use o checklist para fazer perguntas técnicas.
3. Reforce que ajustes devem ser baseados em dados reais.
4. Solicite:
   - A query
   - Plano de acesso (EXPLAIN)
   - Estatísticas
5. Explique que pode haver solução **sem tirar o sistema do ar**, se houver visibilidade completa.

---
