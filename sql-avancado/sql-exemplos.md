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

## 📌 Notas Finais

- Substitua `'SEU_ESQUEMA'`, `'SUA_TABELA'`, `'SEU_DATABASE'` e `'SEU_TABLESPACE'` pelos valores correspondentes ao seu ambiente.
- As tabelas do catálogo `SYSIBM` fornecem informações detalhadas sobre a estrutura e os relacionamentos do banco de dados.
- Utilize essas consultas como base para scripts de auditoria, documentação e manutenção do ambiente DB2.

---

## 📚 Referências

- [IBM Db2 for z/OS Documentation](https://www.ibm.com/docs/en/db2-for-zos/)
- [Retrieving catalog information about foreign keys - IBM](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=design-retrieving-catalog-information-about-foreign-keys)
- [Db2 12 - Application programming and SQL - Examples of recursive common table expressions](https://www.ibm.com/docs/en/db2-for-zos/12?topic=expression-examples-recursive-common-table-expressions)

