# 📘 QUERIES SQL - DB2 for z/OS ( Exemplos )

---

## 🔍 1. Consultar Relacionamentos entre Tabelas

### 1.1. Tabelas que Referenciam uma Tabela Específica (Filhas)

```sql
SELECT A.CREATOR AS ESQUEMA_FILHA,
       A.TBNAME AS TABELA_FILHA,
       A.RELNAME AS NOME_CONSTRAINT,
       B.COLNAME AS COLUNA_CHAVE_ESTRANGEIRA,
       B.COLSEQ AS ORDEM_COLUNA,
       A.REFTBCREATOR AS ESQUEMA_MAE,
       A.REFTBNAME AS TABELA_MAE
  FROM SYSIBM.SYSRELS A
  JOIN SYSIBM.SYSFOREIGNKEYS B
    ON A.RELNAME = B.RELNAME
 WHERE A.REFTBCREATOR = 'SEU_ESQUEMA'
   AND A.REFTBNAME = 'SUA_TABELA'
 ORDER BY A.RELNAME, B.COLSEQ;
```

### 1.2. Tabelas que São Referenciadas por uma Tabela Específica (Pais)

```sql
SELECT A.REFTBCREATOR AS ESQUEMA_MAE,
       A.REFTBNAME AS TABELA_MAE,
       A.RELNAME AS NOME_CONSTRAINT,
       B.COLNAME AS COLUNA_CHAVE_ESTRANGEIRA,
       B.COLSEQ AS ORDEM_COLUNA
  FROM SYSIBM.SYSRELS A
  JOIN SYSIBM.SYSFOREIGNKEYS B
    ON A.RELNAME = B.RELNAME
 WHERE A.CREATOR = 'SEU_ESQUEMA'
   AND A.TBNAME = 'SUA_TABELA'
 ORDER BY A.RELNAME, B.COLSEQ;
```

---

## 🧭 2. Identificar Tabelas Filhas sem Tabela Pai

```sql
SELECT A.CREATOR AS ESQUEMA_FILHA,
       A.TBNAME AS TABELA_FILHA,
       A.RELNAME AS NOME_CONSTRAINT
  FROM SYSIBM.SYSRELS A
 WHERE NOT EXISTS (
       SELECT 1
         FROM SYSIBM.SYSTABLES B
        WHERE B.CREATOR = A.REFTBCREATOR
          AND B.NAME = A.REFTBNAME
       )
 ORDER BY A.CREATOR, A.TBNAME;
```

---

## 📂 3. Listar Todas as Tabelas de um Tablespace

```sql
SELECT CREATOR AS ESQUEMA,
       NAME AS NOME_TABELA
  FROM SYSIBM.SYSTABLES
 WHERE DBNAME = 'SEU_DATABASE'
   AND TSNAME = 'SEU_TABLESPACE'
 ORDER BY CREATOR, NAME;
```

---

## 🧱 4. Consultar Estrutura de uma Tabela (Colunas e Tipos)

```sql
SELECT COLNAME AS NOME_COLUNA,
       COLNO AS NUMERO_COLUNA,
       TYPENAME AS TIPO_DADO,
       LENGTH AS TAMANHO,
       SCALE AS ESCALA,
       NULLS AS ACEITA_NULO
  FROM SYSIBM.SYSCOLUMNS
 WHERE TBNAME = 'SUA_TABELA'
   AND TBCREATOR = 'SEU_ESQUEMA'
 ORDER BY COLNO;
```

---

## 🔗 5. Consultar Índices de uma Tabela

```sql
SELECT NAME AS NOME_INDICE,
       UNIQUERULE AS REGRA_UNICIDADE,
       COLNAMES AS COLUNAS
  FROM SYSIBM.SYSINDEXES
 WHERE TBNAME = 'SUA_TABELA'
   AND TBCREATOR = 'SEU_ESQUEMA'
 ORDER BY NAME;
```

---

## 🧾 6. Consultar Constraints de uma Tabela

```sql
SELECT CONSTNAME AS NOME_CONSTRAINT,
       TYPE AS TIPO_CONSTRAINT,
       TBNAME AS TABELA,
       TBCREATOR AS ESQUEMA
  FROM SYSIBM.SYSCHECKS
 WHERE TBNAME = 'SUA_TABELA'
   AND TBCREATOR = 'SEU_ESQUEMA'
 ORDER BY CONSTNAME;
```

---

## 📊 7. Consultar Tabelas com Mais Relacionamentos

