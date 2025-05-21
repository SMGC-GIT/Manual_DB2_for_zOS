# 📘 Utilitários do DB2 for z/OS

Este material detalha os principais utilitários do DB2 for z/OS utilizados em ambientes corporativos, com explicações técnicas, modelos de JCL e links para documentação oficial da IBM. O objetivo é fornecer um guia completo, técnico e prático para DBAs de desenvolvimento e produção.

---

## ⚙️ RUNSTATS

### 🧩 O que é
`RUNSTATS` coleta estatísticas de distribuição e cardinalidade sobre os dados de tabelas e índices.

### 🎯 Para que serve
Atualiza informações no catálogo do DB2 (`SYSIBM.SYSTABLES`, `SYSINDEXES`, `SYSCOLUMNS`, etc.), usadas pelo otimizador para gerar planos de acesso eficientes.

### 🕓 Quando usar
- Após grandes cargas de dados
- Após `REORG`
- Após `LOAD` ou `DELETE` em massa
- Antes de EXPLAINs críticos

### 🚨 Situações que exigem uso
- Planos de acesso ineficientes
- `SQLCODE -805` e degradação de desempenho
- Desatualização das estatísticas no catálogo

### 💻 Modelo de JCL

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
`REORG` reorganiza fisicamente os dados nas tabelas e índices.

### 🎯 Para que serve
- Elimina espaços vazios (páginas de dados fragmentadas)
- Reordena os dados com base em índices clustering
- Melhora performance de acesso e evita full scans

### 🕓 Quando usar
- Após muitas exclusões/inserções
- Quando a fragmentação estiver alta
- Após `LOAD RESUME`
- Após degradação de desempenho constatada via EXPLAIN

### 🚨 Situações que exigem uso
- `SQLCODE -904` (resource unavailable: reorg pending)
- Tabela marcada como `REORGP` ou `AREO*` no catálogo
- Longos tempos de resposta em consultas com cláusula WHERE simples

### 💻 Modelo de JCL

```jcl
//REORG    JOB ...
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
DSN SYSTEM(DB2A)
REORG TABLESPACE DB2DB01.TSCLIENTE 
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
`LOAD` insere dados em massa a partir de datasets externos (como arquivos sequenciais).

### 🎯 Para que serve
- Popular tabelas rapidamente, sem overhead de logging
- Otimizado para grandes volumes

### 🕓 Quando usar
- Carga inicial de tabelas
- Substituição completa de conteúdo
- Migração entre ambientes

### 🚨 Situações que exigem uso
- Cargas de sistemas legados (batch)
- Ambiente de testes/preparação de massa de dados
- Inicialização de tabelas novas

### 💻 Modelo de JCL

```jcl
//LOADJOB  JOB ...
//STEP1    EXEC PGM=DSNUTILB,REGION=0M,
//         PARM='DB2A,LOADUTIL'
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD UNIT=SYSDA,SPACE=(CYL,(5,5))
//SORTOUT  DD UNIT=SYSDA,SPACE=(CYL,(5,5))
//SORTWK01 DD UNIT=SYSDA,SPACE=(CYL,(5,5))
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
`UNLOAD` exporta dados de tabelas DB2 para datasets externos.

### 🎯 Para que serve
- Geração de backups lógicos
- Transferência entre ambientes
- Extração de dados para cargas futuras

### 🕓 Quando usar
- Antes de alterações destrutivas (como `LOAD REPLACE`)
- Para migrações ou replicações manuais
- Como parte de auditorias

### 🚨 Situações que exigem uso
- Montagem de dados em ambientes de desenvolvimento
- Exportação parcial ou total de tabelas
- Necessidade de salvar o estado da tabela antes de testes

### 💻 Modelo de JCL

