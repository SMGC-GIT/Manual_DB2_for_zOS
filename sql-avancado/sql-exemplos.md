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

## 📚 Referências

- [IBM Db2 for z/OS Documentation](https://www.ibm.com/docs/en/db2-for-zos/)
- [Retrieving catalog information about foreign keys - IBM](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=design-retrieving-catalog-information-about-foreign-keys)
- [Db2 12 - Application programming and SQL - Examples of recursive common table expressions](https://www.ibm.com/docs/en/db2-for-zos/12?topic=expression-examples-recursive-common-table-expressions)
- [IBM Db2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=manuals-sql-reference)
- [Db2 Catalog Tables - Overview](https://www.ibm.com/docs/en/db2-for-zos/12.0?topic=catalog-catalog-tables)
- [Db2 z/OS - RUNSTATS Best Practices](https://www.ibm.com/docs/en/db2-for-zos/12.0?topic=utilities-runstats-statistics-collection)

---

🚀 *Este módulo faz parte do Manual Avançado para DBA DB2 for z/OS. Se desejar expandir ainda mais com scripts de automação, alertas ou integração com repositórios Git e dashboards, é só chamar!*

