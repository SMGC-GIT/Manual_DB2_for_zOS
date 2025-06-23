# 📘 Diagnóstico com Foco em Access Path (EXPLAIN)

> **Seção do Manual**: `04.03.3 – Diagnóstico com foco em Access Path`

---

## 📑 Índice

- [🎯 Objetivo](#-objetivo)
- [🔍 1. O que é o Access Path](#-1-o-que-é-o-access-path)
- [🧰 2. Como gerar o plano de acesso (EXPLAIN)](#-2-como-gerar-o-plano-de-acesso-explain)
- [🔎 3. Interpretação detalhada do EXPLAIN](#-3-interpretação-detalhada-do-explain)
  - [3.1 METHOD](#31-method)
  - [3.2 MATCHCOLS](#32-matchcols)
  - [3.3 ACCESSNAME](#33-accessname)
  - [3.4 SORTN_JOIN / SORTC_JOIN](#34-sortn_join--sortc_join)
  - [3.5 TSLOCKMODE e PARALLELISM_MODE](#35-tslockmode-e-parallelism_mode)
- [🧠 4. Quando o plano está ruim – e como resolver](#-4-quando-o-plano-está-ruim--e-como-resolver)
- [⚙️ 5. Ações possíveis após diagnóstico](#-5-ações-possíveis-após-diagnóstico)
- [🧾 6. Casos práticos explicados](#-6-casos-práticos-explicados)
- [📎 7. Referências Técnicas IBM](#-7-referências-técnicas-ibm)

---

## 🎯 Objetivo

O Access Path determina **como o DB2 executará uma query SQL**. Ele define:
- Qual índice (se algum) será utilizado;
- Se será feito scan completo ou uso seletivo;
- Se haverá ordenação (sort);
- Tipo de join entre tabelas;
- Uso de paralelismo.

> A escolha errada do plano pode degradar uma query em milissegundos **ou travar um ambiente produtivo inteiro**. Esta seção ensina como gerar e interpretar o EXPLAIN com profundidade técnica e ação prática.

---

## 🔍 1. O que é o Access Path

O Access Path (caminho de acesso) é a estratégia que o otimizador do DB2 usa para buscar os dados.

Ele é calculado com base em:
- **Estatísticas disponíveis** (RUNSTATS);
- **Índices existentes**;
- **Seletividade estimada**;
- **Parâmetros do ambiente** (REOPT, DEGREE, ISOLATION, etc).

⚠️ **Importante**: o plano é sensível a qualquer alteração no ambiente — uma nova estatística, índice ou até a forma da query pode **mudar radicalmente a performance**.

---

## 🧰 2. Como gerar o plano de acesso (EXPLAIN)

### 📌 Método 1: REBIND com EXPLAIN
Rebinda um package existente e grava o plano:

```sql
REBIND PACKAGE(PACKX.YY) EXPLAIN(YES)
```

### 📌 Método 2: EXPLAIN direto na query

```sql
EXPLAIN PLAN FOR
SELECT ...
FROM ...
WHERE ...
```

O plano será armazenado em tabelas como:
- `DSN_PREDICAT_TABLE`
- `DSN_PGRANGE_TABLE`
- `DSN_PPLAN_TABLE`
- Ou views customizadas pela equipe DBA

---

## 🔎 3. Interpretação detalhada do EXPLAIN

### 3.1 METHOD – Tipo de acesso

Essa coluna indica **como os dados serão acessados**.

| Valor | Significado                       |
|-------|-----------------------------------|
| 0     | Table Space Scan (TS Scan)        |
| 1     | Matching Index Only               |
| 2     | Matching Index + Data Page        |
| 3     | Index Screening (non-matching)    |
| 4     | Filtro por IN list ou OR          |
| 5+    | JOINs (Nested Loop, Merge, Hybrid)|

---

#### ⚠️ Explicando...

**METHOD = 0 (TS Scan)**  
- Toda a tabela é lida página por página.
- Indica **ausência de índice útil**.
- **Altamente custoso em tabelas grandes**.

**METHOD = 1 ou 2 (Index Matching)**  
- Método ideal. DB2 usa o índice com correspondência nas colunas do filtro.
- `1` → todas as colunas no índice → acesso direto (**Index Only Access**).
- `2` → precisa acessar a tabela para colunas adicionais.

**METHOD = 3 (Index Screening)**  
- Só a **primeira coluna** do índice casou. As demais são filtradas após leitura do índice.
- Melhor que scan, mas pode ser ineficiente.

**METHOD >= 5 – JOIN Types**

- `5` = Nested Loop Join  
- `6` = Merge Scan Join  
- `7` = Hybrid Join

> 💡 Veja abaixo explicações para JOINs.

---

### 3.2 MATCHCOLS – Colunas do índice utilizadas

Quantidade de colunas do índice que **casam exatamente com os filtros da query**.

- `MATCHCOLS = 0` → índice não está ajudando.
- `MATCHCOLS > 0` → ajuda parcial.
- `MATCHCOLS = total de colunas` do índice → **ideal**.

---

### 3.3 ACCESSNAME – Nome do índice utilizado

Se estiver em branco, significa que **nenhum índice foi utilizado**.

- Se `METHOD = 0` e `ACCESSNAME = NULL`, a tabela foi lida por scan.

---

### 3.4 SORTN_JOIN / SORTC_JOIN – Necessidade de ordenação

Essas colunas mostram se será necessário ordenar os dados.

- `SORTN_JOIN = Y` → precisa sort para Nested Loop Join.
- `SORTC_JOIN = Y` → precisa sort para combinar colunas.

> Toda vez que houver `Y`, o otimizador **irá alocar workfiles** → aumento de I/O, uso de CPU e possível `DSNU020I` se não houver espaço.

---

### 3.5 TSLOCKMODE e PARALLELISM_MODE

- `TSLOCKMODE` indica o tipo de lock aplicado no access path (`S`, `IS`, `IX`, etc).
- `PARALLELISM_MODE` mostra se o plano usará **leitura paralela**, útil em scans grandes.

---

## 🧠 4. Quando o plano está ruim – e como resolver

| Sinal no EXPLAIN                  | Impacto real                                | Ação técnica recomendada            |
|----------------------------------|---------------------------------------------|-------------------------------------|
| `METHOD = 0`                     | Leitura completa → alto custo                | Criar índice / reescrever filtro    |
| `MATCHCOLS = 0`                  | Índice não está ajudando                     | Ajustar ordem das colunas no índice |
| `ACCESSNAME` nulo                | Sem índice em uso                            | Verificar RUNSTATS e REBIND         |
| `SORTN_JOIN = Y`                 | Sort extra para JOIN → usa workfile          | Criar índice que suporte ordenação  |
| `Plano muda após RUNSTATS`       | Estatísticas desatualizadas influenciaram   | Confirmar se novo plano é melhor    |

---

## ⚙️ 5. Ações possíveis após diagnóstico

- **RUNSTATS completo com COLGROUP e HISTOGRAM**  
  Garante que o otimizador conheça a distribuição real dos dados.

- **Criação de índice com colunas certas e ordem correta**  
  Ex: colocar coluna mais seletiva primeiro.

- **Uso de `INCLUDE` para cobrir SELECT**  
  Evita acessar a tabela base → **Index Only Access**.

- **REBIND PACKAGE com EXPLAIN**  
  Força reavaliação do plano.

- **Aplicação de `OPTHINT`**  
  Sugere plano desejado para queries críticas.

---

## 🧾 6. Casos práticos explicados

### ✅ Caso 1: Table Scan prejudicando batch noturno

**EXPLAIN:**
- `METHOD = 0`, `MATCHCOLS = 0`, `ACCESSNAME = NULL`

**Diagnóstico:**
- Filtro por `DATA >= :INI AND DATA <= :FIM`
- Não havia índice para `DATA`

**Solução:**
```sql
CREATE INDEX IDX_DATA ON TB_MOVIMENTACAO (DATA)
RUNSTATS INDEX(ALL) TABLESPACE ...
REBIND PACKAGE(...) EXPLAIN(YES)
```

**Novo plano:**  
`METHOD = 2`, `MATCHCOLS = 1`, `ACCESSNAME = IDX_DATA`

📈 **Resultado:** CPU caiu de 400ms para 19ms.

---

### ✅ Caso 2: Sort causando consumo alto de tempdb

**EXPLAIN:**
- `SORTC_JOIN = Y`
- `ORDER BY UF, NOME`

**Diagnóstico:**
- Tabela tinha índice apenas por `UF`.

**Solução:**
```sql
CREATE INDEX IDX_ORDENACAO ON TB_CLIENTES (UF, NOME)
```

**Novo plano:**  
`SORTC_JOIN = N`

📈 **Resultado:** Eliminado uso de sort, queda de uso de workfiles.

---

## ⚠️ Explicando os tipos de JOIN

### 🔁 Nested Loop Join

- Um conjunto pequeno (tabela driver) é processado para buscar dados na tabela alvo.
- Ideal quando a **tabela externa é pequena e o índice da interna é seletivo**.
- Custo alto se o índice da interna não for eficiente.

### 🧮 Merge Scan Join

- Duas tabelas grandes são ordenadas e varridas simultaneamente.
- Requer **ordenamento compatível** entre elas.
- Bom para grandes volumes se houver índice com mesma ordenação.

### ♻️ Hybrid Join

- Estratégia adaptativa.
- DB2 decide dinamicamente qual técnica é melhor durante a execução.
- Usa estatísticas e comportamento histórico.

> 📌 Use o campo `METHOD` e o tipo de join para identificar se o índice está sustentando corretamente o plano.

---

## 📎 7. Referências Técnicas IBM

- [IBM Db2 for z/OS 13 – EXPLAIN Table Descriptions](https://www.ibm.com/docs/en/db2-for-zos/13?topic=information-explain-table-descriptions)
- [Db2 13 for z/OS Performance Topics – Redbook SG248551](https://www.redbooks.ibm.com/abstracts/sg248551.html)
- [Db2 13 – Access Path Selection](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statistics-access-path-selection)

---