```sql
SELECT A.TBNAME AS TABELA,
       A.CREATOR AS ESQUEMA,
       COUNT(B.RELNAME) AS NUMERO_RELACIONAMENTOS
  FROM SYSIBM.SYSTABLES A
  LEFT JOIN SYSIBM.SYSRELS B
    ON A.NAME = B.TBNAME
   AND A.CREATOR = B.CREATOR
 GROUP BY A.TBNAME, A.CREATOR
 ORDER BY NUMERO_RELACIONAMENTOS DESC;
```

---

---

## 🧠 Expansão: Consultas Avançadas para Diagnóstico e Análise

---

## 📌 8. Tabelas Sem Índices

```sql
SELECT A.NAME AS TABELA,
       A.CREATOR AS ESQUEMA
  FROM SYSIBM.SYSTABLES A
 WHERE A.TYPE = 'T'
   AND NOT EXISTS (
       SELECT 1
         FROM SYSIBM.SYSINDEXES B
        WHERE B.TBCREATOR = A.CREATOR
          AND B.TBNAME = A.NAME
       )
 ORDER BY A.CREATOR, A.NAME;
```

---

## 📊 9. Estatísticas de RUNSTATS (última execução)

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       STATS_TIME AS ULTIMA_COLETA_STATS,
       CARDF AS NUMERO_LINHAS,
       NPAGES AS PAGINAS_USADAS,
       FPAGES AS PAGINAS_FISICAS
  FROM SYSIBM.SYSTABLES
 WHERE STATS_TIME IS NOT NULL
 ORDER BY STATS_TIME DESC;
```

---

## 🚧 10. Tabelas em Estado CHECK PENDING

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       CHECKPEND AS ESTADO_CHECKPENDING
  FROM SYSIBM.SYSTABLES
 WHERE CHECKPEND = 'Y'
 ORDER BY CREATOR, NAME;
```

---

## 📈 11. Tabelas com Maior Número de Registros

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       CARDF AS NUMERO_LINHAS
  FROM SYSIBM.SYSTABLES
 WHERE TYPE = 'T'
   AND CARDF IS NOT NULL
 ORDER BY CARDF DESC
 FETCH FIRST 20 ROWS ONLY;
```

---

## 💽 12. Uso de Espaço por Tablespace

```sql
SELECT DBNAME,
       NAME AS TABLESPACE,
       SPACE AS ESPACO_UTILIZADO_KB,
       NPAGES AS NUM_PAGINAS_USADAS,
       FPAGES AS NUM_PAGINAS_TOTAIS
  FROM SYSIBM.SYSTABLESPACE
 ORDER BY SPACE DESC;
```

---

## 📦 13. Tabelas em um Tablespace com Estatísticas de Tamanho

```sql
SELECT T.CREATOR AS ESQUEMA,
       T.NAME AS TABELA,
       T.CARDF AS NUM_LINHAS,
       T.STATS_TIME AS DATA_STATS,
       TS.DBNAME AS DATABASE,
       TS.NAME AS TABLESPACE
  FROM SYSIBM.SYSTABLES T
  JOIN SYSIBM.SYSTABLESPACE TS
    ON T.DBNAME = TS.DBNAME
   AND T.TSNAME = TS.NAME
 WHERE T.CREATOR = 'SEU_ESQUEMA'
   AND TS.NAME = 'SEU_TABLESPACE'
 ORDER BY T.NAME;
```

---

## 🔍 14. Consultar Planos de Acesso (EXPLAIN básico)

```sql
SELECT PLANNO AS NUMERO_PLANO,
       METHOD AS METODO_ACESSO,
       TABNAME AS TABELA,
       ACCESSNAME AS INDICE,
       MATCHCOLS AS COLUNAS_MATCH,
       ACCESSTYPE AS TIPO_ACESSO,
       PREFETCH AS PREFETCH,
       JOIN_TYPE AS TIPO_JOIN
  FROM PLAN_TABLE
 WHERE QUERYNO = 1
 ORDER BY PLANNO;
```

ℹ️ *Pré-requisito*: Executar `EXPLAIN` sobre o SQL desejado, direcionando saída para `PLAN_TABLE`.

---

## 📛 15. Tabelas com Nome Fora do Padrão (ex: sem prefixo)

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA
  FROM SYSIBM.SYSTABLES
 WHERE TYPE = 'T'
   AND NAME NOT LIKE 'PREFIXO_%'
 ORDER BY CREATOR, NAME;
```

---

## 📌 Observações Importantes

