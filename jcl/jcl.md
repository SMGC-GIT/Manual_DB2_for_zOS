# 💼 JCL Essencial para DBA DB2 for z/OS

## 🎯 Objetivo

Este guia tem como objetivo fornecer uma base sólida e prática sobre JCL voltado ao dia a dia de um DBA, com foco em execução de utilitários, controle de jobs, manipulação de datasets e entendimento de execução no JES.

---

## 📂 1. Estrutura Básica de um Job JCL

```jcl
//NOMEJOB  JOB (ACCT),'DESCRIÇÃO',CLASS=A,MSGCLASS=X,REGION=0M
//ETAPA1   EXEC PGM=IEFBR14
//DD1      DD DSN=MEU.ARCHIVE.FILE,DISP=(NEW,CATLG,DELETE),
//             UNIT=SYSDA,SPACE=(TRK,(1,1)),DCB=(RECFM=FB,LRECL=80,BLKSIZE=800)
```

| Elemento     | Descrição                                         |
|--------------|---------------------------------------------------|
| `//NOMEJOB`  | Início do job (nome até 8 caracteres)             |
| `JOB`        | Identifica o job ao sistema JES                   |
| `EXEC`       | Executa um programa ou utilitário                 |
| `DD`         | Define os arquivos de entrada/saída utilizados    |
| `IEFBR14`    | Programa nulo, usado para alocar ou deletar arquivos |

---

## 📌 2. Tipos Comuns de DISP (Disposição de Dataset)

| DISP                  | Significado                          | Quando usar                          |
|-----------------------|--------------------------------------|--------------------------------------|
| `NEW`                 | Cria novo dataset                    | Criação de arquivos intermediários   |
| `OLD`                 | Usa dataset com exclusividade        | Leitura/escrita exclusiva            |
| `SHR`                 | Acesso compartilhado                 | Leitura compartilhada                |
| `MOD`                 | Append no dataset                    | Acrescentar logs, histórico          |
| `(NEW,CATLG,DELETE)`  | Novo, cataloga se OK, apaga se erro  | Situação padrão                      |

---

## 🛠️ 3. Utilitários com JCL

### ✅ Exemplo de RUNSTATS com JCL

```jcl
//JOBNAME  JOB ...
//RUNSTATS EXEC DSNUPROC,SYSTEM=DSN1,UID='USR01'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
RUNSTATS TABLESPACE DB01.TS01 TABLE(OWNER.TABELA1)
         INDEX(ALL) UPDATE ALL
/*
```

### ✅ Exemplo de REORG

```jcl
//REORG   EXEC DSNUPROC,SYSTEM=DSN1,UID='REORG01'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
REORG TABLESPACE DB01.TS01 LOG YES SHRLEVEL CHANGE
/*
```

---

## 🧩 4. Parâmetros Importantes

| Parâmetro     | Descrição                                       |
|---------------|-------------------------------------------------|
| `REGION=0M`   | Usa toda a memória disponível                   |
| `CLASS=A`     | Prioridade do job (varia por política local)    |
| `MSGCLASS=X`  | Classe de saída (X: spool JES, A: impressora)   |

---

## 🔍 5. Visualização e Execução no JES

| Comando    | Descrição                                     |
|------------|-----------------------------------------------|
| `S JOBNAME`| Submete job no SDSF ou painel de comandos     |
| `ST`       | Mostra status de execução dos jobs            |
| `?`        | Ajuda com descrições rápidas                  |
| `SJ`       | Edita e submete novamente um job listado      |

---

## 🔁 6. Parâmetros de Substituição (Simbolics)

```jcl
//STEP01 EXEC PGM=MYPGM,PARM='&PARM1'
//STEPLIB DD DSN=&DSLIB,DISP=SHR
```

- Utilizados em PROCs, INCLUDEs e ambientes padronizados.

---

## 📎 7. INCLUDE Statements

```jcl
// INCLUDE MEMBER=MEUJCL
```

> Útil para centralizar trechos reutilizáveis (ex: DDs padrões, execuções repetidas etc.)

---

## 🧠 8. Dicas Avançadas para DBA

| Dica                                                     | Explicação                                                |
|----------------------------------------------------------|------------------------------------------------------------|
| Use `IEFBR14` para alocar arquivos temporários sem rodar nada | Programa fictício que "não faz nada"                      |
| Combine `COND` ou `IF/THEN/ELSE` para controle entre steps     | Evita execução de etapas após falha                       |
| Prefira `SYSOUT=*` para debug rápido em testes                | Evita criar datasets físicos para logs temporários        |
| Use `SPACE=(TRK,(X,Y))` adequado ao tipo e volume de dados    | Previne abends por falta de espaço                        |

---

## 📚 Referências Oficiais IBM

- 🔗 [IBM JCL Reference Guide](https://www.ibm.com/docs/en/zos/2.5.0?topic=language-job-control)
- 🔗 [IBM z/OS DFSMS Using Data Sets](https://www.ibm.com/docs/en/zos/2.5.0?topic=sets-using-data)
- 🔗 [Db2 Utilities Reference](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-utility-statements)

---