```jcl
//UNLDJOB  JOB ...
//STEP1    EXEC PGM=DSNUTILB,REGION=0M,
//         PARM='DB2A,UNLOADUTIL'
//SYSPRINT DD SYSOUT=*
//SYSREC00 DD DSN=MY.OUTPUT.FILE,DISP=(NEW,CATLG,DELETE),
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
`COPY` realiza o backup físico de tablespaces e indexspaces.

### 🎯 Para que serve  
- Gerar backups consistentes dos objetos de banco
- Proteger contra falhas e quedas de sistema

### 🕓 Quando usar  
- Em janelas noturnas ou em backups regulares
- Antes de alterações de alto risco

### 🚨 Situações que exigem uso  
- Planejamento de recovery
- Estratégia de disaster recovery
- Obrigatório antes de aplicar alterações em produção

### 💻 Modelo de JCL

```jcl
//COPYJOB  JOB ...
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,COPYUTIL'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  COPY TABLESPACE DB2DB01.TSCLIENTE
       COPYDDN(CPY001)
       SHRLEVEL REFERENCE
/*
//CPY001   DD DSN=MY.BACKUP.TSCLIENTE,DISP=(NEW,CATLG),
//         UNIT=SYSDA,SPACE=(CYL,(100,20))
// 
```

### 📚 Referência IBM  
[COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-copy-utility)

---

## ⚙️ RECOVER

### 🧩 O que é  
`RECOVER` restaura objetos de banco de dados a partir de imagens `COPY` e logs.

### 🎯 Para que serve  
- Recuperar tabelas danificadas
- Restaurar o estado de dados em um ponto específico

### 🕓 Quando usar  
- Após falhas físicas ou lógicas
- Após erros de usuário que causam perda de dados

### 🚨 Situações que exigem uso  
- Tabela em estado `RECP`
- `SQLCODE -904` com indicação de resource unavailable
- Situação de desastres e rollback de alterações

### 💻 Modelo de JCL

```jcl
//RECOVJOB JOB ...
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,RECOVUTIL'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  RECOVER TABLESPACE DB2DB01.TSCLIENTE
       TOCOPY MY.BACKUP.TSCLIENTE
/*
//
```

### 📚 Referência IBM  
[RECOVER - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-recover-utility)

---

## ⚙️ CHECK

### 🧩 O que é  
`CHECK` valida consistência entre tabelas e índices.

### 🎯 Para que serve  
- Verificar integridade lógica de dados
- Confirmar se índices refletem o conteúdo da tabela corretamente

### 🕓 Quando usar  
- Após `REORG` ou `LOAD`
- Após recovery
- Em auditorias ou validações pós-job

### 🚨 Situações que exigem uso  
- Erros de consistência detectados em aplicação
- Suspeita de corrupção lógica
- Tabelas com anomalias em acesso

### 💻 Modelo de JCL

```jcl
//CHKJOB   JOB ...
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,CHECKUTIL'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  CHECK DATA TABLESPACE DB2DB01.TSCLIENTE
  CHECK INDEX (ALL)
  CHECK LOB TABLESPACE DB2DB01.TSCLIENTE
/*
//
```

### 📚 Referência IBM  
[CHECK - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-check-utility)

---

## ⚙️ MODIFY RECOVERY

### 🧩 O que é  
`MODIFY RECOVERY` limpa entradas antigas do catálogo (`SYSCOPY`) e datasets de recovery desnecessários.

### 🎯 Para que serve  
- Manter o catálogo enxuto
- Prevenir degradação em utilitários que leem `SYSCOPY`

### 🕓 Quando usar  
- Periodicamente, como parte da administração preventiva
- Após muitos backups acumulados

### 🚨 Situações que exigem uso  
- Lentidão em execução de `COPY`, `RECOVER`
- Catálogo com registros antigos de recovery desnecessários

### 💻 Modelo de JCL

```jcl
//MODRECOV JOB ...
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2A,MODUTIL'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  MODIFY RECOVERY TABLESPACE DB2DB01.TSCLIENTE
         AGE(30)
/*
//
```

### 📚 Referência IBM  
[MODIFY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-modify-recovery-utility)

---

## ⚙️ DSN1COPY

### 🧩 O que é  
'DSN1COY' Utilitário não gerenciado diretamente pelo DB2. Faz cópia física de datasets (VSAM) de objetos do DB2.

### 🎯 Para que serve  
- Copiar tabelaspaces ou indexspaces fora do controle do DB2
- Clonagem manual de objetos
- Diagnóstico técnico com suporte IBM

### 🕓 Quando usar  
- Em ambientes de testes
- Em recuperação de dados com suporte IBM

### 🚨 Situações que exigem uso  
- Cópia de objetos de um banco para outro manualmente
- Recuperação não padrão com suporte IBM

### 💻 Modelo de JCL

```jcl
//COPYDSN1 JOB ...
//STEP1    EXEC PGM=DSN1COPY
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=SOURCE.TSCLIENTE,DISP=SHR
//SYSUT2   DD DSN=TARGET.TSCLIENTE,DISP=SHR
```

### 📚 Referência IBM  
[DSN1COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-dsn1copy-utility)

---

## ⚙️ QUIESCE

### 🧩 O que é  
`QUIESCE` grava ponto de consistência no log DB2.

### 🎯 Para que serve  
- Criar ponto de recuperação
- Marcar momento específico antes de alterações em massa

### 🕓 Quando usar  
- Antes de `REORG`, `LOAD`, `COPY`
- Como parte de procedimento de migração de dados

### 🚨 Situações que exigem uso  
- Garantia de consistência para utilitários subsequentes
- Auditoria e integridade de ambiente

### 💻 Modelo de JCL

```jcl
//QUIESCED JOB ...
//STEP1    EXEC PGM=IKJEFT01
//SYSTSPRT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSTSIN  DD *
DSN SYSTEM(DB2A)
QUIESCE TABLESPACE DB2DB01.TSCLIENTE
END
/*
//
```

### 📚 Referência IBM  
[QUIESCE - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-quiesce-utility)

---

## ⚙️ DIAGNOSE

### 🧩 O que é  
'DIAGNOSE' Utilitário técnico usado apenas sob orientação da IBM. Examina páginas físicas de dados.

### 🎯 Para que serve  
- Diagnóstico profundo de problemas em estruturas internas
- Suporte avançado em problemas críticos

### 🕓 Quando usar  
- Nunca usar por conta própria
- Apenas quando solicitado pela IBM

### 🚨 Situações que exigem uso  
- Corrupção de página detectada
- Pointers inválidos em índices
- Falha no acesso a datasets

### 💻 Modelo de JCL

```jcl
//DIAGNOSE JOB ...
//STEP1    EXEC PGM=DSN1LOGP
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=MY.DB2.LOG,DISP=SHR
```

### 📚 Referência IBM  
[DIAGNOSE - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-diagnose-utility)

---

## 🧰 MERGECOPY

### 🧩 O que é 
'MERGECOPY' Utilitário que combina múltiplas imagens de backup incrementais (`COPY`) para gerar uma única imagem consolidada.

### 🎯 Para que serve   
- Evita manter várias cópias incrementais em disco ou fita, facilitando e acelerando o processo de recuperação.

### 🕓 Quando usar
- Após a execução repetida de `COPY SHRLEVEL CHANGE` com `INCREMENTAL YES`.

### 🚨 Situações que exigem uso 
- Ambientes com muitas alterações e que utilizam cópias incrementais com frequência. Também útil quando o tempo de recuperação precisa ser minimizado.

### 💻 Modelo de JCL
```jcl
//MERGECPY JOB (ACCT),'MERGE COPY',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DSN,UID='MERGECOPY'
//SYSIN    DD *
  MERGECOPY TABLESPACE DBNAME.TSNAME
/*
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD UNIT=SYSDA,SPACE=(CYL,(10,5))
//SORTOUT  DD UNIT=SYSDA,SPACE=(CYL,(10,5))
```

### 📚 Referência IBM
[MERGECOPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-mergecopy-utility)
 

---

## 📋 REPORT RECOVERY

### 🧩 O que é 
'REPORT RECOVERY' Utilitário que gera um relatório com todas as cópias e logs necessários para recuperar um objeto do DB2.

### 🎯 Para que serve 
- Permite verificar previamente se os recursos necessários para uma recuperação estão disponíveis.

### 🕓 Quando usar  
- Antes de realizar um `RECOVER`, em auditorias de backup ou em testes de disaster recovery.

### 🚨 Situações que exigem uso  
- Verificação de estratégia de backup, preparação para recuperação, conformidade com políticas de segurança.

### 💻 Modelo de JCL
```jcl
//REPRECVY JOB (ACCT),'REPORT RECOVERY',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DSN,UID='REPRECVY'
//SYSIN    DD *
  REPORT RECOVERY TABLESPACE DBNAME.TSNAME
/*
//SYSPRINT DD SYSOUT=*
```

### 📚 Referência IBM
[RECOVERY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-report-recovery-utility)


---

## 📘 REPORT TABLESPACESET

### 🧩 O que é   
'REPORT TABLESPACESET' Gera um relatório com todos os objetos que compõem um conjunto de `TABLESPACE`.

### 🎯 Para que serve   
- Mapeia as dependências entre objetos, especialmente em estratégias de recuperação em cascata.

### 🕓 Quando usar  
- Antes de `RECOVER TABLESPACESET`, para entender quais objetos estão envolvidos.

### 🚨 Situações que exigem uso 
- Ambientes com muitos relacionamentos entre `TABLESPACE`, `INDEXSPACE`, e `AUX TABLES`.

### 💻 Modelo de JCL
```jcl
//REPTSPS  JOB (ACCT),'REPORT TS SET',CLASS=A,MSGCLASS=X
//STEP1 EXEC DSNUPROC,SYSTEM=DSN,UID='REPTSPS'
//SYSIN    DD *
  REPORT TABLESPACESET TABLESPACE DBNAME.TSNAME
/*
//SYSPRINT DD SYSOUT=*
```

### 📚 Referência IBM
[REPORT TABLESPACESET - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-report-tablespaceset-utility)


---

## 🪵 DSN1LOGP

### 🧩 O que é  
'DSN1LOGP' Utilitário stand-alone usado para imprimir registros de log do DB2 em formato legível.

### 🎯 Para que serve  
- Diagnóstico avançado de logs para análise de falhas, rastreamento de transações e suporte técnico.

### 🕓 Quando usar  
- Em situações de corrupção, falhas de integridade ou análise detalhada de RBA/LRSN.

### 🚨 Situações que exigem uso  
- Solicitação do suporte IBM, auditorias, falhas não explicadas, análise forense.

### 💻 Modelo de JCL
```jcl
//LOGP JOB (ACCT),'DSN1LOGP',CLASS=A,MSGCLASS=X
//STEP1 EXEC PGM=DSN1LOGP
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=DB2.ACTIVE.LOG
//SYSIN    DD *
  STARTRBA(X'0000000000000000')
  ENDRBA(X'FFFFFFFFFFFFFFFF')
/*
```

### 📚 Referência IBM
[DSN1LOGP - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1logp)


---

## 💾 DSN1COPY

### 🧩 O que é   
'DSN1COPY' Utilitário stand-alone que permite copiar datasets de `TABLESPACE` ou `INDEXSPACE` diretamente em nível físico.

### 🎯 Para que serve  
- Usado para restaurar dados fora do controle do DB2 ou para clonar datasets entre ambientes.

### 🕓 Quando usar  
- Quando o objeto não pode ser recuperado por métodos padrão (`RECOVER`) ou para duplicar dados de forma bruta.

### 🚨 Situações que exigem uso  
- Corrupção, falhas de catalogação, clones entre ambientes.

### 💻 Modelo de JCL
```jcl
//COPYJOB  JOB (ACCT),'DSN1COPY',CLASS=A,MSGCLASS=X
//STEP1 EXEC PGM=DSN1COPY
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=BACKUP.TS.DSN
//SYSUT2   DD DISP=SHR,DSN=DB2.TS.DSN
```

### 📚 Referência IBM
[DSN1COPY - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1copy)


---

## 🔍 DSN1PRNT

### 🧩 O que é    
'DSN1PRNT' Utilitário de diagnóstico que imprime o conteúdo de páginas de dados em hexadecimal e ASCII.

### 🎯 Para que serve  
- Analisa conteúdo físico dos datasets para diagnóstico de corrupção ou inconsistência.

### 🕓 Quando usar  
- Quando páginas estão com erro e o conteúdo precisa ser validado em nível físico.

### 🚨 Situações que exigem uso  
- Corrupção de dados, falha em páginas específicas, validação técnica junto ao suporte.

### 💻 Modelo de JCL
```jcl
//PRNTJOB  JOB (ACCT),'DSN1PRNT',CLASS=A,MSGCLASS=X
//STEP1 EXEC PGM=DSN1PRNT
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=DB2.TS.DSN
```

### 📚 Referência IBM
[DSN1PRNT - IBM Documentation](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-dsn1prnt)


---