- Todas as queries são baseadas nos catálogos padrão do **DB2 for z/OS** (`SYSIBM.SYSTABLES`, `SYSINDEXES`, `SYSTABLESPACE`, etc.).
- Ajuste os filtros conforme sua convenção de nomenclatura (`SEU_ESQUEMA`, `SEU_TABLESPACE`, `PREFIXO_`, etc.).
- Use `WITH UR` (uncommitted read) ao final de queries em ambientes de produção, quando necessário e permitido.

---

## 📌 Notas Finais

- Substitua `'SEU_ESQUEMA'`, `'SUA_TABELA'`, `'SEU_DATABASE'` e `'SEU_TABLESPACE'` pelos valores correspondentes ao seu ambiente.
- As tabelas do catálogo `SYSIBM` fornecem informações detalhadas sobre a estrutura e os relacionamentos do banco de dados.
- Utilize essas consultas como base para scripts de auditoria, documentação e manutenção do ambiente DB2.

---


## 🎯 16. Exemplos Práticos com Resultados Esperados

Consultas avançadas otimizadas para o dia a dia do DBA, com amostras de resultados esperados e links de referência para aprofundamento.

---

### 🔍 1. Tabelas sem Índices

📌 **Objetivo:** Encontrar tabelas que não possuem nenhum índice associado. Pode indicar problemas de performance em acessos.

```sql
SELECT A.NAME AS TABELA,
       A.CREATOR AS ESQUEMA
  FROM SYSIBM.SYSTABLES A
 WHERE A.TYPE = 'T'
   AND NOT EXISTS (
       SELECT 1
         FROM SYSIBM.SYSINDEXES B
        WHERE B.TBCREATOR = A.CREATOR
          AND B.TBNAME = A.NAME
       )
 ORDER BY A.CREATOR, A.NAME
 WITH UR;
```

📋 **Resultado Esperado:**

| TABELA     | ESQUEMA   |
|------------|-----------|
| CLIENTES   | DBUSER01  |
| AUDITORIA  | LOGSADM   |
| TEMP_PEDID | TEMPZONE  |

📚 **Referência:** [SYSINDEXES - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=catalog-sysindexes)

---

### 📊 2. Tabelas com Maior Volume de Registros

📌 **Objetivo:** Listar as tabelas com maior quantidade de registros (estatísticas coletadas via RUNSTATS).

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       CARDF AS NUMERO_LINHAS
  FROM SYSIBM.SYSTABLES
 WHERE TYPE = 'T'
   AND CARDF IS NOT NULL
 ORDER BY CARDF DESC
 FETCH FIRST 10 ROWS ONLY
 WITH UR;
```

📋 **Resultado Esperado:**

| TABELA       | ESQUEMA   | NUMERO_LINHAS |
|--------------|-----------|----------------|
| TRANSACOES   | FINADM    | 25.439.882     |
| FATURAMENTO  | FINADM    | 12.110.024     |
| CLIENTES     | DBUSER01  | 1.380.922      |

📚 **Referência:** [SYSTABLES - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=catalog-systables)

---

### 🛠️ 3. Tabelas em Estado CHECK PENDING

📌 **Objetivo:** Identificar tabelas que exigem verificação via `CHECK DATA`.

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       CHECKPEND AS ESTADO_CHECKPENDING
  FROM SYSIBM.SYSTABLES
 WHERE CHECKPEND = 'Y'
 ORDER BY CREATOR, NAME
 WITH UR;
```

📋 **Resultado Esperado:**

| TABELA     | ESQUEMA | ESTADO_CHECKPENDING |
|------------|---------|---------------------|
| FAT_CUSTOS | FINADM  | Y                   |
| LOG_ERRO   | APPLOG  | Y                   |

📚 **Referência:** [CHECKPEND Column - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=columns-checkpend)

---

### 🔗 4. Relação de Tabelas Filhas sem Tabela Pai (Referential Orphans)

📌 **Objetivo:** Detectar tabelas que têm FK, mas a tabela referenciada não existe (ou foi removida incorretamente).

```sql
SELECT R.TBCREATOR AS ESQUEMA_FILHA,
       R.TBNAME AS TABELA_FILHA,
       R.REFTBCREATOR AS ESQUEMA_PAI,
       R.REFTBNAME AS TABELA_PAI
  FROM SYSIBM.SYSRELS R
 WHERE NOT EXISTS (
       SELECT 1
         FROM SYSIBM.SYSTABLES T
        WHERE T.CREATOR = R.REFTBCREATOR
          AND T.NAME = R.REFTBNAME
       )
 ORDER BY R.TBCREATOR, R.TBNAME
 WITH UR;
```

