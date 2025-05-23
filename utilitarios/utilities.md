# 📘 Utilitários do DB2 for z/OS

Este material detalha os principais utilitários do DB2 for z/OS utilizados em ambientes corporativos, com explicações técnicas, modelos de JCL e links para documentação oficial da IBM. O objetivo é fornecer um guia completo, técnico e prático para DBAs de desenvolvimento e produção.

---

## 📘 Sumário de Utilitários

- [RUNSTATS](#runstats)
- [REORG](#reorg)
- [COPY](#copy)
- [RECOVER](#recover)
- [LOAD](#load)
- [UNLOAD](#unload)
- [CHECK DATA](#check-data)
- [CHECK LOB](#check-lob)
- [MODIFY RECOVERY](#modify-recovery)
- [QUIESCE](#quiesce)
- [DSN1COPY](#dsn1copy)
- [DSN1PRNT](#dsn1prnt)
- [DSN1LOGP](#dsn1logp)
- [DIAGNOSE](#diagnose)
- [MERGECOPY](#mergecopy)
- [REPORT RECOVERY](#report-recovery)
- [REPORT TABLESPACESET](#report-tablespaceset)

---

## RUNSTATS

### 🧩 O que é  
`RUNSTATS` é um utilitário do DB2 que coleta estatísticas de tabelas, índices e tablespaces, armazenando-as no catálogo para uso pelo otimizador.

### 🎯 Para que serve  
- Atualizar cardinalidade (`CARDF`), número de páginas e distribuição de valores nas colunas.  
- Fornecer dados precisos ao otimizador para escolher planos de acesso eficientes.

### 🕓 Quando usar  
- Após grandes cargas (`LOAD`), exclusões em massa ou reorganizações (`REORG`).  
- Antes de EXPLAIN ou REBINDs críticos.  
- Regularmente, conforme política de manutenção.

### 🚨 Situações que exigem uso  
- Planos de acesso ineficientes ou full table scan inesperados.  
- `SQLCODE -805` (plano não encontrado após estatísticas desatualizadas).  
- Desempenho degradado sem mudança de código.

### 💻 Modelo de JCL

```jcl
//RUNSTATS JOB (ACCT),'RUNSTATS',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2A)
  RUNSTATS TABLESPACE(DB2DB01.TSCLIENTE)
           TABLE(ALL) INDEX(ALL)
           FREQVAL NUMCOLS 5 COUNT 5
           HISTOGRAM
           REPORT YES
  END
/*
//
```

```jcl
//RUNSTATS JOB ...
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSTSIN  DD *
DSN SYSTEM(DB2A)
RUNSTATS TABLESPACE DB2DB01.TSCLIENTE 
         TABLE(OWNER.TB_CLIENTE) 
         INDEX(ALL) 
         UPDATE ALL
END
/*
//
```

### 🔍 Principais pontos sobre o RUNSTATS

- **Objetivo:**  
  Atualizar as estatísticas no catálogo do DB2 para auxiliar o otimizador na geração de planos de acesso eficientes. Essas estatísticas descrevem a distribuição de dados nas colunas, cardinalidade de tabelas e índices, número de páginas, entre outros.

- **Como atua internamente:**  
  - Coleta contadores de linhas e páginas (`CARDF`, `NPAGES`, `FPAGES`)
  - Pode coletar histogramas e frequências de valores em colunas (`FREQVAL`, `COLGROUP`)
  - Atualiza as tabelas do catálogo: `SYSIBM.SYSTABLES`, `SYSIBM.SYSCOLUMNS`, `SYSIBM.SYSINDEXES`, entre outras
  - Quando executado com a opção `UPDATE ALL`, sobrescreve todas as estatísticas relevantes

- **Modos de operação:**  
  - `TABLE`: estatísticas da tabela e suas colunas  
  - `INDEX`: estatísticas de índices  
  - `KEYCARD`: coleta cardinalidade de chaves compostas  
  - `FREQVAL n`: coleta os `n` valores mais frequentes  
  - `HISTOGRAM NUMQUANTILES n`: divide valores em quantis para melhor análise de distribuição

- **Implicações no desempenho:**  
  - Estatísticas desatualizadas podem levar o otimizador a escolher planos de acesso ineficientes (como full scans ou nested loops desnecessários)  
  - Após `LOAD`, `REORG`, ou grandes volumes de `DELETE`/`INSERT`, é altamente recomendado rodar o `RUNSTATS`

- **Boas práticas:**  
  - Sempre usar `SHRLEVEL CHANGE` em ambientes concorrentes  
  - Planejar coletas parciais (ex: apenas `INDEX`) para reduzir impacto  
  - Evitar uso excessivo de `FREQVAL` e `HISTOGRAM` sem necessidade real, pois aumentam o tempo de coleta

- **Limitações:**  
  - Não detecta estatísticas fora do escopo coletado  
  - Pode ser afetado por particionamento: se não executado com `REPORT NO PART`, ignora alguns dados relevantes  
  - Depende de amostragem (`SAMPLE`) quando especificado, o que pode gerar variações nos resultados

- **Interação com o otimizador:**  
  - Estatísticas alimentam o _Query Optimizer_ (RDS) para decidir entre uso de índices, _merge joins_, _nested loops_, etc  
  - Sem estatísticas ou com dados imprecisos, o DB2 pode gerar planos com alto custo estimado erroneamente

- **Riscos em ambientes críticos:**  
  - Se usado incorretamente em produção com `SHRLEVEL NONE`, pode causar indisponibilidade  
  - Substituição inadvertida de estatísticas manuais ou ajustadas pode causar regressões no desempenho



### 📚 Referência IBM  
[RUNSTATS - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-runstats-utility)

---

## ⚙️ REORG

### 🧩 O que é  
`REORG` reorganiza fisicamente os dados em tabelas e índices, removendo fragmentação e melhorando o acesso.

### 🎯 Para que serve  
- Compactar páginas com espaços vazios.  
- Reordenar dados segundo índices clustering.  
- Melhorar performance e reduzir I/O.

### 🕓 Quando usar  
- Após grandes volumes de deletes/inserts/updates.  
- Quando uma tabela ou índice está em estado `REORGP` ou `AREO*`.  
- Periodicamente, conforme monitoramento de performance.

### 🚨 Situações que exigem uso  
- `SQLCODE -904` (resource unavailable: reorg pending).  
- Full table scans constantes em tabelas fragmentadas.  
- Lentidão em queries simples com filtros diretos.

### 💻 Modelo de JCL

```jcl
//REORG    JOB (ACCT),'REORG',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  DSN SYSTEM(DB2A)
  REORG TABLESPACE(DB2DB01.TSCLIENTE)
        SORTDATA SORTKEYS STATISTICS
        SHRLEVEL CHANGE
  END
/*
//
```

### 🔍 Principais pontos sobre o REORG

- **Objetivo:**  
  O utilitário `REORG` reorganiza fisicamente os dados em uma tabela ou índice. Ele reordena os registros com base em chaves clustering, elimina espaços vazios e melhora a eficiência de acesso aos dados.

- **Como atua internamente:**  
  - Lê as páginas VSAM da tabela e regrava os dados em ordem de clustering (se houver índice clustering)
  - Elimina páginas parcialmente cheias ou completamente vazias, reduzindo `FPAGES` (número de páginas físicas)
  - Pode atualizar estatísticas básicas se utilizado com `STATISTICS`
  - Reorganiza estruturas de índice (`INDEXES`) se especificado
  - Pode ser executado online (`SHRLEVEL CHANGE/REFERENCE`) ou offline (`SHRLEVEL NONE`)

- **Modos de operação:**  
  - `SHRLEVEL CHANGE`: permite acesso simultâneo com mínima interferência  
  - `SHRLEVEL REFERENCE`: permite leitura, mas bloqueia updates  
  - `SHRLEVEL NONE`: exclusivo; ninguém acessa durante o REORG  
  - `REORG TABLESPACE` ou `REORG INDEX`: permite reorganizar somente as estruturas desejadas  
  - `SORTDATA` e `SORTKEYS`: controlam o uso de ordenação temporária, influenciando desempenho e uso de disco

- **Impactos positivos no desempenho:**  
  - Melhora a eficácia do _prefetch_ sequencial  
  - Reduz _getpages_ e _page reads_ em joins e varreduras  
  - Evita _page splits_ frequentes em índices  
  - Reduz espaço em disco ocupado por dados "órfãos" após `DELETE`

- **Situações que exigem execução do REORG:**  
  - Tabelas com muitas operações de `INSERT`/`DELETE` causando fragmentação  
  - Após execução de `LOAD RESUME`  
  - Quando a tabela ou índice está marcado com status `REORGP` ou `AREO*` no catálogo  
  - SQLCODE -904 com resource unavailable indicando reorganização pendente

- **Riscos e cuidados:**  
  - Execução com `SHRLEVEL NONE` pode causar indisponibilidade  
  - Quando mal programado, pode gerar _locking_ excessivo ou uso elevado de _sort work datasets_  
  - Tabelas muito grandes exigem planejamento e uso de partições, ou podem causar indisponibilidade prolongada  
  - Caso haja falhas de integridade, o objeto pode ficar em `CHECK PENDING` e requerer `CHECK DATA` antes de nova reorganização

- **Recomendações práticas:**  
  - Monitorar estatísticas como `NEARINDREF`, `NEAROFFPOSF` para avaliar necessidade de `REORG`  
  - Utilizar `REORGCHK` periodicamente para verificar se os critérios para reorganização são atingidos  
  - Agendar em janelas de baixa concorrência, exceto quando `SHRLEVEL CHANGE` for viável  
  - Sempre considerar executar `RUNSTATS` após um `REORG` completo para garantir precisão nos planos de acesso

- **Interações com outros utilitários:**  
  - Após `REORG`, estatísticas podem estar desatualizadas (exigir `RUNSTATS`)  
  - Se a tabela estava em `CHECK PENDING`, o `REORG` sozinho **não** resolve — é necessário `CHECK DATA`  
  - Pode ser encadeado com `COPY` para gerar imagem consistente pós-reorganização


### 📚 Referência IBM  
[REORG - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-reorg-utility)

---

## ⚙️ LOAD

### 🧩 O que é  
`LOAD` insere dados em massa em tabelas DB2 diretamente de datasets externos, reduzindo overhead de logging.

### 🎯 Para que serve  
- Popular tabelas rapidamente em cargas iniciais.  
- Substituir dados existentes (`REPLACE`).  
- Operações de migração ou testes.

### 🕓 Quando usar  
- Carga inicial de tabelas grandes.  
- Atualizações massivas quando o logging padrão é insuficiente.  
- Ambiente de desenvolvimento ou testes com dados de produção.

### 🚨 Situações que exigem uso  
- Tabelas vazias ou semi-vazias que precisam ser preenchidas rapidamente.  
- Necessidade de bypass do log para performance.  
- Uso de `UNLOAD` seguido de `LOAD` para ETL.

### 💻 Modelo de JCL

```jcl
//LOADJOB  JOB (ACCT),'LOAD',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSNUTILB,REGION=0M,  
//         PARM='DB2A,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD UNIT=SYSDA,SPACE=(CYL,(5,5))
//SORTOUT  DD UNIT=SYSDA,SPACE=(CYL,(5,5))
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC00)
       RESUME NO
       REPLACE
       INTO TABLE OWNER.TB_CLIENTE
       FIELDS TERMINATED BY ','
       (CODCLI POSITION(1:5) CHAR,
        NOME   POSITION(6:35) CHAR)
/*
//SYSREC00 DD DSN=MY.INPUT.FILE,DISP=SHR
```

### 🔍 Principais pontos sobre o LOAD

- **Objetivo:**  
  Realizar cargas em massa de dados externos para uma tabela DB2 com alta eficiência, substituindo ou adicionando registros diretamente ao dataset.

- **Modos de carga:**  
  - `REPLACE`: Remove todos os registros da tabela antes da carga.  
  - `RESUME YES`: Acrescenta os dados aos existentes.  
  - `RESUME NO`: Gera erro se a tabela já contiver dados.  

- **Formatos de entrada suportados:**  
  - `DELIMITED`: Arquivos delimitados por caractere (ex: CSV com `;`).  
  - `FIXED`: Formato de largura fixa por campo.  
  - `SPANNED`, `VARIABLE`, `VB`: Utilizados conforme o layout dos arquivos.

- **Parâmetros comuns:**  
  - `ENFORCE CONSTRAINTS`: Aplica constraints de integridade referencial.  
  - `KEEPDICTIONARY`: Mantém dicionário de compressão (caso tenha sido criado com `REORG`).  
  - `SHRLEVEL REFERENCE` / `CHANGE`: Define o nível de concorrência permitido durante a carga.  
  - `REUSE`: Reaproveita o espaço já alocado nos datasets.

- **Consequências pós-execução:**  
  - Cargas com `COPY NO` podem deixar a tabela em estado `COPY PENDING`.  
  - Recomenda-se executar um `COPY` após `LOAD` bem-sucedido para evitar pendências de recuperação.  
  - Necessidade de `REBUILD INDEX` ou `RUNSTATS` após carga em massa para garantir desempenho.

- **Cuidados especiais:**  
  - A cláusula `REPLACE` destrói os dados: use com máxima cautela.  
  - Incompatibilidades entre colunas da tabela e o layout do arquivo externo causam falhas.  
  - Arquivos com caracteres especiais podem exigir atenção à codificação (`CCSID`).  

- **Performance:**  
  - Consideravelmente mais rápida que `INSERT`.  
  - Otimiza operações em lote, especialmente com datasets organizados.

- **Observações gerais:**  
  - O `LOAD` ignora triggers e constraints opcionais durante a carga, dependendo das configurações.  
  - É ideal para processos de ETL, migração e refreshes periódicos de ambientes.


### 📚 Referência IBM  
[LOAD - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-load-utility)

---

## ⚙️ UNLOAD

### 🧩 O que é  
`UNLOAD` exporta dados de tabelas DB2 para datasets externos em formato legível ou binário.

### 🎯 Para que serve  
- Extração de dados para relatórios ou migração.  
- Gerar backups lógicos antes de operações destrutivas.  
- Preparação de dados para ETL.

### 🕓 Quando usar  
- Antes de executar `LOAD REPLACE`.  
- Para migração de dados entre ambientes.  
- Em processos de auditoria e análise.

### 🚨 Situações que exigem uso  
- Necessidade de exportar dados em massa.  
- Criação de snapshots lógicos de tabelas.  
- Testes de carregamento com dados reais.

### 💻 Modelo de JCL

```jcl
//UNLDJOB  JOB (ACCT),'UNLOAD',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSNUTILB,REGION=0M,  
//         PARM='DB2A,UNLOAD'
//SYSPRINT DD SYSOUT=*
//SYSREC00 DD DSN=MY.OUTPUT.FILE,  
//         DISP=(NEW,CATLG,DELETE),
//         UNIT=SYSDA,SPACE=(CYL,(10,5)),
//         DCB=(RECFM=FB,LRECL=80,BLKSIZE=0)
//SYSIN    DD *
  UNLOAD DATA FROM TABLE OWNER.TB_CLIENTE
       UNLDDN SYSREC00
       FORMAT DELIMITED
       WITH COLUMN NAMES
/*
//
```

### 🔍 Principais pontos sobre o UNLOAD

- **Objetivo:**  
  Extrair dados de uma ou mais tabelas DB2 em formatos estruturados, exportando-os para datasets sequenciais externos.

- **Modos de extração:**  
  - `DELIMITED`: Exporta dados com separadores por campo (ex: `;`, `|`, `,`).  
  - `FIXED`, `VARIABLE`: Exportações com formatos posicionais para sistemas legados.  
  - `SPANNED`: Permite registros maiores que o limite padrão de linha.

- **Parâmetros úteis:**  
  - `FIELDTERMINATOR`: Define o separador de campos.  
  - `SELECT`: Permite filtrar linhas (ex: `UNLOAD TABLE (...) SELECT * WHERE ...`).  
  - `MOD`: Permite chamada de rotinas externas para processar registros antes de gravar.

- **Casos de uso comuns:**  
  - Exportação de dados para outros ambientes ou bancos.  
  - Backup lógico de tabelas para possíveis cargas futuras via `LOAD`.  
  - Análise externa dos dados em ferramentas não-DB2 (Excel, Python, etc.).

- **Vantagens:**  
  - Alta flexibilidade de formato e compatibilidade.  
  - Permite extração completa ou parcial (com cláusulas WHERE).  
  - Produz datasets legíveis e manipuláveis externamente.

- **Cuidados necessários:**  
  - Não preserva estruturas como índices, constraints ou compressão.  
  - Sensível à codificação de caracteres (EBCDIC vs UTF-8).  
  - Volume pode ser elevado em tabelas grandes; prever alocação de espaço adequada.

- **Integração com LOAD:**  
  - Arquivos gerados por `UNLOAD` podem ser utilizados como entrada para `LOAD`.  
  - Deve-se manter compatibilidade de tipos e formatos entre exportação e importação.

- **Segurança e compliance:**  
  - Dados exportados podem conter informações sensíveis — aplicar criptografia quando necessário.  
  - Registrar operações de `UNLOAD` em trilhas de auditoria.


### 📚 Referência IBM  
[UNLOAD - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-unload-utility)

---

## ⚙️ COPY

### 🧩 O que é  
`COPY` é um utilitário do DB2 que faz backup físico de tablespaces e indexspaces, gerando imagens consistentes para recuperação.

### 🎯 Para que serve  
- Gerar cópias confiáveis de tabelaspaces/indexspaces  
- Garantir ponto de recuperação após falhas ou antes de manutenções de alto risco

### 🕓 Quando usar  
- Em janelas de backup programadas  
- Antes de operações de manutenção (REORG, LOAD, ALTER)  
- Em estratégias de disaster recovery

### 🚨 Situações que exigem uso  
- Preparação para RECOVER após falha  
- Cumprimento de políticas de backup  
- Segurança de dados antes de alterações críticas

### 💻 Modelo de JCL

```jcl
//COPYJOB  JOB (ACCT),'COPY TABLESPACE',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,COPY'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  COPY TABLESPACE(DB2DB01.TSCLIENTE)
       COPYDDN(CPY001)
       SHRLEVEL CHANGE
/*
//CPY001   DD DSN=MY.BACKUP.TSCLIENTE,DISP=(NEW,CATLG,DELETE),
//         UNIT=SYSDA,SPACE=(CYL,(50,5))
// 
```

### 🔍 Principais pontos sobre o COPY

- **Objetivo:**  
  O utilitário `COPY` é utilizado para gerar imagens de backup consistentes de objetos do banco de dados (tablespaces ou índices). Essas cópias são usadas posteriormente por utilitários como `RECOVER`, `MERGECOPY` ou até para auditorias e testes.

- **Modos de operação:**  
  - `COPY TABLESPACE`: copia os dados de um tablespace específico  
  - `COPY INDEXSPACE`: copia os dados de índices, principalmente quando se deseja backup de índices particionados  
  - `COPYDDN`/`COPYDDN2`: define os DDs onde as cópias serão gravadas  
  - `FULL YES`: faz backup completo (full image copy)  
  - `FULL NO`: realiza copy incremental, apenas páginas alteradas desde o último full  
  - `SHRLEVEL REFERENCE`: permite leitura simultânea durante o COPY  
  - `SHRLEVEL CHANGE`: permite alterações nos dados durante o COPY (usa logs para garantir consistência)

- **Tipos de cópia:**  
  - **Image Copy Full:** cópia completa de todas as páginas alocadas no objeto  
  - **Incremental Copy:** apenas páginas modificadas desde a última full  
  - **Inline Image Copy:** gerada automaticamente ao final de um `LOAD`, `REORG` ou `REBUILD INDEX` com a cláusula `COPY YES`

- **Importância estratégica:**  
  - Essencial para garantir capacidade de **RECOVER** eficiente e granular  
  - Pode ser usada para auditorias, ambiente de homologação ou replicações manuais  
  - Permite restaurar objetos sem depender de logs extensos (melhor RTO)

- **Cuidados e boas práticas:**  
  - Validar regularmente o catálogo `SYSIBM.SYSCOPY` para garantir que haja ao menos uma cópia full recente  
  - Planejar a retenção e volume das cópias — ocupam espaço proporcional ao tamanho do objeto  
  - Fazer `COPY` após operações destrutivas como `LOAD REPLACE`, `REORG`, ou alterações estruturais  
  - Para objetos críticos, usar `COPY SHRLEVEL CHANGE` para não impactar o sistema

- **Status no catálogo:**  
  - Cada execução gera entradas em `SYSCOPY` com o tipo de operação (`F`, `I`, `R`, etc.)  
  - Utilitário `REPORT RECOVERY` pode ser usado para identificar quais objetos têm cópia válida  
  - Se o objeto for excluído e recriado, as cópias anteriores perdem validade

- **Interações com outros utilitários:**  
  - **RECOVER** depende de cópias válidas — ausência de copy impede recuperação  
  - **MERGECOPY** pode consolidar várias cópias incrementais em uma full  
  - Pode ser usado em conjunto com `QUIESCE` para garantir pontos de consistência antes da cópia  
  - `COPY` após `REORG` é recomendável para ter backup do novo layout físico

- **Riscos e limitações:**  
  - Cópias com `SHRLEVEL CHANGE` exigem log para restauração completa  
  - Cópias inválidas (após `DROP` ou falhas) não são automaticamente removidas do catálogo  
  - Cópias muito antigas podem não ser utilizáveis se o log circular estiver desatualizado


### 📚 Referência IBM  
[COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-copy-utility)

---

## ⚙️ RECOVER

### 🧩 O que é  
`RECOVER` restaura tablespaces e indexspaces a partir de imagens de backup (`COPY`) e logs de transação, retornando-os a um estado consistente.

### 🎯 Para que serve  
- Restaurar dados após falhas físicas ou lógicas  
- Recuperar objetos até um ponto específico no tempo

### 🕓 Quando usar  
- Após falhas de I/O ou corrupção de dados  
- Em procedimentos de disaster recovery

### 🚨 Situações que exigem uso  
- TABELASPACES marcados como `RECOVER PENDING` no catálogo  
- `SQLCODE -904` (resource unavailable) indicando recovery pending  
- Situações emergenciais de restauração

### 💻 Modelo de JCL

### ✅ JCL: RECOVER TOCOPY

```jcl 
//RECOVC1  JOB (ACCT),'RECOVER TOCOPY',
//             CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP01   EXEC PGM=DSNUTILB,REGION=0M,
//             PARM='DB2P,RECOVC1'
//STEPLIB  DD DISP=SHR,DSN=DB2P.DSNLOAD
//SYSPRINT DD SYSOUT=*
//UTPRINT  DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DBNAME.TSNAME
    TOCOPY COPY01
/*
//*
// Nota:
// - "COPY01" deve corresponder ao nome do conjunto de cópia registrado no catálogo.
// - DBNAME.TSNAME representa o nome do banco de dados e da tablespace.
//
```


### ✅ JCL: RECOVER TORBA

```jcl
//RECOVRBA JOB (ACCT),'RECOVER TORBA',
//             CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP01   EXEC PGM=DSNUTILB,REGION=0M,
//             PARM='DB2P,RECOVRBA'
//STEPLIB  DD DISP=SHR,DSN=DB2P.DSNLOAD
//SYSPRINT DD SYSOUT=*
//UTPRINT  DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DBNAME.TSNAME
    TORBA X'0000000000001234567890AB'
/*
//*
// Nota:
// - Substitua o RBA (hexadecimal) com o valor exato fornecido pelo log ou suporte.
```


### ✅ JCL: RECOVER TOLOGPOINT

```jcl
//RECOVLOG JOB (ACCT),'RECOVER TOLOGPOINT',
//             CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP01   EXEC PGM=DSNUTILB,REGION=0M,
//             PARM='DB2P,RECOVLOG'
//STEPLIB  DD DISP=SHR,DSN=DB2P.DSNLOAD
//SYSPRINT DD SYSOUT=*
//UTPRINT  DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DBNAME.TSNAME
    TOLOGPOINT X'000000000000000123456789ABCD'
/*
//*
// Nota:
// - Use TOLOGPOINT quando souber o ponto exato no log fornecido por DSN1LOGP ou suporte.
```


### ✅ JCL: RECOVER TOCURRENT

```jcl
//RECOVCUR JOB (ACCT),'RECOVER TOCURRENT',
//             CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP01   EXEC PGM=DSNUTILB,REGION=0M,
//             PARM='DB2P,RECOVCUR'
//STEPLIB  DD DISP=SHR,DSN=DB2P.DSNLOAD
//SYSPRINT DD SYSOUT=*
//UTPRINT  DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DBNAME.TSNAME
    TOCURRENT
/*
//*
// Nota:
// - TOCURRENT aplica todas as alterações disponíveis até o último log. 
```


### ✅ JCL: RECOVER TOUTIL

```jcl
//RECOVUTL JOB (ACCT),'RECOVER TOUTIL',
//             CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP01   EXEC PGM=DSNUTILB,REGION=0M,
//             PARM='DB2P,RECOVUTL'
//STEPLIB  DD DISP=SHR,DSN=DB2P.DSNLOAD
//SYSPRINT DD SYSOUT=*
//UTPRINT  DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DBNAME.TSNAME
    TOUTIL LATEST
/*
//*
// Nota:
// - TOUTIL pode ser usado para retornar ao estado após uma utilidade específica (LOAD, REORG, etc.).
// - LATEST refere-se à última utilidade registrada no catálogo.
```

### 🔍 Principais pontos sobre o RECOVER

- **Objetivo:**  
  Restaurar objetos DB2 (tabelas, tablespaces, índices) a um estado consistente após falhas, exclusões acidentais ou testes destrutivos, utilizando backups e logs do sistema.

- **Tipos de recuperação:**  
  - `TOCOPY`: Recupera para o estado de um determinado image copy.  
  - `TORBA` / `TOLRSN`: Aponta uma posição específica no log (antes de uma falha).  
  - `TOLOGPOINT`: Ponto exato no log especificado manualmente.  
  - `TOCURRENT`: Aplica todas as atualizações possíveis até o fim dos logs disponíveis.  
  - `TOUTIL`: Restaura ao ponto de uma utilidade anterior (ex: após `LOAD`, `REORG`).  

- **Unidades de recuperação:**  
  - `TABLESPACE`: Mais comum. Restaura a tablespace inteira.  
  - `INDEXSPACE`: Para recuperação exclusiva de índices.  
  - `TABLE` (em ambientes específicos): Pode recuperar apenas uma tabela dentro da tablespace (requer logs detalhados e suporte adequado).  

- **Pré-requisitos:**  
  - Exigência de pelo menos uma cópia válida (`COPY`) para recuperação baseada em imagem.  
  - Logs arquivados e ativos devem estar disponíveis para aplicar alterações após a cópia.  
  - Permissões específicas: DBA ou autorizados com acesso a utilitários sensíveis.  

- **Modo de operação:**  
  - Apaga os dados atuais da tablespace (modo exclusivo).  
  - Restaura os dados da cópia (`COPY`, `MERGECOPY`, `DSN1COPY`).  
  - Reaplica todas as alterações de log (rollforward) até o ponto desejado.

- **Parâmetros comuns:**  
  - `AUTO` / `MANUAL`: Define se o DB2 tentará automaticamente encontrar cópias e logs.  
  - `SKIP LOCKED OBJECTS`: Evita falhas se algum objeto estiver indisponível.  
  - `LOGONLY`: Aplica apenas logs sem restaurar cópia (ex: testes ou pós-LOAD).  
  - `REBUILD INDEX`: Pode ser necessário após recuperação incompleta.

- **Situações comuns de uso:**  
  - Recuperar após falhas lógicas ou corrupção de dados.  
  - Reverter efeitos de operações destrutivas como `DELETE` ou `DROP`.  
  - Restaurar ambiente de homologação ou testes com base na produção.  

- **Cuidados e limitações:**  
  - Operação exclusiva: exige que nenhum outro job acesse o objeto durante o processo.  
  - `TOCOPY` não recupera alterações após a cópia — pode gerar perda de dados.  
  - Cópias inválidas ou ausência de logs podem inviabilizar a recuperação.  
  - Necessário entender bem o ponto no tempo desejado para evitar sobrescrever dados bons.

- **Performance e tempo:**  
  - Depende do tamanho da tablespace e quantidade de log a reaplicar.  
  - Backups frequentes + logs bem organizados reduzem o tempo de recuperação.

- **Validação e auditoria:**  
  - Após recuperação, recomenda-se:  
    - `CHECK INDEX` e `CHECK DATA`.  
    - `RUNSTATS` para atualização de estatísticas.  
    - Comparação com versões esperadas, quando possível.


### 📚 Referência IBM  
[RECOVER - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-recover-utility)

---

## ⚙️ CHECK DATA

### 🧩 O que é  
`CHECK DATA` é um utilitário do DB2 usado para validar as restrições de integridade referencial (constraints) em tabelas. Ele verifica a consistência dos dados conforme definido por chaves primárias, estrangeiras e restrições definidas.

### 🎯 Para que serve  
- Validar os relacionamentos entre tabelas (ex: foreign keys)
- Verificar se os dados de uma tabela seguem as regras de integridade definidas
- Remover o estado `CHECK PENDING (CHKP)` quando as validações são bem-sucedidas

### 🕓 Quando usar  
- Após utilizar `LOAD` com `ENFORCE NO`
- Após `REPAIR SET NOCOPYPEND`
- Após alteração de estrutura com impacto em constraints
- Quando uma tabela entra em estado `CHECK PENDING`

### 🚨 Situações que exigem uso  
- A tabela está inacessível para DML e marcada com `CHKP`
- É necessário validar os dados para restabelecer integridade
- Manutenção preventiva após cargas em massa

⚠️ **Importante:** Se houver dados que violam constraints, o utilitário **não remove o estado CHECK PENDING**. Nesse caso, é necessário **corrigir os dados manualmente** ou com ferramentas auxiliares (ex: SPUFI, QMF, UPDATE/DELETE).

---

### 📌 Principais pontos sobre o CHECK DATA

#### ✅ Objetivo

O utilitário `CHECK DATA` é usado para validar a **consistência referencial e restrições de integridade** dos dados em tabelas DB2, com base nas definições de:

- **Foreign keys (chaves estrangeiras)**
- **Constraints de integridade (PRIMARY KEY, UNIQUE, CHECK)**
- **Relacionamentos declarados no catálogo**

Ele verifica se os dados armazenados estão em conformidade com essas regras, especialmente após operações manuais, LOADs, UNLOAD/EDIT/RELOAD ou falhas de integridade que possam ter sido ignoradas.

---

#### 🛠️ Quando e por que utilizar

- Após o uso de **LOAD RESUME YES SHRLEVEL CHANGE** sem especificar `ENFORCE CONSTRAINTS`.
- Após **recuperações parciais de dados** onde relacionamentos entre tabelas possam ter sido corrompidos.
- Antes de liberar uma base para uso produtivo, especialmente em **migrações ou cargas em massa**.
- Como parte de rotinas periódicas de **auditoria e validação**.

---

#### 📂 Onde atua

- Em **tablespaces** e **tabelas individuais**.
- Pode verificar múltiplas tabelas em uma única execução.
- Verifica apenas **restrições declaradas** no catálogo DB2 — não checa regras implementadas via lógica de aplicação.

---

#### ⚙️ Como funciona

- Utiliza os metadados do catálogo (SYSRELS, SYSFOREIGNKEYS, etc.) para identificar relacionamentos e constraints.
- Percorre os dados de base e tenta encontrar inconsistências: por exemplo, registros órfãos, duplicados onde não deveriam existir, violações de CHECK constraints, etc.
- Gera saída detalhada no `SYSPRINT`, indicando quais registros e quais regras foram violadas.

---

#### 💡 Considerações importantes

- Não corrige dados — apenas **identifica violações**.
- Em ambientes produtivos, deve ser executado com cuidado, idealmente com SHRLEVEL REFERENCE (sem acesso concorrente).
- Pode ser **dispendioso em termos de I/O** se houver muitos relacionamentos complexos ou grande volume de dados.

---

#### 🧾 Parâmetros comuns

| Parâmetro        | Descrição |
|------------------|-----------|
| `TABLESPACE`     | Especifica a tablespace a ser verificada. |
| `TABLE`          | Lista tabelas específicas dentro da tablespace. |
| `FORCERULES`     | Força verificação mesmo que as regras estejam marcadas como desativadas. |
| `SHRLEVEL`       | Pode ser `REFERENCE` ou `CHANGE`. |
| `SCOPE`          | Define se a verificação é `ALL`, `CONSTRAINTS`, ou `CHECKS` apenas. |

---

#### 📋 Exemplo de uso (SYSIN)

```sql
CHECK DATA TABLESPACE DBNAME.TSNAME
  TABLE (TAB1, TAB2)
  SCOPE ALL
  SHRLEVEL REFERENCE
/*
```

---

#### ❗ Riscos e limitações

- Não identifica problemas **fora das regras declaradas**.
- Pode gerar **falsos positivos** se há inconsistências já conhecidas e toleradas pela aplicação.
- Pode falhar se tabelas estiverem com bloqueios ou problemas estruturais.

---

#### ✅ Conclusão

O `CHECK DATA` é um utilitário **essencial** para garantir que os dados estejam em conformidade com as regras de integridade declaradas no DB2. Embora não corrija dados, ele oferece uma **visão crítica da saúde relacional** do banco. Seu uso é fortemente recomendado após cargas, restaurações ou sempre que houver dúvida sobre a consistência dos dados. Ele faz parte do arsenal de controle de qualidade dos DBAs e pode ser integrado em **procedimentos de validação** automatizados em ambientes sensíveis.

---

### 💻 Modelo de JCL

```jcl
//CHECKDT JOB ...
//STEP1   EXEC PGM=DSNUTILB,PARM='DB2A'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  CHECK DATA TABLESPACE DB2DB01.TSCLIENTE
/*
//
```

### 📚 Referência IBM
[CHECK DATA - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-check-data-utility)

---

## 🧪 CHECK INDEX

O utilitário `CHECK INDEX` verifica a **integridade física e lógica dos índices** associados às tabelas DB2, garantindo que as estruturas de índice não estejam corrompidas e que apontem corretamente para os dados.

---

### 📌 Principais pontos sobre o CHECK INDEX

#### ✅ Objetivo

- Validar a **estrutura física dos índices**.
- Garantir que os índices estejam **consistentes com os dados** da tabela.
- Detectar problemas como páginas corrompidas, ponteiros inválidos ou dados inconsistentes no índice.

---

#### 🛠️ Quando e por que utilizar

- Após operações de manutenção que envolvam índices (REORG INDEX, DROP/CREATE).
- Quando ocorrerem erros ou falhas suspeitas relacionados a índices.
- Após restaurações ou recuperações para garantir a integridade dos índices.
- Como parte de auditoria para confirmar a saúde dos índices em ambientes críticos.

---

#### 📂 Onde atua

- Em índices individuais ou grupos de índices.
- Pode ser aplicado em índices primários e secundários.

---

#### ⚙️ Como funciona

- Percorre as páginas dos índices no tablespace de índices.
- Verifica estrutura B-tree (ou outro tipo) de cada índice.
- Confirma que os ponteiros estejam válidos e as folhas estejam corretamente encadeadas.
- Compara as entradas dos índices com os dados correspondentes, quando possível.

---

#### 💡 Considerações importantes

- Pode demandar tempo e I/O significativos em índices grandes.
- Não altera os dados ou índices, apenas gera relatório.
- Deve ser executado preferencialmente em janela de manutenção.

---

#### 🧾 Parâmetros comuns

| Parâmetro        | Descrição                       |
|------------------|--------------------------------|
| `INDEXSPACE`     | Especifica o tablespace de índice a ser verificado. |
| `INDEX`          | Nome(s) do(s) índice(s) a verificar. |
| `SHRLEVEL`       | `RELEASE` ou `CHANGE` para controle de concorrência. |

---

#### 📋 Exemplo de SYSIN

```sql
CHECK INDEX INDEXSPACE DBPROD.TSCLIENTE_IDX
    INDEX (IDX_CLIENTES_NOME)
    SHRLEVEL REFERENCE
/*
```

---

### 📄 JCL de exemplo

```jcl
//CHKIDX01 JOB (ACCT),'CHECK INDEX',CLASS=A,MSGCLASS=X,REGION=0M
//UTIL    EXEC PGM=DSNUTILB,REGION=0M,
//        PARM='DB2A'
//STEPLIB  DD DSN=DSNEXIT,DISP=SHR
//         DD DSN=DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
CHECK INDEX INDEXSPACE DBPROD.TSCLIENTE_IDX
    INDEX (IDX_CLIENTES_NOME)
    SHRLEVEL REFERENCE
/*
```

---

### 📚 Referência IBM
[CHECK INDEX - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-check-index-utility)

---

## 🧪 CHECK LOB

O utilitário `CHECK LOB` é usado para verificar a integridade física e lógica dos objetos LOB (Large Objects) armazenados em tabelas DB2, garantindo que os dados LOB não estejam corrompidos.

---

### 📌 Principais pontos sobre o CHECK LOB

#### ✅ Objetivo

- Validar a **estrutura física dos dados LOB** (BLOBs, CLOBs, etc.).
- Detectar corrupção em páginas de dados LOB.
- Garantir a consistência entre os ponteiros no registro da tabela e os dados LOB armazenados.

---

#### 🛠️ Quando e por que utilizar

- Após operações de manutenção que afetem tabelas com LOBs (ex: REORG).
- Quando forem detectados erros relacionados a LOBs durante consultas ou atualizações.
- Após falhas de sistema ou restaurações que possam comprometer dados LOB.
- Como parte de auditorias ou verificações periódicas de integridade.

---

#### 📂 Onde atua

- Trabalha sobre os tablespaces de LOB.
- Verifica os objetos LOB associados às tabelas específicas.

---

#### ⚙️ Como funciona

- Varre as páginas do tablespace LOB.
- Verifica estruturas internas e ponteiros usados para armazenar grandes objetos.
- Confirma que não haja páginas corrompidas ou dados inconsistentes.
- Gera relatório de inconsistências, caso existam.

---

#### 💡 Considerações importantes

- Pode ser um processo demorado para LOBs muito grandes ou numerosos.
- Não altera os dados, apenas reporta problemas.
- Deve ser usado com cautela em ambientes de produção, preferencialmente em janela de manutenção.

---

#### 🧾 Parâmetros comuns

| Parâmetro        | Descrição                              |
|------------------|---------------------------------------|
| `LOBSPACE`       | Especifica o tablespace de LOB a verificar. |
| `TABLE`          | Nome da tabela que contém os LOBs.    |
| `SHRLEVEL`       | Nível de concorrência durante a verificação (`CHANGE`, `REFERENCE`). |

---

#### 📋 Exemplo de SYSIN

```sql
CHECK LOB LOBSPACE DBPROD.TSCLIENTE_LOB
    TABLE DBPROD.CLIENTES
    SHRLEVEL CHANGE
/*
```

---

### 📄 JCL de exemplo

```jcl
//CHKLOB01 JOB (ACCT),'CHECK LOB',CLASS=A,MSGCLASS=X,REGION=0M
//UTIL    EXEC PGM=DSNUTILB,REGION=0M,
//        PARM='DB2A'
//STEPLIB  DD DSN=DSNEXIT,DISP=SHR
//         DD DSN=DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
CHECK LOB LOBSPACE DBPROD.TSCLIENTE_LOB
    TABLE DBPROD.CLIENTES
    SHRLEVEL CHANGE
/*
```

---

### 📚 Referência IBM
[CHECK LOB - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-check-lob-utility)

---

## ⚙️ MODIFY RECOVERY

### 🧩 O que é  
`MODIFY RECOVERY` limpa entradas antigas do catálogo (`SYSCOPY`) e remove imagens de backup desnecessárias.

### 🎯 Para que serve  
- Manter o catálogo enxuto  
- Melhorar performance de utilitários de recovery

### 🕓 Quando usar  
- Periodicamente, como manutenção preventiva  
- Após acúmulo de muitas cópias de backup

### 🚨 Situações que exigem uso  
- Lentidão em comandos COPY/RECOVER  
- Catálogo de recovery muito volumoso


## Principais pontos sobre MODIFY RECOVERY

### Objetivo
- Controlar e modificar o estado dos processos de recuperação em DB2.
- Permitir o cancelamento, continuação ou ajuste de operações de recuperação pendentes.
- Manipular flags internas relacionadas à recuperação de dados e logs.

### Quando e por que utilizar
- Quando um processo de recuperação foi interrompido ou está travado.
- Para limpar estados pendentes após falhas em RECOVER ou LOAD.
- Para ajustar e prosseguir com a recuperação sem reiniciar totalmente o processo.
- Durante troubleshooting ou sob orientação do suporte IBM.

### Como funciona
- Permite comandos para alterar o estado de recuperação de tabelas ou tablespaces.
- Usado para remover bloqueios que impedem novas operações.
- Controla flags internas indicando se a recuperação está pendente, em andamento ou abortada.
- Não altera dados, apenas o controle do processo de recuperação.

### Considerações importantes
- Deve ser usado com cautela, pois manipula estados internos do DB2.
- Requer conhecimento técnico detalhado ou suporte IBM.
- Uso incorreto pode deixar o banco inconsistente ou em estado inválido.


### 💻 Modelo de JCL

```jcl
//MODRECOV JOB (ACCT),'MODIFY RECOVERY',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,MODIFY'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  MODIFY RECOVERY TABLESPACE(DB2DB01.TSCLIENTE) AGE(30)
/*
//

```

### 📚 Referência IBM  
[MODIFY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-modify-recovery-utility)

---

## ⚙️ DSN1COPY

### 🧩 O que é  
`DSN1COPY` é um utilitário stand-alone que copia fisicamente datasets VSAM (LDS) de tablespaces ou indexspaces, sem passar pelo subsistema DB2. Ele trabalha em nível de dataset, replicando bit a bit o conteúdo.

### 🎯 Para que serve  
- Restaurar fisicamente objetos corrompidos quando o DB2 não consegue fazê-lo  
- Clonar datasets entre ambientes de teste, desenvolvimento e produção  
- Permitir cópias offline “brutas” antes de operações de manutenção de baixo nível

### 🕓 Quando usar  
- Se o utilitário `COPY` ou `RECOVER` do DB2 não resolver devido a corrupção de catálogo  
- Em cenários de suporte IBM em que é necessário acessar dados inacessíveis  
- Para manter snapshot físico exato de um tablespace antes de testes críticos

### 🚨 Situações que exigem uso  
- Catálogo DB2 sem reconhecimento do dataset (erro SQLCODE -904 “TSNAME is pending REQ”)  
- Falhas de I/O nos VSAM LDS que impedem utilitários padrão  
- Recuperação de datasets danificados externamente

### 🛠️ Parâmetros e Observações  
- **DSN1COPY**: não utiliza parâmetros DB2; todas as opções são definidas via DD statements  
- **SYSUT1**: dataset de origem (LDS do tablespace/indexspace)  
- **SYSUT2**: dataset de destino, deve ter o mesmo tamanho e atributos de DCB  
- **Recomendação**: use o mesmo DISP e atributos (RECFM, LRECL, BLKSIZE) para evitar truncamento  
- **Limitações**: não atualiza o catálogo DB2; após cópia, é preciso executar `ALTER TABLESPACE` + `REORG` para revalidar

## Principais pontos sobre DSN1COPY

### Objetivo
- Copiar blocos de dados físicos de um dataset VSAM para outro.
- Ferramenta de baixo nível usada para duplicar dados de tablespaces e índices no DB2.
- Frequentemente usada para restaurar dados de ambientes de teste ou para recuperação manual de falhas.

### Quando e por que utilizar
- Para restaurar dados a partir de backups "fora" do controle do DB2.
- Quando se precisa clonar uma tablespace ou índice para análise fora do ambiente de produção.
- Ao recuperar dados corrompidos de forma física, em testes de disaster recovery ou troubleshooting.

### Como funciona
- Atua no nível físico, copiando blocos de dados byte a byte.
- Requer que os datasets de origem e destino tenham exatamente o mesmo layout (incluindo formato e tamanho da página).
- Não atualiza catálogos DB2, logs ou headers – operação é externa ao controle do DB2.
- Geralmente utilizado junto com DSN1PRNT para inspecionar o conteúdo antes/depois da cópia.

### Considerações importantes
- Não é um utilitário lógico – **não valida conteúdo** e pode corromper dados se mal usado.
- Deve ser usado **apenas por DBAs experientes ou sob orientação IBM**.
- Após o uso, pode ser necessário executar `DSN1CHKR`, `CHECK DATA`, `REORG` ou `REBUILD INDEX` para restaurar a integridade.
- Ideal para uso em ambientes de teste, duplicação, e investigação de problemas físicos.

### Exemplos comuns de uso
- Clonagem de uma tablespace de produção para ambiente de testes.
- Restauração manual de dados após falha em backup lógico (COPY).
- Leitura de páginas danificadas via DSN1PRNT após cópia física.

### Alternativas
- `COPY`/`RECOVER` são preferíveis para operações lógicas de backup/restauração.
- `DSN1COPY` é útil quando estas alternativas não estão disponíveis ou não são viáveis.

### Aviso
- O uso incorreto de DSN1COPY **pode causar corrupção grave no banco de dados**.
- Sempre documente e teste previamente seu uso em ambientes seguros.


### 💻 Modelo de JCL

```jcl
//COPYDSN1 JOB (ACCT),'DSN1COPY',CLASS=A,MSGCLASS=X
//STEP1   EXEC PGM=DSN1COPY
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=ORIG.TSCLIENTE,  
//             UNIT=SYSDA,SPACE=(CYL,(50,0)),  
//             DCB=(RECFM=U,LRECL=4096,BLKSIZE=4096)
//SYSUT2   DD DISP=NEW,DSN=COPY.TSCLIENTE,  
//             UNIT=SYSDA,SPACE=(CYL,(50,0)),  
//             DCB=(RECFM=U,LRECL=4096,BLKSIZE=4096)
//SYSIN    DD DUMMY
```

### 📚 Referência IBM  
[DSN1COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-dsn1copy-utility)

---

## ⚙️ QUIESCE

### 🧩 O que é  
`QUIESCE` é um utilitário que cria um ponto de consistência (checkpoint) no subsistema DB2 sem interromper as aplicações ativas. Ele garante que todos os buffers modificados (dirty pages) sejam gravados no log, estabelecendo um ponto seguro para operações subsequentes.

### 🎯 Para que serve  
- Marcar um ponto de recuperação no log DB2 antes de `REORG`, `LOAD` ou `COPY`  
- Assegurar que não haja transações pendentes não registradas  
- Suportar estratégias de recuperação e migração online

### 🕓 Quando usar  
- Antes de executar utilitários que requerem quiesce (ex.: `REORG SHRLEVEL REFERENCE`)  
- No início e fim de grandes cargas de dados online  
- Durante procedimentos de upgrade ou migração de esquema

### 🚨 Situações que exigem uso  
- Garantir consistência de dados em backup incrementais  
- Evitar rollback de transações longas durante manutenção  
- Preparar o ambiente para operações de alta criticidade sem downtime

### 🛠️ Parâmetros e Observações  
- **DATABASE**: especifica o nome lógico do DB2 onde será gravado o checkpoint  
- **ALL**: opcional, para aplicar quiesce a todas as databases no subsistema  
- O utilitário não interfere nas aplicações, mantendo o acesso READ/WRITE  
- Após `QUIESCE`, utilitários como `REORG` poderão usar menor locking, pois há ponto de referência

---

## Principais pontos sobre o utilitário QUIESCE

### Escopos possíveis
- `QUIESCE TABLESPACE database.tablespace`
- `QUIESCE TABLESPACESET`
- `QUIESCE DATABASE`
- `QUIESCE TABLE`

### Parâmetros úteis
- `WRITE YES` ou `WRITE NO`: determina se será gerado um registro de log que pode ser utilizado pelo RECOVER.
- `SCOPE GLOBAL`: força quiesce de todos os membros do data sharing group.
- `FORCE`: força rollback das unidades de trabalho abertas, se necessário.

### Considerações importantes
- Não impede novas atualizações após a conclusão – apenas garante a consistência no instante do QUIESCE.
- Operação leve, mas pode aguardar unidades de trabalho demoradas se o parâmetro `FORCE` não for usado.
- Deve ser usada com cuidado em ambientes concorrentes para não interromper transações críticas.

### Exemplos comuns de uso
- Pré-requisito antes de copiar datasets do DB2 com ferramentas externas.
- Estabelecer ponto de recuperação consistente antes de atualizações massivas em produção.
- Como parte de scripts de manutenção ou automação de backup/restauração.

### Alternativas
- `COPY SHRLEVEL REFERENCE` também estabelece consistência, mas com maior controle sobre backup.
- `STOP` seguido de `COPY` é outra abordagem, mas mais intrusiva.

### Aviso
- O QUIESCE não copia ou salva dados – apenas estabelece um **marco lógico** no log para possível recuperação futura.


### 💻 Modelo de JCL

```jcl
//QUIESCE  JOB (ACCT),'QUIESCE',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2A)
  QUIESCE DATABASE(DB2DB01)
  END
/*
//SYSOUT   DD SYSOUT=*

```

### 📚 Referência IBM  
[QUIESCE - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-quiesce-utility)

---

## ⚙️ DIAGNOSE

### 🧩 O que é  
`DIAGNOSE` é um programa stand-alone de diagnóstico avançado do DB2, distribuído com módulos internos de suporte IBM, mas **não faz parte** do conjunto de utilities rotineiros para DBAs em produção. Não é documentado para uso geral e só está disponível quando solicitado pelo suporte IBM.

### 🎯 Para que serve  
- Examinar páginas de dados (VSAM LDS) e estruturas de índice em nível físico  
- Identificar corrupção de páginas (checksums inválidos, ponteiros corrompidos)  
- Coletar evidências detalhadas antes de substituir datasets  

### 🕓 Quando usar  
- **Somente sob orientação do suporte IBM**  
- Após erros irrecuperáveis em páginas de dados (SQLCODEs de nível I/O ou defeitos de integridade interna)  
- Quando diagnóstico via logs, IFCIDs ou `CHECK DATA` não for suficiente  

### 🚨 Situações que exigem uso  
- Erros de checksum detectados em páginas de dados  
- Ponteiros de índice inválidos causando falhas de acesso  
- Objetos marcados como corrompidos, sem solução por `RECOVER` ou `REORG`  

---

### 🔍 Principais pontos sobre o DIAGNOSE

- **Objetivo:** Examinar o conteúdo binário de páginas de dados e índices para identificar corrupção a nível físico.  
- **Por que o suporte IBM recomenda:**  
  - Após falhas graves de I/O ou corrupção não explicada  
  - Para coletar evidências antes de restaurar ou substituir datasets  
- **Como é invocado:**  
  Geralmente via JCL executando o programa `DSN1LOGP` (ou similar), informando RBA/LRSN e gravando dumps brutos.  
- **Riscos e limitações:**  
  - Não há rollback — apenas leitura de dados brutos  
  - Pode gerar enorme volume de informação sem utilidade se usado sem critério  
  - Não altera o catálogo DB2 nem os dados  

---

❓ **Conclusão:**  
Embora faça parte dos módulos de diagnóstico do DB2, o `DIAGNOSE` **não é** um utilitário de rotina como `RUNSTATS`, `REORG` ou `COPY`. Deve ser utilizado **apenas** sob instrução do suporte IBM para casos extremos de corrupção física.

---

### 💻 Modelo de JCL

```jcl
//DIAGJOB  JOB (ACCT),'DIAGNOSE',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSN1LOGP,REGION=0M
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=DB2.ACTIVE.LOG
//SYSIN    DD *
  STARTRBA(X'0000000000000000')
  ENDRBA(X'FFFFFFFFFFFFFFFF')
/*
```

### 📚 Referência IBM  
[DIAGNOSE - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-diagnose-utility)

---

## ⚙️ MERGECOPY

### 🧩 O que é  
`MERGECOPY` é um utilitário oficial do DB2 for z/OS que consolida múltiplas cópias incrementais de backup (`COPY INCREMENTAL`) em uma única imagem completa, armazenando o resultado como nova imagem de `SYSCOPY`.

### 🎯 Para que serve  
- Reduzir o número de imagens incrementais a serem gerenciadas  
- Facilitar e acelerar o processo de recuperação ao utilizar uma única imagem consolidada  
- Otimizar espaço em mídia de backup

### 🕓 Quando usar  
- Após várias execuções de `COPY SHRLEVEL CHANGE INCREMENTAL YES`  
- Periodicamente, para evitar acúmulo excessivo de imagens incrementais  
- Antes de um procedimento de recuperação que exija uma única imagem de restauração

### 🚨 Situações que exigem uso  
- Várias entradas `INCREMENTAL` para o mesmo objects no catálogo `SYSCOPY`  
- Tempo de recuperação muito elevado devido ao grande número de imagens a serem aplicadas  
- Políticas de retenção que exigem consolidação de backups

---

### 🔍 Principais pontos sobre o MERGECOPY

- **Objetivo:**  
  Consolidar backups incrementais em uma única imagem completa para simplificar `RECOVER`.

- **Por que usar:**  
  - Melhora a eficiência do `RECOVER`  
  - Reduz overhead de gerenciar múltiplas imagens  
  - Minimiza riscos de falha ao aplicar várias imagens

- **Como é invocado:**  
  Através de JCL chamando `DSNUPROC` (ou `DSNUTILB`) com parâmetro `MERGECOPY`, especificando o tablespace/indexspace-alvo e o nome do novo dataset de saída.

- **Riscos e limitações:**  
  - **Não altera** as imagens originais de backup; cria uma nova imagem  
  - Opera em nível de SYSCOPY — se o catálogo estiver inconsistente, pode falhar  
  - Requer espaço de disco/fita suficiente para armazenar a imagem consolidada

---

### 💻 Modelo de JCL

```jcl
//MERGECPY JOB (ACCT),'MERGECOPY',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IKJEFT01,REGION=0M
//SYSTSPRT DD SYSOUT=*
//SYSIN    DD *
  DSN SYSTEM(DB2A)
  MERGECOPY TABLESPACE(DB2DB01.TSCLIENTE)
           COPYDDN(CPYINCR1,CPYINCR2,CPYINCR3)
           OUTDDN(CPYFULL)
/*
//CPYINCR1 DD DSN=BK.INCR1.TSCLIENTE,DISP=SHR
//CPYINCR2 DD DSN=BK.INCR2.TSCLIENTE,DISP=SHR
//CPYINCR3 DD DSN=BK.INCR3.TSCLIENTE,DISP=SHR
//CPYFULL  DD DSN=BK.FULL.TSCLIENTE,                       
//             DISP=(NEW,CATLG,DELETE),
//             UNIT=SYSDA,SPACE=(CYL,(50,10))
//SYSOUT   DD SYSOUT=*

```

### 📚 Referência IBM
[MERGECOPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-mergecopy-utility)
 

---

## ⚙️ REPORT RECOVERY

### 🧩 O que é  
`REPORT RECOVERY` é um utilitário oficial do DB2 for z/OS que gera um inventário das imagens de backup (COPY) e arquivos de log necessários para recuperar um tablespace, indexspace ou conjunto de tablespaces (TABLESPACESET).

### 🎯 Para que serve  
- Listar todas as imagens de backup (FULL e incremental) e logs indispensáveis para um recovery bem-sucedido  
- Validar a consistência e completude das cópias antes de executar `RECOVER`  
- Auxiliar no planejamento de procedimentos de recuperação e auditorias de backup

### 🕓 Quando usar  
- Antes de executar `RECOVER` em produção  
- Em testes de disaster recovery e simulações de falha  
- Em auditorias de conformidade para demonstrar disponibilidade de backups

### 🚨 Situações que exigem uso  
- Validação prévia em cenários de recuperação emergencial  
- Políticas de segurança que exigem comprovação de backup  
- Quando múltiplos níveis de cópia (FULL + incrementais) precisam ser aplicados na ordem correta

---

### 🔍 Principais pontos sobre o REPORT RECOVERY

- **Objetivo:**  
  Garantir que todas as imagens e logs necessários estejam presentes e acessíveis antes de iniciar o `RECOVER`.

- **Como é invocado:**  
  Através de JCL chamando `DSNUPROC` ou `DSNUTILB` com o parâmetro `REPORT RECOVERY`, especificando o objeto a ser validado.

- **Saída:**  
  Um relatório detalhado, via SYSOUT, indicando para cada objeto:
  - Nome do dataset de backup  
  - Tipo de cópia (FULL, INCREMENTAL)  
  - Intervalo de logs necessário (start/stop LRSN ou RBA)  

- **Riscos e limitações:**  
  - Não altera dados nem o catálogo DB2  
  - Se faltar alguma imagem ou log, apontará erro mas não executa o recovery  
  - Depende de acesso correto às fitas ou volumes de disco com as imagens

---

### 💻 Modelo de JCL

```jcl
//REPRECOV JOB (ACCT),'REPORT RECOVERY',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=IKJEFT01,REGION=0M
//SYSTSPRT DD SYSOUT=*
//SYSIN    DD *
  DSN SYSTEM(DB2A)
  REPORT RECOVERY TABLESPACE(DB2DB01.TSCLIENTE
       COPYDDN(CPYFULL,CPYINCR1,CPYINCR2)
       LOGDDN(LOG1,LOG2)
/*
//CPYFULL  DD DSN=BK.FULL.TSCLIENTE,DISP=SHR
//CPYINCR1 DD DSN=BK.INCR1.TSCLIENTE,DISP=SHR
//CPYINCR2 DD DSN=BK.INCR2.TSCLIENTE,DISP=SHR
//LOG1     DD DSN=DB2.LOG.VOLUME1,DISP=SHR
//LOG2     DD DSN=DB2.LOG.VOLUME2,DISP=SHR
//SYSOUT   DD SYSOUT=*

```

### 📚 Referência IBM
[RECOVERY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-report-recovery-utility)


---

## ⚙️ REPORT TABLESPACESET

### 🧩 O que é  
`REPORT TABLESPACESET` é um utilitário oficial do DB2 for z/OS que gera um relatório dos objetos (tablespaces, indexspaces, LOB e aux tables) que compõem um conjunto lógico de tablespaces.

### 🎯 Para que serve  
- Mapear dependências entre tablespaces e seus objetos auxiliares  
- Auxiliar no planejamento de recuperação em cascata (TABLESPACESET)  
- Fornecer visão completa dos componentes que precisam ser incluídos em operações de backup ou recovery

### 🕓 Quando usar  
- Antes de executar `RECOVER TABLESPACESET`  
- Em auditorias de dependências de banco de dados  
- Durante planejamento de manutenção que impacte múltiplos tablespaces

### 🚨 Situações que exigem uso  
- Ambientes com diversos tablespaces inter-relacionados  
- Recuperação em cascata em que é preciso tratar todos os objetos dependentes  
- Migração ou clone de ambientes complexos

---

### 🔍 Principais pontos sobre o REPORT TABLESPACESET

- **Objetivo:**  
  Identificar todos os tablespaces, indexspaces, LOB tablespaces e aux tables que fazem parte de um TABLESPACESET.

- **Como é invocado:**  
  Via JCL chamando `DSNUPROC` ou `DSNUTILB` com o parâmetro `REPORT TABLESPACESET`, informando o tablespace principal.

- **Saída:**  
  Um relatório via SYSOUT listando:
  - Table spaces e index spaces  
  - LOB and aux tables  
  - Ordem de dependência

- **Riscos e limitações:**  
  - Operação somente de leitura, sem alterar o catálogo  
  - Se objetos estiverem em estado inconsistente, podem não ser listados corretamente  
  - Depende do catálogo estar atualizado (RUNSTATS) para refletir alterações recentes

---

### 💻 Modelo de JCL

```jcl
//REPTSPS JOB (ACCT),'REPORT TS SET',CLASS=A,MSGCLASS=X
//STEP1   EXEC PGM=IKJEFT01,REGION=0M
//SYSTSPRT DD SYSOUT=*
//SYSIN   DD *
  DSN SYSTEM(DB2A)
  REPORT TABLESPACESET TABLESPACE(DB2DB01.TSCLIENTE)
/*
//SYSOUT  DD SYSOUT=*  

```

### 📚 Referência IBM
[REPORT TABLESPACESET - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-report-tablespaceset-utility)


---

## ⚙️ DSN1LOGP

### 🧩 O que é  
`DSN1LOGP` é um programa stand-alone de diagnóstico do DB2 for z/OS, utilizado para ler e imprimir registros de log de transação em formato legível. Não faz parte do conjunto de utilitários rotineiros de DBA.

### 🎯 Para que serve  
- Analisar transações detalhadamente, registradas em logs do DB2  
- Diagnose de falhas críticas de I/O ou corrupção de dados  
- Fornecer informações granulares (RBA/LRSN) para suporte IBM  

### 🕓 Quando usar  
- **Somente sob orientação do suporte IBM**  
- Após abends relacionados a logs ou transações  
- Em investigações onde `EXPLAIN`, IFCIDs e utilitários padrão não forem suficientes  

### 🚨 Situações que exigem uso  
- Logs corrompidos ou inacessíveis por utilitários convencionais  
- Necessidade de rastrear operações específicas de RBA/LRSN  
- Diagnóstico avançado de deadlocks ou falhas de checkpoint  

---

### 🔍 Principais pontos sobre o DSN1LOGP

- **Objetivo:**  
  Imprimir registros de log brutos em formato interpretável, permitindo a reconstrução da sequência de transações.

- **Como é invocado:**  
  Através de JCL executando `DSN1LOGP`, informando o dataset de log e, opcionalmente, o intervalo de RBA/LRSN.

- **Saída:**  
  - Texto legível no SYSOUT com operações de insert/update/delete e checkpoints  
  - Informações de transações iniciadas, confirmadas ou abortadas

- **Riscos e limitações:**  
  - Volume de saída pode ser muito grande; use filtros de RBA/LRSN  
  - Não altera dados nem o catálogo DB2  
  - Exige conhecimento avançado para interpretar o dump de log

---

### ❓ Conclusão:
O DSN1LOGP não é um utilitário de rotina para DBAs, mas um programa de diagnóstico avançado disponibilizado via suporte IBM para analisar logs de transação em profundidade. Deve ser usado exclusivamente quando instruído pela IBM, para resolver casos críticos de corrupção ou falhas de log.

---

### 💻 Modelo de JCL

```jcl
//LOGPJOB  JOB (ACCT),'DSN1LOGP',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSN1LOGP
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=DB2.LOG.VOLUME1
//SYSIN    DD *
  STARTRBA(X'0000000000000000')
  ENDRBA(X'FFFFFFFFFFFFFFFF')
/*

```

### 📚 Referência IBM
[DSN1LOGP - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1logp)


---

## ⚙️ DSN1COPY

### 🧩 O que é  
`DSN1COPY` é um utilitário stand-alone do DB2 for z/OS que realiza cópia física bit a bit de datasets VSAM (LDS) de tablespaces ou indexspaces, independentemente do subsistema DB2.

### 🎯 Para que serve  
- Restaurar objetos corrompidos quando o catálogo DB2 não consegue reconhecê-los  
- Clonar tablespaces/indexspaces inteiros para ambientes de teste ou desenvolvimento  
- Fazer backups “brutos” offline antes de operações de manutenção de baixo nível  

### 🕓 Quando usar  
- Quando os utilitários `COPY` ou `RECOVER` padrão não conseguem acessar ou restaurar um objeto  
- Em situações de suporte IBM que exigem recuperação manual de datasets  
- Para obter um snapshot exato de um tablespace antes de aplicar patches ou upgrades críticos  

### 🚨 Situações que exigem uso  
- Catálogo DB2 inconsistente ou marcado como `RECOVER PENDING` sem solução via utilitários  
- Falhas de I/O em VSAM LDS que impedem operações convencionais  
- Necessidade de duplicar fisicamente um objeto para diagnósticos aprofundados  

---

### 🔍 Principais pontos sobre o DSN1COPY

- **Objetivo:**  
  Copiar fisicamente um dataset VSAM (LDS) de origem para destino, mantendo todos os atributos (DCB, tamanho, forma de gravação).

- **Como é invocado:**  
  Via JCL chamando o programa `DSN1COPY`, sem parâmetros DB2, utilizando DD statements para definir origem e destino.

- **Saída:**  
  - Um novo dataset idêntico ao original em nível físico  
  - Logs mínimos de atividade via SYSOUT  

- **Riscos e limitações:**  
  - Não atualiza o catálogo DB2; após a cópia, é necessário usar `ALTER TABLESPACE ... REORG` para reincluir o objeto no DB2  
  - Destino deve ter atributos de DCB compatíveis (RECFM, LRECL, BLKSIZE)  
  - Pode causar inconsistências se usado durante atividade normal de DB2, pois ignora locks  

---

### ❓ Conclusão:
Embora poderoso, o DSN1COPY deve ser usado com cautela e apenas em cenários de suporte avançado, quando os utilitários padrão de backup e recovery não forem suficientes. Ele não interage com o catálogo DB2 e, portanto, exige etapas adicionais (ex: REORG) para reintegração do objeto copiado.

---

### 💻 Modelo de JCL

```jcl
//COPYDSN1 JOB (ACCT),'DSN1COPY',CLASS=A,MSGCLASS=X
//STEP1   EXEC PGM=DSN1COPY
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=ORIG.TSCLIENTE,  
//           UNIT=SYSDA,SPACE=(CYL,(50,0)),  
//           DCB=(RECFM=U,LRECL=4096,BLKSIZE=4096)
//SYSUT2   DD DISP=(NEW,CATLG,DELETE),DSN=COPY.TSCLIENTE,  
//           UNIT=SYSDA,SPACE=(CYL,(50,0)),  
//           DCB=(RECFM=U,LRECL=4096,BLKSIZE=4096)
//SYSIN    DD DUMMY

```

### 📚 Referência IBM
[DSN1COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1copy)


---

### ⚙️ DSN1PRNT

### 🧩 O que é  
`DSN1PRNT` é um utilitário stand-alone do DB2 for z/OS usado para imprimir o conteúdo físico de datasets VSAM (LDS) de tablespaces ou indexspaces em formato hexadecimal e ASCII, permitindo análise forense de páginas de dados.

### 🎯 Para que serve  
- Diagnosticar corrupção de dados em nível de página  
- Verificar estrutura interna de páginas (planos de página, headers, registros)  
- Suportar investigação de falhas críticas de acesso a dados  

### 🕓 Quando usar  
- Sob orientação do suporte IBM, em casos de corrupção física detectada  
- Após erros de I/O reportados por SQLCODEs de nível de hardware  
- Quando outras ferramentas de diagnóstico não fornecerem detalhes suficientes  

### 🚨 Situações que exigem uso  
- Páginas marcadas como inválidas ou “bad” em logs de DB2/SMF  
- Mensagens de erro indicando checksum ou formato de página incorreto  
- Falha de utilitários padrão (`RECOVER`, `REORG`) sem causa aparente  

---

### 🔍 Principais pontos sobre o DSN1PRNT

- **Objetivo:**  
  Exibir o conteúdo bruto de páginas de dados para análise técnica de corrupção ou inconsistência.

- **Como é invocado:**  
  Via JCL executando o programa `DSN1PRNT`, fornecendo o dataset de origem e, opcionalmente, parâmetros de offset e comprimento.

- **Saída:**  
  - Texto no SYSOUT mostrando off-sets, códigos de página e dados em hexadecimal/ASCII  
  - Artefatos para análise manual ou por ferramentas de suporte

- **Riscos e limitações:**  
  - Gera grande volume de saída; use filtros de offset para focar em páginas específicas  
  - Não confere locks ou integridade de transações; apenas lê dados brutos  
  - Requer conhecimento avançado de estruturas VSAM e formato de página DB2  

---

### 💻 Modelo de JCL

```jcl
//PRNTJOB  JOB (ACCT),'DSN1PRNT',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSN1PRNT,REGION=0M
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=DB2.TSCLIENTE,  
//           UNIT=SYSDA,SPACE=(CYL,(50,0)),  
//           DCB=(RECFM=U,LRECL=4096,BLKSIZE=4096)
//SYSIN    DD *
  STARTPAGE(1) ENDPAGE(5)
  HEXASCII
/*
```

### 📚 Referência IBM
[DSN1PRNT - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1prnt)


---

