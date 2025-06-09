# 💼 JCL Essencial para DBA DB2 for z/OS

## 🎯 Objetivo

Este guia tem como objetivo fornecer uma base sólida e prática sobre JCL voltado ao dia a dia de um DBA, com foco em execução de utilitários, controle de jobs, manipulação de datasets e entendimento de execução no JES.

---
## 🧩 Seção: JCL Básico - Parte 1

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

## 🧩 Seção: JCL Básico - Parte 2

### 📘 Tópicos Abordados:
1. Como funciona o fluxo de execução de um JOB
2. O papel das DD statements
3. Tipos de datasets (SEQ, PDS, VSAM, SYSOUT)
4. Entendendo o STEPLIB e JOBLIB
5. Utilização correta de parâmetros como DISP, SPACE, DCB
6. Controle de Condições (COND e IF/THEN/ELSE)
7. Explicações simples e eficazes para iniciantes e iniciados

---

### 🔹 1. 🚦 Fluxo de execução de um JOB

Um JOB é composto por:
- Uma **declaração de JOB** (`//NOMEJOB JOB ...`)
- Um ou mais **steps de execução** (`//STEP1 EXEC PGM=`)
- Cada step pode conter várias **instruções DD**, que definem arquivos, parâmetros, dispositivos, etc.

```jcl
//RELATORIO JOB (1234),'RELATÓRIO MENSAL',CLASS=A,MSGCLASS=X
//STEP1     EXEC PGM=RELAT001
//ENTRADA   DD DSN=EMPRESA.ENTRADA.DADOS,DISP=SHR
//SAIDA     DD SYSOUT=A
```

---

### 🔹 2. 📂 Instruções DD (Data Definition)

As `DD` statements associam **arquivos/datasets** a programas.

```jcl
//ARQENTR  DD DSN=MEU.ARQUIVO.ENTRADA,DISP=SHR
//ARQSAIDA DD DSN=MEU.ARQUIVO.SAIDA,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(TRK,(10,5),RLSE),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=800)
```

- **DSN**: Nome do dataset
- **DISP**: Status (NEW, OLD, SHR, MOD) + ações
- **SPACE**: Alocação (TRK, CYL, PRIMARY/SECONDARY)
- **DCB**: Parâmetros físicos (formato, tamanho)

---

### 🔹 3. 🧾 Tipos de datasets

| Tipo      | Descrição                               | Exemplo                          |
|-----------|------------------------------------------|----------------------------------|
| SEQ       | Sequencial                               | `DSN=MEU.DADO.SEQ`               |
| PDS       | Particionado (múltiplos membros)         | `DSN=PROGRAMA.SOURCE(MAIN)`      |
| VSAM      | Acesso direto (KSDS, ESDS, RRDS)         | `DSN=CLIENTES.KSDS`              |
| SYSOUT    | Impressão de saída                       | `SYSOUT=A`                       |

---

### 🔹 4. 📌 JOBLIB vs STEPLIB

- **JOBLIB**: válido para **todos os steps**
- **STEPLIB**: válido apenas para **o step atual**

```jcl
//JOBLIB   DD DSN=LOAD.MEUSPROGRAMAS,DISP=SHR
```

```jcl
//STEP2    EXEC PGM=PROGRAMA2
//STEPLIB  DD DSN=OUTRO.LOAD.LIB,DISP=SHR
```

---

### 🔹 5. 📐 Parâmetros DISP, SPACE, DCB

#### DISP: Controle de uso

```jcl
DISP=(NEW,CATLG,DELETE)
```
- **NEW**: novo dataset
- **CATLG**: catalogar se o step terminar com sucesso
- **DELETE**: excluir em caso de falha

#### SPACE

```jcl
SPACE=(CYL,(5,2),RLSE)
```
- 5 cilindros primários, 2 secundários
- `RLSE` libera o espaço não utilizado

#### DCB

```jcl
DCB=(RECFM=FB,LRECL=80,BLKSIZE=800)
```
- **RECFM**: formato físico (FB = Fixed Block)
- **LRECL**: tamanho lógico de cada registro
- **BLKSIZE**: tamanho do bloco de dados

---

### 🔹 6. 🔁 Controle Condicional (COND, IF/THEN)

#### ✅ Usando COND

```jcl
//STEP2 EXEC PGM=PGM2,COND=(4,LT,STEP1)
```
👉 Executa STEP2 **somente se** STEP1 tiver RC < 4

#### ✅ IF/THEN/ELSE

```jcl
//IFSTEP   EXEC PGM=NAOIMPORTA
// IF (STEP1.RC = 0) THEN
//PRINT    EXEC PGM=IMPRIMEOK
// ELSE
//PRINTERR EXEC PGM=IMPRIMEERRO
// ENDIF
```

---

### 🔹 7. 🔒 Outras boas práticas

| Prática                             | Vantagem                                                  |
|------------------------------------|------------------------------------------------------------|
| Sempre catalogar datasets novos    | Facilita reuso e manutenção                               |
| Usar SYSOUT para debug             | Ajuda a rastrear mensagens e falhas                       |
| Separar libraries por função       | Organização e segurança                                   |
| Usar `CLASS`, `MSGCLASS` otimizadas| Controla onde e como o job roda e onde a saída será gerada|

---

### 📎 Referências Oficiais