📋 **Resultado Esperado:**

| ESQUEMA_FILHA | TABELA_FILHA | ESQUEMA_PAI | TABELA_PAI |
|---------------|---------------|--------------|-------------|
| LOGAPP        | HIST_ACESSO   | APPADM       | USUARIOS    |
| CONTROLE      | MOV_ESTOQUE   | ERP          | PRODUTOS    |

📚 **Referência:** [SYSRELS - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=catalog-sysrels)

---

### 🧾 5. Todas as Tabelas de um Tablespace

📌 **Objetivo:** Identificar todas as tabelas associadas a determinado tablespace.

```sql
SELECT T.NAME AS TABELA,
       T.CREATOR AS ESQUEMA,
       T.DBNAME,
       T.TSNAME
  FROM SYSIBM.SYSTABLES T
 WHERE T.DBNAME = 'DBFINANCA'
   AND T.TSNAME = 'TS_CONTABIL'
 ORDER BY T.CREATOR, T.NAME
 WITH UR;
```

📋 **Resultado Esperado:**

| TABELA      | ESQUEMA | DBNAME     | TSNAME       |
|-------------|---------|------------|--------------|
| BALANCETE   | FINADM  | DBFINANCA  | TS_CONTABIL  |
| LANCAMENTOS | FINADM  | DBFINANCA  | TS_CONTABIL  |

📚 **Referência:** [SYSTABLES - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=catalog-systables)

---

### ⏳ 6. Consultas que Monitoram Tempo de Execução

📌 **Objetivo:** Identificar as 10 últimas queries executadas com maior tempo de CPU, úteis para análise de gargalos.

```sql
SELECT SUBSTR(STMT_TEXT, 1, 80) AS STMT,
       ELAPSED_TIME_TOTAL / 1000 AS TEMPO_TOTAL_MS,
       CPU_TIME_TOTAL / 1000 AS CPU_TOTAL_MS,
       NUM_EXECUTIONS
  FROM SYSIBMADM.TOP_DYNAMIC_SQL
 ORDER BY ELAPSED_TIME_TOTAL DESC
 FETCH FIRST 10 ROWS ONLY;
```

📋 **Resultado Esperado:**

| STMT                                  | TEMPO_TOTAL_MS | CPU_TOTAL_MS | NUM_EXECUTIONS |
|---------------------------------------|----------------|--------------|----------------|
| SELECT * FROM FATURAMENTO WHERE ...   | 18891          | 10321        | 5              |
| DELETE FROM LOG_ACESSO WHERE ...      | 15422          | 9210         | 3              |
| UPDATE CLIENTES SET ...               | 14230          | 9100         | 2              |

📚 **Referência:** [TOP_DYNAMIC_SQL - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=views-top-dynamic-sql)

---

### 🔐 7. Objetos sem REORG desde Última Atualização

📌 **Objetivo:** Verificar quais objetos não passaram por REORG após sua última modificação. Pode indicar fragmentação e degradação de performance.

```sql
SELECT NAME AS TABELA,
       CREATOR AS ESQUEMA,
       STATS_TIMESTAMP,
       REORGLASTTIME,
       CASE
         WHEN REORGLASTTIME IS NULL THEN 'NUNCA'
         WHEN REORGLASTTIME < STATS_TIMESTAMP THEN 'REORG NECESSÁRIO'
         ELSE 'OK'
       END AS STATUS_REORG
  FROM SYSIBM.SYSTABLESPACESTATS
 ORDER BY REORGLASTTIME NULLS FIRST
 FETCH FIRST 10 ROWS ONLY;
```

📋 **Resultado Esperado:**

| TABELA        | ESQUEMA  | STATS_TIMESTAMP     | REORGLASTTIME       | STATUS_REORG     |
|---------------|----------|---------------------|---------------------|------------------|
| TRANSACOES    | FINADM   | 2024-11-10-14.00.00 | NULL                | NUNCA            |
| VENDAS_MENSAL | ERP      | 2025-01-05-08.00.00 | 2024-12-01-03.20.00 | REORG NECESSÁRIO |
| CLIENTES      | DBUSER01 | 2025-05-01-10.30.00 | 2025-05-01-10.35.00 | OK               |

📚 **Referência:** [SYSTABLESPACESTATS - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=catalog-systablespacestats)

---


