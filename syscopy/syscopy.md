## 🧾 Capítulo Especial: SYSCOPY no DB2 for z/OS

---

### 📘 Visão Geral

A tabela catálogo `SYSIBM.SYSCOPY` é o **registro oficial e histórico** das operações executadas sobre os objetos do banco de dados no DB2 for z/OS.

Ela é fundamental para:

- 🔍 Auditoria de cópias, recoveries e reorganizações
- ♻️ Automatização de limpeza e retenção de dados
- 📦 Identificação de backups válidos para recuperação
- ⚙️ Validação de políticas de integridade de dados

---

### 🧱 Estrutura Essencial da Tabela SYSCOPY

| Coluna        | Tipo          | Descrição                                                                 |
|---------------|---------------|---------------------------------------------------------------------------|
| `DBNAME`      | CHAR(8)       | Nome do banco de dados                                                   |
| `TSNAME`      | CHAR(8)       | Nome do tablespace                                                       |
| `ICTYPE`      | CHAR(1)       | Tipo da operação registrada                                              |
| `STYPE`       | CHAR(1)       | Subtipo (quando aplicável)                                              |
| `TIMESTAMP`   | TIMESTAMP     | Data/hora da execução                                                    |
| `DSNUM`       | SMALLINT      | Número da partição (0 = todas)                                           |
| `DSNAME`      | VARCHAR(44)   | Dataset resultante (ex: imagem de backup)                               |
| `DUMPID`      | VARCHAR(7)    | Código do DUMP, se houver                                               |
| `ICTS`, `ICSPACE`, `ICTIME`   | Informações complementares                                               |

> Use SYSCOPY para entender tudo que aconteceu com o objeto ao longo do tempo.

---

### 🧩 Operações Registradas na SYSCOPY

A tabela SYSCOPY **não registra apenas utilitários** — ela registra **eventos relacionados ao ciclo de vida dos dados**.

#### ✅ Categorias de registros:

| Categoria           | Exemplos de Registro (ICTYPE) |
|---------------------|-------------------------------|
| **Cópias**          | F, I, P, S, E                 |
| **Recuperações**    | R                             |
| **Reorganizações**  | Y, z, q                       |
| **Carga e Extração**| L, U                          |
| **Check**           | C                             |
| **Quiesce**         | Q                             |
| **Merge**           | M                             |
| **Modificações**    | A                             |
| **Falhas/Erros**    | T, B, Z                       |

---

### 🔠 Lista Completa de ICTYPE/STYPE

| ICTYPE | STYPE | Significado                                                     |
|--------|--------|-----------------------------------------------------------------|
| F      |        | COPY FULL                                                       |
| I      |        | COPY INCREMENTAL                                                |
| P      |        | COPY com SHRLEVEL CHANGE                                        |
| S      |        | COPY para objetos LOB                                           |
| E      |        | COPY inline feita por LOAD/REORG SHRLEVEL REFERENCE            |
| R      |        | RECOVER executado                                               |
| M      |        | MERGECOPY executado                                             |
| Q      |        | QUIESCE executado                                               |
| L      |        | LOAD com LOG NO                                                 |
| U      |        | UNLOAD                                                          |
| C      | D      | CHECK DATA                                                      |
| C      | L      | CHECK LOB                                                       |
| A      |        | MODIFY RECOVERY                                                 |
| T      |        | Falha em utilitário                                             |
| B      |        | COPY de imagem do CATÁLOGO                                      |
| Y      |        | REORG com LOG NO                                                |
| Z      |        | COPY removida com MODIFY RECOVERY                               |
| q      |        | COPY inline (LOAD ou REORG com LOG NO)                          |
| z      |        | Remoção de COPY inline                                          |

---

### 🛠️ Consultas Essenciais

#### 1. 📦 Últimos backups por objeto

```sql
SELECT DBNAME, TSNAME, ICTYPE, STYPE, TIMESTAMP, DSNAME
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('F','I','P','S','E')
ORDER BY TIMESTAMP DESC;
```

---

#### 2. 📈 Histórico completo de um objeto

```sql
SELECT ICTYPE, STYPE, TIMESTAMP, DSNAME
FROM SYSIBM.SYSCOPY
WHERE DBNAME = 'DBX'
  AND TSNAME = 'TSCLIENT'
ORDER BY TIMESTAMP DESC;
```

---

#### 3. 🧹 Identificar entradas antigas

```sql
SELECT *
FROM SYSIBM.SYSCOPY
WHERE TIMESTAMP < CURRENT DATE - 30 DAYS;
```

---

#### 4. 🧾 Consultar entradas de REORG com LOG NO

```sql
SELECT DBNAME, TSNAME, ICTYPE, TIMESTAMP
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('Y', 'q');
```

---

#### 5. 🧼 Entradas removidas por MODIFY

```sql
SELECT *
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('Z', 'z');
```

---

### 📊 Dashboard SQL: Monitoramento e Auditoria SYSCOPY

#### 🔎 Obter visão geral dos últimos eventos por tipo

```sql
SELECT ICTYPE, STYPE, COUNT(*) AS QTD, MAX(TIMESTAMP) AS MAIS_RECENTE
FROM SYSIBM.SYSCOPY
GROUP BY ICTYPE, STYPE
ORDER BY MAIS_RECENTE DESC;
```

---

#### 🧓 Objetos com cópias mais antigas

```sql
SELECT DBNAME, TSNAME, MIN(TIMESTAMP) AS BACKUP_MAIS_ANTIGO
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('F','I','P')
GROUP BY DBNAME, TSNAME
ORDER BY BACKUP_MAIS_ANTIGO;
```

---

#### 🧾 Objetos sem backup recente (últimos 15 dias)

```sql
SELECT DBNAME, TSNAME, MAX(TIMESTAMP) AS ULTIMO_BACKUP
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('F','I','P')
GROUP BY DBNAME, TSNAME
HAVING MAX(TIMESTAMP) < CURRENT DATE - 15 DAYS
ORDER BY ULTIMO_BACKUP;
```

---

#### 📦 Último backup por partição

```sql
SELECT DBNAME, TSNAME, DSNUM, MAX(TIMESTAMP) AS BACKUP_PART
FROM SYSIBM.SYSCOPY
WHERE ICTYPE IN ('F','I')
GROUP BY DBNAME, TSNAME, DSNUM
ORDER BY BACKUP_PART DESC;
```

---

### 📚 Boas Práticas

- 🔄 Use `MODIFY RECOVERY` para manter o SYSCOPY limpo
- ✅ Use `REPORT RECOVERY` para validar dependências antes de deploys
- 🧠 Monitore objetos com REORG LOG NO, pois removem backups anteriores
- 📊 Automatize consultas acima com alertas de auditoria

---

### 📌 Referências Úteis

- SYSCOPY:  
  [IBM Docs - SYSCOPY Table](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-syscopy-table)

- REPORT RECOVERY:  
  [IBM Docs - REPORT RECOVERY Utility](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=utilities-report-recovery)

---

## ✅ Conclusão

O SYSCOPY é mais que um catálogo: é o **diário oficial de eventos utilitários** do DB2.  
Dominar suas colunas, interpretar seus registros e automatizar sua análise é o que define um DBA de excelência.

> **"SYSCOPY conta a história que os logs não mostram."**