- [IBM JCL Language Reference (z/OS 2.5)](https://www.ibm.com/docs/en/zos/2.5.0?topic=reference-job-control-language)
- [IBM JCL User Guide](https://www.ibm.com/docs/en/zos/2.5.0?topic=zos-job-control-language-users-guide)

---

## 💻 Seção: JCL - Parte 3  
### Execução de Programas COBOL com DB2 (via IKJEFT01)

---

### 📘 Tópicos Abordados:
1. O que é o IKJEFT01
2. Como executar um programa COBOL que usa DB2
3. Parâmetros essenciais: DBRM, PLAN, STEPLIB
4. Utilização do DSN e RUN PROGRAM
5. Como tratar o retorno SQL e RC do JCL
6. Exemplo comentado de execução completa
7. Referências oficiais IBM

---

### 🔹 1. O que é IKJEFT01?

O IKJEFT01 é um programa **do ambiente TSO (Time Sharing Option)** que permite executar comandos TSO em batch. Ele é frequentemente utilizado para executar programas que interagem com o **DB2** através do utilitário **DSN**.

✅ Quando usamos IKJEFT01:
- Para rodar programas COBOL que usam SQL embutido (pré-compilados)
- Para executar comandos DB2 como BIND, REBIND, RUNSTATS, etc.

---

### 🔹 2. Estrutura de um JCL para rodar programa COBOL + DB2

```jcl
//RODASQL  JOB (1234),'EXECUTA DB2',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1    EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DSNLOAD.LIB.PRODUCAO,DISP=SHR
//         DD DSN=COBOL.LOAD.LIB,DISP=SHR
//SYSTSPRT DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2P)
  RUN PROGRAM(MINHAPGM) PLAN(MEUPLANO) -
      LIB('LOAD.LIB.PRODUCAO')
  END
```

---

### 🔹 3. Detalhamento dos componentes

| Componente    | Função                                                                 |
|---------------|------------------------------------------------------------------------|
| `PGM=IKJEFT01`| Chama o interpretador TSO em batch                                     |
| `SYSTSIN`     | Instruções que seriam digitadas no TSO (como DSN, RUN, END)           |
| `DSN SYSTEM()`| Inicia o ambiente DB2 conectado ao sistema (ex: DB2P ou DB2T)          |
| `RUN PROGRAM()`| Nome do programa COBOL pré-compilado e ligado                         |
| `PLAN()`      | Plano DB2 associado (criado via BIND do DBRM)                          |
| `LIB()`       | Biblioteca onde está o módulo LOAD do programa                         |
| `STEPLIB`     | Bibliotecas adicionais para localizar o módulo executável e utilitários|
| `SYSTSPRT`    | Saída de impressão do TSO (inclui mensagens DB2 e resultados SQL)      |

---

### 🔹 4. Pré-requisitos para o programa rodar corretamente

✅ Antes de executar o JCL acima, é necessário que:

- O programa COBOL tenha sido **pré-compilado com o DB2 precompiler**, gerando o **DBRM**.
- O **DBRM tenha sido BINDado** em um **PLAN** correspondente.
- O módulo **LOAD tenha sido gerado pelo linkage editor** e esteja disponível na biblioteca `LIB()` usada no RUN.

---

### 🔹 5. Interpretação de retornos

- **RC=0000** → Execução normal.
- **SQLCODE=0** → Sucesso SQL.
- **SQLCODE<0** → Erro SQL (ex: -904 = recurso indisponível).
- **SQLCODE>0** → Alerta (ex: +100 = fim de dados).

🔎 Os retornos SQL são mostrados em `SYSTSPRT`, que deve ser verificado com atenção.

---

### 🔹 6. Exemplo completo e comentado

```jcl
//EXECDB2  JOB (9999),'PROGRAMA DB2',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//*
//* STEP EXECUTA PROGRAMA COBOL COM SQL EMBUTIDO
//*
//PASSODB2 EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2P.DSNLOAD,DISP=SHR
//         DD DSN=EMPRESA.LOADLIB,DISP=SHR
//SYSTSPRT DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2P)
  RUN PROGRAM(PROGSQL1) PLAN(PLNSQL1) -
      LIB('EMPRESA.LOADLIB')
  END
```

💬 Comentários:
- `PROGSQL1`: nome do módulo gerado com o linkage editor
- `PLNSQL1`: plano DB2 já associado via BIND ao DBRM do programa
- `DB2P`: identificação do subsistema DB2 de produção
- `EMPRESA.LOADLIB`: biblioteca onde está o módulo executável

---

### 🔹 7. Erros comuns e soluções rápidas

| Erro                             | Causa provável                                    | Ação sugerida                       |
|----------------------------------|---------------------------------------------------|-------------------------------------|
| SQLCODE -805                     | DBRM não encontrado no PLAN                       | Verificar BIND e nome correto do PLAN |
| RC=12 ou RC=16 no JCL            | Falha no step / erro grave                        | Verificar `SYSTSPRT` e parâmetros   |
| ABEND S806                       | Programa não encontrado no LOADLIB                | Verificar STEPLIB ou LIB()          |
| SQLCODE -911 ou -913             | Deadlock ou timeout                               | Analisar locks e tempo de execução  |

---

### 📎 Referências Oficiais

- [IBM - Executando programas DB2 com IKJEFT01](https://www.ibm.com/docs/en/db2-for-zos/13?topic=applications-running-batch)
- [TSO/E Programming Services - IKJEFT01](https://www.ibm.com/docs/en/zos/2.5.0?topic=interfaces-ikjeft01)

---