# Consulta para Identificar Tabelas que Contêm um Determinado Campo no DB2 for z/OS

## 🎯 Objetivo

Localizar todas as **tabelas base** (tipo `'T'`) que possuem um ou mais **campos com nome específico ou padrão**, usando as tabelas de catálogo do DB2:

- `SYSIBM.SYSCOLUMNS`
- `SYSIBM.SYSTABLES`

---

## 🧠 Consulta Base (Campo Específico)

```sql
SELECT 
    C.TBCREATOR     AS TABLE_SCHEMA,
    C.TBNAME        AS TABLE_NAME,
    C.NAME          AS COLUMN_NAME,
    C.COLNO         AS COLUMN_POSITION,
    C.COLTYPE       AS DATA_TYPE,
    C.LENGTH        AS COLUMN_LENGTH
FROM SYSIBM.SYSCOLUMNS C
JOIN SYSIBM.SYSTABLES T
  ON C.TBCREATOR = T.CREATOR
 AND C.TBNAME    = T.NAME
WHERE T.TYPE = 'T'
  AND UPPER(C.NAME) = 'COD_CLIENTE'
ORDER BY C.TBCREATOR, C.TBNAME, C.COLNO;
```

---

## 🔁 Variação com Lista de Campos

```sql
WHERE UPPER(C.NAME) IN ('COD_CLIENTE', 'ID_CLIENTE', 'CPF')
```

---

## 🔍 Variação com Padrão `LIKE`

```sql
WHERE UPPER(C.NAME) LIKE '%ID%'
```

---

## ⚙️ Otimizações Recomendadas

- **Filtrar por schemas específicos** (opcional):
  ```sql
  AND C.TBCREATOR IN ('SCHEMA1', 'SCHEMA2')
  ```

- **Amostragem controlada**:
  ```sql
  FETCH FIRST 50 ROWS ONLY
  ```

- **Evite LIKE com coringa à esquerda** (`%nome`) se não houver índice.

---

## 📦 Exemplo de Resultado Esperado

| TABLE_SCHEMA | TABLE_NAME | COLUMN_NAME | COLUMN_POSITION | DATA_TYPE | COLUMN_LENGTH |
|--------------|------------|-------------|------------------|-----------|----------------|
| CLIENTES     | PESSOA     | COD_CLIENTE | 1                | INTEGER   | 4              |
| CLIENTES     | CONTRATO   | COD_CLIENTE | 3                | INTEGER   | 4              |

---

## 🧩 Tabelas Envolvidas

- `SYSIBM.SYSCOLUMNS`: Contém metadados das colunas.  
- `SYSIBM.SYSTABLES`: Contém metadados das tabelas, incluindo tipo (`T`, `V`, etc.).

---

## ✅ Uso Recomendado

Essa consulta é ideal para:

- Levantamento de impacto antes de alteração de campos.
- Refatoração de tabelas.
- Análise de padronização de nomenclatura.
- Identificação de redundâncias ou uso indevido de campos.

---
---

## ✅ Boas Práticas de Execução

- Sempre utilize `WITH UR` em ambientes produtivos para evitar locks desnecessários.
- Valide estatísticas com `RUNSTATS` antes de confiar nos valores de `CARDF`.
- Use *`FETCH FIRST n ROWS ONLY`* quando for necessário limitar amostras com performance.
- Personalize consultas substituindo nomes genéricos (ex: `'TS_CONTABIL'`, `'DBFINANCA'`) conforme seu ambiente.

---


## 📚 Referências

- [IBM Db2 for z/OS Documentation](https://www.ibm.com/docs/en/db2-for-zos/)
- [Retrieving catalog information about foreign keys - IBM](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=design-retrieving-catalog-information-about-foreign-keys)
- [Db2 12 - Application programming and SQL - Examples of recursive common table expressions](https://www.ibm.com/docs/en/db2-for-zos/12?topic=expression-examples-recursive-common-table-expressions)
- [IBM Db2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=manuals-sql-reference)
- [Db2 Catalog Tables - Overview](https://www.ibm.com/docs/en/db2-for-zos/12.0?topic=catalog-catalog-tables)
- [Db2 z/OS - RUNSTATS Best Practices](https://www.ibm.com/docs/en/db2-for-zos/12.0?topic=utilities-runstats-statistics-collection)

---

🚀 *Este módulo faz parte do Manual Avançado para DBA DB2 for z/OS. Se desejar expandir ainda mais com scripts de automação, alertas ou integração com repositórios Git e dashboards, é só chamar!*

