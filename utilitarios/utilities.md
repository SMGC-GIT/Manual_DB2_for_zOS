# 📘 Utilitários do DB2 for z/OS

Este material detalha os principais utilitários do DB2 for z/OS utilizados em ambientes corporativos, com explicações técnicas, modelos de JCL e links para documentação oficial da IBM. O objetivo é fornecer um guia completo, técnico e prático para DBAs de desenvolvimento e produção.

---

## ⚙️ RUNSTATS

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

```jcl
//RECOVJOB JOB (ACCT),'RECOVER',CLASS=A,MSGCLASS=X
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,RECOVER'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE(DB2DB01.TSCLIENTE)
         TOCOPY MY.BACKUP.TSCLIENTE
/*
//
```

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

