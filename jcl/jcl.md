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

## 💻 Seção: JCL - Parte 4  
### Execução de Utilitários DB2 no JCL (RUNSTATS, REORG, COPY, etc.)

---

### 📘 Tópicos Abordados:
1. O que são utilitários DB2 e para que servem
2. Visão geral do uso em JCL
3. Utilitário RUNSTATS (estatísticas)
4. Utilitário REORG (reestruturação física)
5. Utilitário COPY (backup)
6. Utilitário CHECK DATA (validação de integridade)
7. Utilitário LOAD/UNLOAD (carga e descarga de dados)
8. Cuidados, parâmetros importantes e retorno de execução
9. Referências IBM

---

### 🔹 1. O que são utilitários DB2?

Utilitários DB2 são **programas fornecidos pelo DB2** para executar tarefas administrativas ou operacionais, como:
- Atualizar estatísticas do catálogo
- Reorganizar tabelas
- Realizar backups (image copy)
- Validar integridade referencial
- Carregar ou extrair dados em massa

São executados geralmente em batch via **JCL + IKJEFT01** ou diretamente via **PGM=DSNUTILB**.

---

### 🔹 2. Visão geral do uso em JCL

A forma mais comum de executar utilitários DB2 é através do programa **DSNUTILB**, que interpreta comandos no formato utilitário, fornecidos na entrada (`SYSIN`).

---

### 🔹 3. Utilitário RUNSTATS

Atualiza estatísticas sobre tabelas e índices no catálogo do DB2, ajudando o otimizador de consultas.

```jcl
//RUNSTATS JOB (9999),'ESTATISTICAS',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2P'
//STEPLIB  DD DSN=DB2P.DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  RUNSTATS TABLESPACE DBNOME.TSNOME
           TABLE(ALL)
           INDEX(ALL)
           FREQVAL NUMCOLS 10
           HISTOGRAM
           REPORT YES
/*
```

📝 Parâmetros úteis:
- `TABLE(ALL)` → estatísticas de todas as tabelas
- `INDEX(ALL)` → inclui índices
- `FREQVAL`, `HISTOGRAM` → melhoram distribuição estatística para o otimizador

---

### 🔹 4. Utilitário REORG

Reorganiza fisicamente os dados das tabelas para eliminar fragmentação e melhorar performance.

```jcl
//REORGTAB JOB (9999),'REORG TABELA',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1    EXEC PGM=DSNUTILB,PARM='DB2P'
//STEPLIB  DD DSN=DB2P.DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  REORG TABLESPACE DBNOME.TSNOME
         LOG YES
         SHRLEVEL REFERENCE
         STATISTICS TABLE(ALL) INDEX(ALL)
/*
```

📌 Tipos de `SHRLEVEL`:
- `REFERENCE` → leitura permitida, mas sem escrita
- `CHANGE` → permite leitura e escrita (alta disponibilidade)

---

### 🔹 5. Utilitário COPY

Gera cópia física de tabelaspaces e índices, essencial para backup e recuperação.

```jcl
//COPYTS JOB (9999),'BACKUP DB2',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1  EXEC PGM=DSNUTILB,PARM='DB2P'
//STEPLIB  DD DSN=DB2P.DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=BACKUP.DBNOME.TSNOME(+1),DISP=(NEW,CATLG),
//            UNIT=SYSDA,SPACE=(CYL,(100,10)),DCB=DSORG=PS
//SYSIN    DD *
  COPY TABLESPACE DBNOME.TSNOME
       COPYDDN(SYSUT1)
/*
```

⚠️ É preciso configurar corretamente o `DD` de destino (`SYSUT1`) e definir GDG para manter versões do backup.

---

### 🔹 6. Utilitário CHECK DATA

Verifica se os dados de uma tabela respeitam regras de integridade referencial.

```jcl
//CHECKDT JOB (9999),'CHECK INTEGRIDADE',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1  EXEC PGM=DSNUTILB,PARM='DB2P'
//STEPLIB DD DSN=DB2P.DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  CHECK DATA TABLESPACE DBNOME.TSNOME
/*
```

🎯 Ajuda a identificar por que uma tabela está em `CHECK PENDING`.

---

### 🔹 7. Utilitário LOAD/UNLOAD

- **LOAD** → carga em massa de dados
- **UNLOAD** → exportação de dados

```jcl
//UNLOADT JOB (9999),'UNLOAD TABELA',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1 EXEC PGM=DSNUTILB,PARM='DB2P'
//STEPLIB DD DSN=DB2P.DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//UNLDOUT  DD DSN=EXPORT.DADOS.CLIENTE(+1),DISP=(NEW,CATLG),
//            SPACE=(CYL,(100,10)),UNIT=SYSDA
//SYSIN    DD *
  UNLOAD DATA FROM TABLE EMPRESA.CLIENTES
         OUTFILE(UNLDOUT)
/*
```

---

### 🔹 8. Cuidados e retorno de execução

| Situação                          | Causa provável                               | Solução sugerida                         |
|-----------------------------------|----------------------------------------------|-------------------------------------------|
| RC=04 no utilitário               | Avisos não críticos                          | Verifique logs para ajustes               |
| RC=08 ou RC=12                    | Erros graves                                 | Verifique parâmetros, permissões          |
| Tabela em `REORG PENDING`        | Falta de REORG após alterações               | Executar REORG e RUNSTATS                 |
| Tabela em `COPY PENDING`         | Falta de COPY após carga                     | Executar COPY imediatamente               |
| `CHECK PENDING` persistente      | Dados inválidos para restrições referenciais | Executar CHECK DATA e tratar exceções     |

---

### 📎 Referências Oficiais IBM

- [IBM - DB2 Utilities Overview](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-overview)
- [RUNSTATS Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-runstats-utility)
- [REORG Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-reorg-utility)
- [COPY Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-copy-utility)
- [CHECK Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-check-utility)
- [UNLOAD Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-unload-utility)

---

## 💻 Seção: JCL - Parte 5  
### Controle de Execução: Condições, Encadeamento e Orquestração de Steps

---

### 📘 Tópicos Abordados:
1. Agendamento e execução de JOBs
2. Encadeamento de steps com COND
3. Controle com IF/THEN/ELSE/ENDIF
4. Interpretação do código de retorno (RETCODE)
5. Melhores práticas de controle e orquestração
6. Exemplos práticos
7. Referência IBM

---

### 🔹 1. Agendamento e execução de JOBs

Os JOBs JCL podem ser:
- Submetidos manualmente (`SUB`)
- Agendados via scheduler (ex: **TWS**, **Control-M**, **ZEKE**)

Um JOB típico:
```jcl
//NOMEJOB JOB (ACCT),'MINHA ROTINA',CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
```

📌 Campos importantes:
- `CLASS` → define prioridade
- `MSGCLASS` → onde será enviada a saída
- `NOTIFY` → notifica o usuário após execução

---

### 🔹 2. Encadeamento com COND

O parâmetro `COND` permite **controlar a execução de steps com base em códigos de retorno anteriores**.

#### Sintaxe:
```jcl
//ETAPA2 EXEC PGM=PROG2,COND=(8,LT)
```

👆 Significa: "Se algum step anterior retornou RC **menor que 8**, **não execute** este step".

| Código | Significado                |
|--------|----------------------------|
| `COND=(4,LT)` | NÃO executa se RC anterior for < 4 |
| `COND=(0,EQ)` | NÃO executa se RC anterior for = 0 |

🎯 Lógica é **inversa**: se condição for **VERDADEIRA**, o step é **ignorado**.

---

### 🔹 3. Controle com IF/THEN/ELSE/ENDIF

Para maior controle, o JCL permite estrutura condicional com `IF`.

```jcl
//STEP1 EXEC PGM=PROG1
//STEP2 IF (STEP1.RC = 0) THEN
//         EXEC PGM=PROG2
//      ELSE
//         EXEC PGM=PROG3
//      ENDIF
```

📌 `STEP1.RC` → refere-se ao código de retorno da etapa anterior

🔍 Comparações possíveis:
- `=`, `¬=`, `<`, `>`, `<=`, `>=`

---

### 🔹 4. Interpretação do código de retorno (RETCODE)

Cada programa ou utilitário retorna um **RC (return code)**. O JCL avalia este valor para:
- Saber se o step terminou com sucesso
- Decidir se o próximo step será executado

| RC    | Significado                      |
|-------|----------------------------------|
| 0     | Sucesso                          |
| 4     | Sucesso com alertas              |
| 8     | Erros ou falhas                  |
| 12+   | Falhas críticas, exceções        |

---

### 🔹 5. Melhores práticas de controle

✅ **Padronize RC esperados** por step  
✅ Use `IF/THEN/ELSE` para lógica mais clara que `COND`  
✅ Utilize `STEPn.RC` para leitura direta de códigos  
✅ Evite cascatas desnecessárias de steps não controlados  
✅ Em jobs críticos, registre todos RCs em `SYSOUT` para auditoria

---

### 🔹 6. Exemplos práticos

#### ✔️ Exemplo com COND
```jcl
//STEP01 EXEC PGM=PROGRAMA1
//STEP02 EXEC PGM=PROGRAMA2,COND=(0,EQ)
```
📌 STEP02 **não será executado se STEP01 terminar com RC=0**.

---

#### ✔️ Exemplo com IF/THEN/ELSE
```jcl
//STEP01 EXEC PGM=VALIDA
//STEP02 IF (STEP01.RC = 0) THEN
//         EXEC PGM=CONTINUA
//      ELSE
//         EXEC PGM=ABORTA
//      ENDIF
```
📌 Roteia a execução com base no resultado da validação anterior.

---

#### ✔️ Exemplo com múltiplos níveis
```jcl
//STP1 EXEC PGM=INICIO
//STP2 IF (STP1.RC <= 4) THEN
//        EXEC PGM=MEIO
//        IF (STP2.RC = 0) THEN
//            EXEC PGM=FIM
//        ELSE
//            EXEC PGM=FALHA2
//        ENDIF
//     ELSE
//        EXEC PGM=FALHA1
//     ENDIF
```

---

### 📎 Referências Oficiais IBM

- [JCL - IF/THEN/ELSE/ENDIF](https://www.ibm.com/docs/en/zos/2.5.0?topic=statements-ifthenelseendif-statement)
- [JCL - COND Parameter](https://www.ibm.com/docs/en/zos/2.5.0?topic=statements-cond-parameter)
- [Return Codes in Batch Jobs](https://www.ibm.com/docs/en/zos/2.5.0?topic=statements-job-return-codes)

---

# 🗂️ JCL - Parte 6: Arquivos e Datasets no z/OS

O uso correto de **arquivos (datasets)** no JCL é essencial para garantir que os programas executem com sucesso, sem erros de alocação, acesso, lock ou falta de espaço. Esta seção explora **em profundidade** como utilizar datasets no JCL, desde o básico até exemplos mais completos.

---

## 📘 1. O que são Datasets?

Datasets são estruturas de arquivos do z/OS que armazenam informações de forma organizada. Podem ser:

- **Sequenciais** – lidos de forma linear
- **Particionados (PDS/PDS/E)** – com múltiplos membros
- **VSAM** – com acesso indexado (KSDS, ESDS, RRDS), muito usados por aplicações DB2/CICS

---

## 📂 2. Tipos de Datasets e Suas Aplicações

| Tipo      | Aplicação comum                                       | Exemplo de nome                    |
|-----------|--------------------------------------------------------|------------------------------------|
| Sequencial| Arquivos simples, logs, entrada/saída de programas     | `EMPRESA.RELATORIO.SEMANAL`        |
| PDS       | Armazenar JCLs, programas fonte, cópias COBOL          | `EMPRESA.JCL.LIB`, `EMPRESA.COPYLIB`|
| PDS/E     | Igual ao PDS, mas com melhorias de performance e estrutura | `EMPRESA.JCL.EPDS`             |
| VSAM      | Arquivos com acesso direto, usados por DB2/CICS        | `EMPRESA.CLIENTES.KSDS`            |

---

## 🧾 3. Componentes do DD Statement

Cada arquivo no JCL é definido por um `//DD` (Data Definition). Componentes comuns:

| Parâmetro     | Explicação |
|---------------|------------|
| `DSN=`        | Nome do dataset |
| `DISP=`       | Disposição do arquivo (status, ação normal, ação abend) |
| `SPACE=`      | Espaço a ser alocado (ex: TRACKs, CYLINDERS, bytes) |
| `UNIT=`       | Unidade de armazenamento (ex: SYSDA para discos) |
| `DCB=`        | Características físicas: RECFM, LRECL, BLKSIZE, DSORG |
| `VOL=SER=`    | Volume físico onde o dataset reside |
| `LABEL=`      | Informações de rotulagem, se for fita ou outros dispositivos |

---

## 🔄 4. DISP em Detalhe

```jcl
DISP=(status,normal-abend,abnormal-abend)
```

| Status | Significado                          |
|--------|--------------------------------------|
| NEW    | Cria um novo dataset                 |
| OLD    | Acessa com exclusividade             |
| SHR    | Acesso compartilhado                 |
| MOD    | Append (acrescenta ao final)         |

| Ação    | Efeito após step                    |
|---------|-------------------------------------|
| CATLG   | Catalogar no catálogo do sistema    |
| KEEP    | Manter sem catalogar                |
| DELETE  | Apagar                              |
| UNCATLG | Remover do catálogo                 |

---

## 📎 5. DCB – Data Control Block

Parâmetro que define o formato físico do arquivo:

```jcl
DCB=(RECFM=FB,LRECL=80,BLKSIZE=800,DSORG=PS)
```

| Parâmetro | Significado                                   |
|-----------|-----------------------------------------------|
| RECFM     | Formato do registro (FB=Fixed Blocked, VB=Variable Blocked) |
| LRECL     | Tamanho lógico de cada linha/registro         |
| BLKSIZE   | Tamanho de bloco (normalmente múltiplo de LRECL) |
| DSORG     | Organização do dataset (PS=Sequencial, PO=PDS) |

---

## 🔰 6. Exemplos Práticos

### ✅ Exemplo Simples – Leitura com DISP=SHR

```jcl
//LEITURA  EXEC PGM=PROG01
//INFILE   DD DSN=EMPRESA.ARQ.ENTRADA,DISP=SHR
```

> Permite que o programa leia o arquivo mesmo que outros estejam acessando.

---

### ✅ Exemplo Com Dataset de Saída

```jcl
//ESCREVER EXEC PGM=PROG02
//OUTFILE  DD DSN=EMPRESA.ARQ.SAIDA,
//            DISP=(NEW,CATLG,DELETE),
//            UNIT=SYSDA,
//            SPACE=(TRK,(10,5)),
//            DCB=(RECFM=FB,LRECL=100,BLKSIZE=1000)
```

> Cria novo dataset, aloca espaço e define formato. Se o job terminar normalmente, o arquivo será catalogado. Se abend, será deletado.

---

### ✅ Dataset Temporário Entre Steps

```jcl
//STEP1    EXEC PGM=PROG01
//TEMPDS   DD DSN=&&MEUARQ,DISP=(NEW,PASS),
//            SPACE=(CYL,(1,1)),UNIT=SYSDA

//STEP2    EXEC PGM=PROG02
//TEMPIN   DD DSN=&&MEUARQ,DISP=(OLD,DELETE)
```

> `&&` indica dataset temporário. `PASS` permite uso no próximo step. `DELETE` apaga após uso.

---

### ✅ Dataset Interno via SYSIN

```jcl
//STEP01   EXEC PGM=IDCAMS
//SYSIN    DD *
  DELETE EMPRESA.ARQ.TESTE
  SET MAXCC=0
/*
//SYSOUT   DD SYSOUT=*
```

> SYSIN contém comandos direto no JCL (entrada embutida). SYSOUT envia resultado para spool.

---

## 🧩 7. Organização de arquivos por convenção

Use nomes padronizados para facilitar rastreio, segurança e gerenciamento.

| Tipo de Uso       | Convenção Exemplo                    |
|-------------------|--------------------------------------|
| Entrada de dados  | `EMPRESA.ARQ.ENTRADA`                |
| Saída de dados    | `EMPRESA.ARQ.SAIDA`                  |
| Logs              | `EMPRESA.LOGS.PROGXX.DATA`           |
| Programas         | `EMPRESA.COBOL.FONTE`                |
| JCLs              | `EMPRESA.JCL.LIB`                    |
| Cópias (COPYBOOK) | `EMPRESA.COPYLIB`                    |

---

## 🔐 8. Segurança e RACF

O acesso a datasets pode ser controlado via **RACF (Resource Access Control Facility)**. Os administradores definem quem pode:

- Ler (READ)
- Escrever/alterar (UPDATE)
- Executar (EXECUTE)
- Excluir (DELETE)

> **Dica:** Se ocorrer erro `IEC150I 913-38`, provavelmente há **falta de permissão** no RACF.

---

## ✅ 9. Boas Práticas (Com Justificativas Detalhadas)

| Prática                                  | Justificativa Detalhada |
|------------------------------------------|---------------------------|
| Nomear arquivos de forma padronizada     | Facilita auditoria, rastreabilidade e manutenção de jobs, pois identifica claramente o propósito do arquivo no nome. |
| Usar `DISP=SHR` sempre que possível      | Evita conflitos de alocação e permite acesso concorrente em leitura, importante quando múltiplos jobs consultam o mesmo arquivo de referência. |
| Limitar espaço alocado com `SPACE`       | Garante uso eficiente de recursos de disco, evitando alocação exagerada e possíveis falhas por falta de espaço. É essencial em ambientes compartilhados. |
| Especificar `DCB` correto                | Alinha o formato físico do dataset ao que o programa espera, evitando erros de leitura/gravação como `S806`, `S013`, `S522`. |
| Preferir datasets temporários quando possível | Reduz a quantidade de arquivos no catálogo e minimiza a necessidade de limpeza manual ou criação de rotinas de housekeeping. Ideal para arquivos intermediários entre steps. |
| Validar permissões de acesso (RACF)      | Previne falhas em produção por falta de autorização. Ajuda na segurança dos dados, seguindo política de menor privilégio. |
| Deletar arquivos obsoletos com DISP=DELETE | Mantém o ambiente limpo, organizado e evita reutilização indevida de arquivos antigos com dados sensíveis. |
| Usar `LABEL=` corretamente               | Evita erros em arquivos de fita ou discos que exigem rotulagem, comuns em ambientes com suporte legado. |
| Revisar uso de `MOD` com cautela         | Append em arquivos pode causar duplicações se mal usado. Ideal para logs, mas perigoso para arquivos de controle ou produção. |

---

## 📚 Referências Oficiais IBM

- 🔗 [JCL Language Reference – IBM z/OS](https://www.ibm.com/docs/en/zos/3.1.0?topic=language-job-control-jcl)
- 🔗 [Managing Datasets – IBM z/OS](https://www.ibm.com/docs/en/zos/3.1.0?topic=datasets-using)
- 🔗 [DCB and Dataset Parameters](https://www.ibm.com/docs/en/zos/2.3.0?topic=jobs-specifying-data-control-blocks)

---

# 🗂️ JCL - Parte 7: COND, Códigos de Retorno (RC) e Controle de Execução Condicional

Controlar a execução condicional de steps com base em **códigos de retorno (Return Code, RC)** é uma das práticas mais poderosas e críticas no JCL. Permite a construção de fluxos inteligentes, evitando a execução de steps desnecessários, controlando dependências e prevenindo reprocessamentos indevidos.

---

## 🧠 1. O que é COND no JCL?

O parâmetro `COND=` define **condições para *pular* (não executar)** um step com base no RC de steps anteriores.

> 💡 **Importante:** `COND=` **NÃO controla a execução** de um step diretamente, mas **impede a execução caso a condição seja satisfeita**.

---

## ⚙️ 2. Sintaxe e Significado

```jcl
COND=(rc,operador)
```

| Elemento   | Significado |
|------------|-------------|
| `rc`       | Return Code a ser comparado (ex: 4, 8, 12) |
| `operador` | `LT`, `LE`, `EQ`, `NE`, `GE`, `GT` |

> 🛑 Se a condição for **verdadeira**, o step será **ignorado** (SKIPPED).

---

### ✔️ Exemplos Práticos

#### ✅ Exemplo 1 – Ignora se RC anterior for maior que 4:

```jcl
//STEP02 EXEC PGM=PROG2,COND=(4,LT)
```

> Se **algum step anterior** retornou RC **< 4**, a condição **não é satisfeita**, o step executa normalmente.  
> Se **RC ≥ 4**, então `4 LT RC` → **condição verdadeira**, **STEP02 é ignorado**.

---

#### ✅ Exemplo 2 – Ignorar se o step anterior deu RC=12

```jcl
//STEP02 EXEC PGM=PROG2,COND=(12,EQ)
```

> Se **RC = 12**, então `12 EQ 12` → condição satisfeita → step é ignorado.

---

#### ✅ Exemplo 3 – Ignorar se qualquer step anterior deu RC maior ou igual a 8:

```jcl
//STEP03 EXEC PGM=PROG3,COND=(8,LE)
```

> Se RC = 8, 12 ou 16 → `8 LE RC` é verdadeiro → step é ignorado.

---

## 🧾 3. Parâmetro COND no JOB vs no STEP

| Onde usar? | Efeito |
|------------|--------|
| `COND=` no JOB card | Avalia **todos os RCs** dos steps anteriores no job inteiro |
| `COND=` no STEP     | Avalia RCs de steps anteriores **apenas até aquele ponto** |

---

### 🧠 Exemplo: `COND` no JOB card

```jcl
//JOBX     JOB (ACCT),'TESTE',COND=(4,GT)
//STEP01   EXEC PGM=OK1
//STEP02   EXEC PGM=OK2
```

> Se qualquer step retornar RC > 4, o JCL ignora os steps seguintes.

---

## ✅ 4. Uso de RETURN CODE (RC) no COBOL e Outros Programas

Ao final da execução de um programa, é comum retornar um código de status:

```cobol
    MOVE 8 TO RETURN-CODE.
```

| RC  | Significado típico                      |
|-----|------------------------------------------|
| 0   | Sucesso                                  |
| 4   | Sucesso com advertência                  |
| 8   | Erro (dados inválidos, falha de lógica)  |
| 12  | Erro grave (sintaxe, arquivo não lido)   |
| 16  | Erro de sistema ou falta de recurso      |

> 💡 Você pode criar sua lógica de RC no programa para controlar o fluxo no JCL.

---

## 🧪 5. Exemplos Completos de Controle Condicional

### 🧷 Cenário A – Executar somente se o step anterior foi OK (RC=0)

```jcl
//STEP1   EXEC PGM=PROG1
//STEP2   EXEC PGM=PROG2,COND=(0,NE)
```

> Se STEP1 retornar RC = 0 → `0 NE 0` é falso → STEP2 é executado  
> Se STEP1 retornar RC ≠ 0 → condição verdadeira → STEP2 é ignorado

---

### 🧷 Cenário B – Fazer rollback apenas se houver erro no processamento

```jcl
//PROCESS EXEC PGM=COBPROC
//ROLLBCK EXEC PGM=UNDO,COND=(0,EQ)
```

> `0 EQ 0` só é verdadeiro se o RC anterior for 0 → nesse caso o rollback é ignorado.  
> Se houve erro (RC ≠ 0), rollback será executado.

---

### 🧷 Cenário C – Executar utilitário de análise só se RC for 8 ou mais

```jcl
//ANALISA EXEC PGM=PROG3,COND=(8,GT)
```

> RC = 0, 4 → `8 GT RC` = verdadeiro → step **ignorado**  
> RC = 8, 12 → `8 GT RC` = falso → step **executado**

---

## 🚧 6. Cuidado com múltiplos steps: Avaliação de COND

### ❗ Erro comum:

```jcl
//STEP1 EXEC PGM=OK1
//STEP2 EXEC PGM=OK2,COND=(4,LT)
```

Se STEP1 retornar RC = 8: `4 LT 8` é verdadeiro → STEP2 é ignorado  
Porém se STEP1 retornar RC = 0: `4 LT 0` = falso → STEP2 é executado

> 🔍 Sempre pense como: **"se a condição for verdadeira, o step é ignorado"**.

---

## 💎 7. Boas Práticas com COND e RC

| Prática                                 | Justificativa |
|-----------------------------------------|----------------|
| Use `COND=(0,NE)` para steps dependentes | Garante que o step só rode se o anterior tiver sucesso total (RC=0) |
| Evite `COND=EVEN` e `COND=ONLY`, exceto se realmente necessário | Essas formas legadas são menos claras e dificultam a leitura/manutenção |
| Use nomes significativos nos steps      | Ajuda a rastrear qual step produziu qual RC e sua influência na execução |
| Avalie RCs no programa COBOL            | Permite controle preciso de comportamento no JCL (ex: diferenciar RC=4 e RC=8) |
| Teste em ambiente seguro com RCs simulados | Ajuda a validar lógicas de COND antes da produção |
| Sempre documente a lógica de COND no JCL | Melhora compreensão futura, principalmente se o RC não for 0 |

---

## 🧰 8. Alternativas a COND: IF/THEN/ELSE no JCL

O JCL moderno permite controle condicional usando blocos estruturados:

```jcl
//STEP1 EXEC PGM=PROG1
// IF (STEP1.RC = 0) THEN
//STEP2 EXEC PGM=PROG2
// ELSE
//STEP3 EXEC PGM=ERROLOG
// ENDIF
```

> Mais legível e estruturado. Ideal para fluxos complexos.

---

## 🧠 Conclusão

Dominar o uso de `COND` e dos códigos de retorno no JCL é essencial para o controle eficaz de fluxos batch. Ele permite:

- Evitar execução desnecessária
- Garantir consistência de dados
- Controlar dependências de steps
- Reduzir riscos de falhas em produção

---

## 📚 Referências

- 🔗 [z/OS JCL Reference – COND](https://www.ibm.com/docs/en/zos/3.1.0?topic=statements-exec-cond-parameter)
- 🔗 [Structured Conditional Processing (IF/THEN/ELSE)](https://www.ibm.com/docs/en/zos/3.1.0?topic=statements-ifthenelseendif-construct)

---

# 🧩 JCL - Parte 8: IF/THEN/ELSE/ENDIF – Controle Condicional Estruturado no JCL com foco em DB2

O uso de `IF/THEN/ELSE/ENDIF` no JCL traz clareza e controle estrutural avançado à execução condicional de steps. Essa abordagem moderna é preferida em ambientes corporativos, especialmente quando lidamos com **programas COBOL com DB2** ou **utilitários de manutenção de objetos do banco de dados**.

---

## 🧠 1. O que é IF/THEN/ELSE/ENDIF no JCL?

É uma construção **estruturada de decisão** que permite controlar a execução de steps com base em códigos de retorno anteriores.

> 💡 É mais legível e flexível que `COND=`, além de facilitar a manutenção de fluxos complexos.

---

## 🔤 2. Sintaxe Básica

```jcl
//STEP1 EXEC PGM=PGM1
// IF (STEP1.RC = 0) THEN
//STEP2 EXEC PGM=PGM2
// ELSE
//STEP3 EXEC PGM=PGM3
// ENDIF
```

---

## 🧾 3. Condições Suportadas

| Comparador | Significado |
|------------|-------------|
| `=`        | Igual       |
| `¬=` ou `NE` | Diferente  |
| `>`        | Maior       |
| `<`        | Menor       |
| `>=`       | Maior ou igual |
| `<=`       | Menor ou igual |

Também é possível combinar múltiplas condições com `AND`, `OR`.

---

## 📌 4. Aplicações em DB2 – Exemplos Comuns

### ✅ Exemplo 1 – Executar DSNTIAUL somente se programa COBOL+DB2 tiver sucesso

```jcl
//STEP1 EXEC PGM=IKJEFT01
//SYSTSIN DD *
DSN SYSTEM(DB01)
RUN PROGRAM(MYPROG) PLAN(MYPLAN) LIB('PROG.LOADLIB')
END
/*
// IF (STEP1.RC = 0) THEN
//STEP2 EXEC PGM=IKJEFT01
//SYSTSIN DD *
DSN SYSTEM(DB01)
RUN PROGRAM(DSNTIAUL) PLAN(DSNTIAUL) LIB('SYS3.DB2.LOADLIB')
END
/*
//SYSTSPRT DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
// ELSE
//ERROMSG EXEC PGM=IEBGENER
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD *
==> FALHA NA EXECUÇÃO DO PROGRAMA DB2
/*
//SYSUT2   DD SYSOUT=*
//SYSIN    DD DUMMY
// ENDIF
```

> 📌 Se o programa principal (MYPROG) rodar com RC=0, executa DSNTIAUL para extração. Caso contrário, exibe mensagem de erro.

---

### ✅ Exemplo 2 – Reorganizar apenas se análise indicou necessidade

```jcl
//ANALISA EXEC PGM=DSNUTILB,REGION=0M,
//         PARM='DB01,ANALISE'
//SYSIN DD *
  REPORT TABLESPACE DBNAME.TSNAME
/*
// IF (ANALISA.RC = 4 OR ANALISA.RC = 8) THEN
//REORG EXEC PGM=DSNUTILB,REGION=0M,
//       PARM='DB01,REORGTABLESPACE'
//SYSIN DD *
  REORG TABLESPACE DBNAME.TSNAME LOG NO
/*
// ENDIF
```

> ✅ RC 4 ou 8 pode indicar necessidade de reorganização. O IF garante que o REORG só será executado nesses casos.

---

### ✅ Exemplo 3 – Validar se programa retornou RC=12 e fazer tratamento

```jcl
//RUNPGM EXEC PGM=IKJEFT01
//SYSTSIN DD *
DSN SYSTEM(DB01)
RUN PROGRAM(MYPROG) PLAN(MYPLAN)
END
/*
// IF (RUNPGM.RC = 12) THEN
//NOTIFY EXEC PGM=IEBGENER
//SYSUT1 DD *
ATENÇÃO: O PROGRAMA DB2 RETORNOU RC=12. VERIFIQUE O LOG.
/*
//SYSUT2 DD SYSOUT=*
//SYSIN  DD DUMMY
//SYSPRINT DD SYSOUT=*
// ENDIF
```

> ✅ Permite criar mensagens de alerta quando o programa retorna RC específico (ex: erro de lógica ou falha de leitura).

---

## 🧰 5. Combinando Condições: IF com múltiplos testes

```jcl
// IF ((STEP1.RC > 0) AND (STEP2.RC = 0)) THEN
//...
// ENDIF
```

> 💡 Usa-se parênteses para agrupar e garantir avaliação correta de múltiplas condições.

---

## 🧵 6. Uso em Manutenção de Tabelas DB2

### ✅ Exemplo – Realizar RUNSTATS após LOAD se RC for 0 ou 4

```jcl
//LOAD EXEC PGM=DSNUTILB,PARM='DB01,LOAD'
//SYSIN DD *
  LOAD DATA INDDN(SYSREC) INTO TABLE MYDB.MYTBL
/*
// IF (LOAD.RC = 0 OR LOAD.RC = 4) THEN
//STATS EXEC PGM=DSNUTILB,PARM='DB01,RUNSTATS'
//SYSIN DD *
  RUNSTATS TABLESPACE MYDB.TS01 TABLE(MYDB.MYTBL)
/*
// ENDIF
```

> 💡 Garantimos que o RUNSTATS será executado apenas após LOAD bem-sucedido ou com advertência tolerável.

---

## 🛠️ 7. Boas Práticas

| Prática | Justificativa |
|--------|----------------|
| Prefira `IF/THEN/ELSE` para fluxos legíveis | Evita armadilhas do `COND=` e facilita leitura por outros profissionais |
| Nomeie os steps com clareza | Ex: `RUNLOAD`, `RUNREORG`, `VALIDA`, `NOTIFYERR` |
| Documente os motivos dos blocos `IF` | Comentários no JCL ajudam na manutenção |
| Evite encadeamentos complexos de IF | Prefira dividir em steps separados se necessário |
| Valide em ambiente de testes com RCs simulados | Para garantir que fluxos com `IF` se comportem como esperado |

---

## 🧠 Conclusão

O uso de `IF/THEN/ELSE/ENDIF` no JCL traz controle de execução estruturado e robusto, ideal para rotinas críticas com DB2, como:

- Execução condicional de programas COBOL+DB2
- Manutenção de objetos DB2 (REORG, STATS, LOAD)
- Controle de fluxo com base em RCs conhecidos

Dominar essa estrutura permite construir JCLs mais inteligentes, seguros e fáceis de manter.

---

## 📚 Referências

- 🔗 [IBM JCL User Guide - IF/THEN/ELSE/ENDIF](https://www.ibm.com/docs/en/zos/3.1.0?topic=statements-ifthenelseendif-construct)
- 🔗 [IBM DB2 Utilities Guide](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-db2-utility-guide-reference)

---

# 🧩 JCL - Parte 9: Parâmetros de Execução no JCL (EXEC & JOB) – Uso em DB2

Os parâmetros de execução permitem controlar **como, quando e onde** um job será processado no ambiente z/OS. Entender cada um deles é essencial para garantir a correta **execução de programas COBOL/DB2** e **utilitários como RUNSTATS, REORG, DSNTIAUL, etc**.

Nesta seção, vamos detalhar os principais parâmetros usados nas instruções `JOB` e `EXEC`, sempre com exemplos aplicados ao contexto do DB2.

---

## 🧷 1. Parâmetros na Instrução `JOB`

### 🎯 Objetivo: Controlar o comportamento geral do job, como identificação, saída, limites de tempo e recursos.

### ✔️ Sintaxe:

```jcl
//MEUJOB JOB (1234),'JOSE DBA',
//  CLASS=A,MSGCLASS=X,MSGLEVEL=(1,1),
//  TIME=1440,REGION=0M,NOTIFY=&SYSUID
```

---

### 📌 Parâmetros comuns:

| Parâmetro | Função | Detalhe |
|----------|--------|---------|
| `CLASS=` | Prioridade de execução no sistema | A = produção, T = teste, B = batch leve (pode variar por site) |
| `MSGCLASS=` | Destino da saída do job (JES output) | Ex: X (arquivo em spool), A (impressora) |
| `MSGLEVEL=` | Controle do que será listado no JESYSMSG | `(1,1)` exibe todas mensagens, incl. datasets alocados |
| `NOTIFY=` | Notifica o TSO do usuário após término do job | Com `&SYSUID`, notifica quem submeteu |
| `TIME=` | Limita o tempo máximo de execução do job | `TIME=1440` = máximo permitido (24h) |
| `REGION=` | Quantidade de memória (RAM) alocada | `REGION=0M` = máximo permitido |

> 💡 Recomenda-se usar `TIME=1440` e `REGION=0M` para evitar interrupções em jobs DB2 pesados (REORG, LOAD, etc.)

---

### ✅ Exemplo real:

```jcl
//DB2UTL JOB (ACCT),'RUNSTATS PROD',
//  CLASS=A,MSGCLASS=X,MSGLEVEL=(1,1),
//  TIME=1440,REGION=0M,NOTIFY=&SYSUID
```

> 🔎 Executa um job de RUNSTATS em ambiente de produção com prioridade e uso total de recursos.

---

## 🧷 2. Parâmetros na Instrução `EXEC`

### 🎯 Objetivo: Controlar a execução de cada step individual, especialmente programas DB2 ou utilitários.

### ✔️ Sintaxe:

```jcl
//STEP1 EXEC PGM=IKJEFT01,REGION=0M,TIME=1440
```

---

### 📌 Parâmetros úteis:

| Parâmetro | Função | Detalhe |
|----------|--------|---------|
| `PGM=` | Nome do programa a ser executado | Ex: `IKJEFT01` (TSO DB2), `DSNUTILB`, `DSNTIAUL` |
| `PARM=` | Parâmetros passados ao programa | Ex: `PARM='DB01,REORG'` |
| `TIME=` | Tempo máximo do step | Recomendado: `1440` ou omitido |
| `REGION=` | Memória dedicada ao step | `REGION=0M` para máximo |
| `COND=` | Evita execução se condição anterior for satisfeita | Substituído por `IF/THEN` no padrão moderno |
| `TYPRUN=` | Tipo de execução | `HOLD`, `SCAN` ou `COPY` (veja abaixo) |

---

## 🧮 3. TYPRUN= – Executando com opções especiais

| Valor | Descrição | Uso Prático |
|-------|-----------|-------------|
| `SCAN` | Verifica erros de sintaxe, não executa | Útil em validação antes de enviar job crítico |
| `HOLD` | Mantém job aguardando liberação no spool | Ideal para revisão manual pré-execução |
| `COPY` | Faz cópia do job para análise | Pouco comum |

```jcl
//STEP1 EXEC PGM=DSNUTILB,TYPRUN=SCAN
```

> 🔍 Simula execução do utilitário para detectar problemas no JCL sem rodar de fato.

---

## 🧪 4. Exemplo completo de JOB com EXEC para programa DB2

```jcl
//MEUJOB JOB (9999),'COBOL DB2 RUN',
//  CLASS=T,MSGCLASS=X,MSGLEVEL=(1,1),
//  TIME=1440,REGION=0M,NOTIFY=&SYSUID
//*
//STEP1 EXEC PGM=IKJEFT01,REGION=0M,TIME=1440
//STEPLIB  DD DSN=DB2.PROD.LOADLIB,DISP=SHR
//SYSTSIN  DD *
DSN SYSTEM(DBP1)
RUN PROGRAM(MYPROG) PLAN(MYPLAN) -
  LIB('COBOL.LOADLIB')
END
/*
//SYSPRINT DD SYSOUT=*
//SYSTSPRT DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
```

> ✅ Programa `MYPROG` compilado em COBOL com acesso a DB2 (plano `MYPLAN`), usando IKJEFT01.

---

## 🧩 5. Recomendações práticas para DB2

| Situação | Parâmetros recomendados |
|----------|-------------------------|
| Jobs com REORG ou LOAD | `REGION=0M`, `TIME=1440`, `CLASS=A` |
| Jobs de teste COBOL+DB2 | `CLASS=T`, `MSGCLASS=X`, `TIME=1440` |
| Jobs em produção noturna | `TYPRUN=HOLD`, revisão antes da liberação |
| Análise antes da execução | `TYPRUN=SCAN`, valida sintaxe sem rodar |

---

## 🛠️ 6. Boas práticas com parâmetros de execução

- Use sempre `REGION=0M` para jobs DB2 – evita abends por falta de memória.
- Evite `TIME=0` (tempo ilimitado) – use `TIME=1440` para controle seguro.
- Centralize parâmetros comuns no cabeçalho `JOB` para evitar repetição nos `EXEC`.
- Utilize `MSGLEVEL=(1,1)` para rastrear datasets e execuções no JESYSMSG.
- Evite `COND=` se possível – prefira `IF/THEN/ELSE`.

---

## 📚 Referências

- 🔗 [IBM z/OS JCL User’s Guide](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-job-control-language-reference)
- 🔗 [IBM DB2 for z/OS Utility Guide](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-db2-utility-guide-reference)
- 🔗 [JCL Parameters Quick Reference (IBM Redbooks)](https://www.redbooks.ibm.com)

---

# 🧩 JCL - Parte 10: Controle de Dados – DD Statements aplicados ao DB2

As instruções `DD` (Data Definition) são fundamentais para o funcionamento dos steps JCL, pois definem **onde o programa encontra dados de entrada, onde grava saídas, quais bibliotecas usar, arquivos temporários, tabelas DB2, e muito mais**.

No contexto do DB2, o uso correto de DD Statements garante a execução adequada de **programas COBOL com SQL embutido**, **utilitários como RUNSTATS, LOAD, REORG, DSNTIAUL, DSNUTILB**, além de permitir diagnósticos eficientes com `SYSPRINT`, `SYSOUT`, `SYSUDUMP`, entre outros.

---

## 🎯 1. Estrutura básica de uma DD Statement

```jcl
//NOMEDD   DD  PARÂMETROS
```

Exemplos comuns de parâmetros:
- `DSN=`: nome do dataset
- `DISP=`: status de alocação (NEW, SHR, OLD, MOD)
- `SPACE=`: espaço necessário (TRACKS, CYLINDERS)
- `UNIT=`: tipo de dispositivo (SYSDA, 3390)
- `SYSOUT=*`: direciona a saída para spool (impresso)

---

## 📂 2. DD Statements mais usados em jobs DB2

Abaixo, detalhamos os principais DDs usados em programas COBOL/DB2 e utilitários DB2.

---

### 🔹 `STEPLIB`

Aponta para a biblioteca de carga que contém os módulos executáveis do DB2.

```jcl
//STEPLIB DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
```

> ⚠️ Essencial para execução de `IKJEFT01`, `DSNUTILB` e outros programas relacionados ao DB2.

---

### 🔹 `SYSTSIN`

Entrada de comandos TSO para steps com `PGM=IKJEFT01`. Nele definimos comandos como `DSN`, `RUN`, `END`, usados para rodar programas COBOL com SQL embutido.

```jcl
//SYSTSIN DD *
  DSN SYSTEM(DBP1)
  RUN PROGRAM(MYPROG) PLAN(MYPLAN) -
    LIB('USR.LOADLIB')
  END
```

---

### 🔹 `SYSTSPRT`

Saída da execução do TSO (resultado de comandos, mensagens do DB2, status de retorno).

```jcl
//SYSTSPRT DD SYSOUT=*
```

---

### 🔹 `SYSIN`

Entrada de dados ou parâmetros para utilitários DB2 (LOAD, REORG, RUNSTATS, etc).

```jcl
//SYSIN DD *
  LOAD DATA INDDN(SYSREC) LOG NO REPLACE
    INTO TABLE DBCLIENTE.CLIENTE
    (ID POSITION(1:5),
     NOME POSITION(6:35),
     UF POSITION(36:37))
```

---

### 🔹 `SYSPRINT`

Gera relatórios e mensagens da execução dos utilitários.

```jcl
//SYSPRINT DD SYSOUT=*
```

---

### 🔹 `SYSREC`

Dataset com os dados a serem carregados (utilizado no LOAD).

```jcl
//SYSREC DD DSN=USR.CLIENTES.INPUT,DISP=SHR
```

---

### 🔹 `SYSUT1`, `SYSUT2`, `SORTIN`, `SORTOUT`

Arquivos temporários ou intermediários, comumente usados por SORT, REORG, UNLOAD.

```jcl
//SYSUT1   DD UNIT=SYSDA,SPACE=(CYL,(10,5))
//SORTOUT  DD DSN=TEMP.OUTPUT.SORTED,DISP=(NEW,CATLG,DELETE),
//            UNIT=SYSDA,SPACE=(CYL,(10,5))
```

---

### 🔹 `SYSERR`, `SYSOUT`, `SYSUDUMP`, `SYSABEND`

Mensagens de erro, dumps de abend, logs de execução.

```jcl
//SYSERR   DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
//SYSABEND DD SYSOUT=*
```

---

### 🔹 Outros DDs específicos

| DDNAME     | Utilidade                                         |
|------------|---------------------------------------------------|
| `DSNTRACE` | Geração de rastreamento do DB2 para diagnóstico   |
| `SYSDUMP`  | Geração de dumps para abends                      |
| `SYSABOUT` | Saída alternativa de utilitários                  |
| `DSNTRACE` | Diagnóstico profundo de execuções com erro        |

---

## 🧪 3. Exemplo completo com RUNSTATS

```jcl
//RUNSTATS JOB (ACCT),'RUNSTATS EXEMPLO',
//  CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//*
//STEP1 EXEC PGM=DSNUTILB,PARM='DBP1'
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
//SYSIN    DD *
  RUNSTATS TABLESPACE DBCLIENTE.TSCLIENTE
    TABLE(ALL) INDEX(ALL)
/*
```

---

## 🧪 4. Exemplo completo com programa COBOL + DB2 (IKJEFT01)

```jcl
//EXECDB2 JOB (ACCT),'EXECUTAR PROG DB2',
//  CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//*
//STEP01 EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD DSN=USR.LOADLIB,DISP=SHR
//         DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(DBP1)
  RUN PROGRAM(MYPROG) PLAN(MYPLAN) -
    LIB('USR.LOADLIB')
  END
/*
//SYSTSPRT DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
```

---

## 📌 5. Dicas práticas para uso de DDs em ambientes DB2

| Boas práticas | Motivo |
|---------------|--------|
| Nomear DDs conforme convenções do DB2 | Facilita entendimento e suporte |
| Direcionar saídas para `SYSOUT=*` sempre que possível | Visualização rápida no spool |
| Evitar espaço em excesso com `SPACE=(CYL,(10,5),RLSE)` | Liberação automática de espaço |
| Testar jobs com `TYPRUN=SCAN` antes da execução real | Verificação de sintaxe sem execução |
| Utilizar `DISP=(NEW,CATLG,DELETE)` para arquivos temporários | Boa gestão de datasets no catálogo |

---

## 📚 Referências

- [IBM z/OS JCL User Guide](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-job-control-language-reference)
- [IBM DB2 for z/OS Utility Guide](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-db2-utility-guide-reference)
- [DFSORT Application Programming Guide](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-dfsort-application-programming)

---

# 🧩 JCL - Parte 11: Uso de Variáveis e Parâmetros Simbólicos no JCL

## 🧠 O que são parâmetros simbólicos?

Parâmetros simbólicos são **variáveis substituíveis** no JCL, definidos e resolvidos no momento da submissão do job. Eles proporcionam **flexibilidade, reutilização e padronização** de jobs, permitindo a passagem de valores como datas, nomes de datasets, identificadores de ambiente e muito mais.

---

## 🎯 1. Tipos de variáveis mais comuns

### 🔹 Variáveis automáticas (simbólicos do sistema)
São fornecidas pelo z/OS automaticamente:

| Variável       | Significado                                  |
|----------------|----------------------------------------------|
| `&SYSUID`      | ID do usuário que submeteu o job             |
| `&SYSDATE`     | Data da submissão (formato YYMMDD)           |
| `&SYSTIME`     | Hora da submissão (formato HHMMSS)           |
| `&SYSNAME`     | Nome do sistema                              |
| `&JOBNAME`     | Nome do job                                  |

Exemplo:

```jcl
//MYJOB JOB (ACCT),'&SYSUID',CLASS=A,MSGCLASS=X
```

---

### 🔹 Parâmetros simbólicos definidos pelo usuário

Você pode definir seus próprios parâmetros em **procedimentos (PROCs)** ou diretamente em **parâmetros de steps**, assim:

```jcl
//MYPROC PROC DB2SYS=DBP1,PROG=MYPROG
//STEP01 EXEC PGM=IKJEFT01
//SYSTSIN DD *
  DSN SYSTEM(&DB2SYS)
  RUN PROGRAM(&PROG) PLAN(&PROG) -
    LIB('USR.LOADLIB')
  END
/*
//MYPROC PEND
```

Chamada do PROC:

```jcl
//CALLDB2 EXEC MYPROC,DB2SYS=DSN1,PROG=PROGTEST
```

---

## 🛠️ 2. Exemplos práticos no contexto DB2

### 🔸 Exemplo 1: Job parametrizado para diferentes programas

```jcl
//RUNDB2  JOB (ACCT),'DB2 PROG PARAM',
//  CLASS=A,MSGCLASS=X
//*
//STEP01 EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//         DD DSN=USR.LOADLIB,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(&DB2SYS)
  RUN PROGRAM(&PROG) PLAN(&PLAN) -
    LIB('&LIB')
  END
/*
//SYSTSPRT DD SYSOUT=*
```

Chamada com substituições:

```jcl
//  SET DB2SYS=DBP1
//  SET PROG=FATURADB
//  SET PLAN=FATURA13
//  SET LIB=USR.LOADLIB
```

---

### 🔸 Exemplo 2: Gerar datasets com data e usuário no nome

```jcl
//STEP01  EXEC PGM=IEBGENER
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=INPUT.DATA,DISP=SHR
//SYSUT2   DD DSN=BACKUP.&SYSUID..&SYSDATE,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(1,1)),UNIT=SYSDA
//SYSIN    DD DUMMY
```

Resultado:  
`DSN=BACKUP.JOSE.250522` (exemplo)

---

## 📌 3. Boas práticas com simbólicos (continuação)

| Boas práticas                                 | Justificativa                                |
|-----------------------------------------------|-----------------------------------------------|
| Usar simbólicos para sistemas DB2             | Facilita execução em múltiplos ambientes      |
| Centralizar parâmetros em PROC ou SET         | Melhora manutenção e clareza                  |
| Combinar com IF/THEN/ELSE (condicional JCL)   | Permite lógica mais inteligente no job flow   |
| Evitar nomes genéricos como &A, &B             | Torna o código difícil de entender e manter   |
| Comentar o uso das variáveis                  | Ajuda na manutenção por outros profissionais  |
| Evitar substituições dinâmicas em produção    | Minimiza risco de erro e facilita auditoria   |

---

## 🔄 4. Como os simbólicos são resolvidos?

A substituição das variáveis simbólicas ocorre na **expansão do job** no momento da submissão. A prioridade de resolução é:

1. Parâmetros informados no `EXEC` do PROC
2. Instruções `SET` no início do job
3. Valores default no `PROC`

Exemplo completo:

```jcl
//SETENV  SET DB2SYS=DSN1,PROG=CLIENTES,PLAN=PLCLIENT,LIB=MY.LOAD
//*
//EXEC01  EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//         DD DSN=&LIB,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(&DB2SYS)
  RUN PROGRAM(&PROG) PLAN(&PLAN) -
    LIB('&LIB')
  END
/*
//SYSTSPRT DD SYSOUT=*
```

Esse modelo permite alterar facilmente o ambiente DB2 (`DSN1`, `DSN2`, etc.), o programa e seu plano, apenas ajustando os valores no `SET`.

---

## 🧩 5. Integração simbólica em PROCs

### 🧷 Exemplo de PROC reutilizável

```jcl
//DB2PROC PROC DB2SYS=DSN1,PROG=GENERIC,PLAN=GENPLAN,LIB=USR.LOAD
//RUNSTEP EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//         DD DSN=&LIB,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(&DB2SYS)
  RUN PROGRAM(&PROG) PLAN(&PLAN) -
    LIB('&LIB')
  END
/*
//SYSTSPRT DD SYSOUT=*
//DB2PROC PEND
```

### 🧷 Chamada personalizada do PROC

```jcl
//EXECDB2 EXEC DB2PROC,DB2SYS=DSN2,PROG=FATURAMENTO,PLAN=FATURA13
```

Este modelo torna possível padronizar todos os jobs de programas DB2 na empresa e só alterar os parâmetros conforme necessário. Ajuda no controle de versões, auditoria e mudança de ambientes de forma segura e prática.

---

## 🔗 Referência oficial IBM

Para aprofundamento no uso de simbólicos em JCL:

🔹 [IBM z/OS MVS JCL Reference – Symbolic Parameters](https://www.ibm.com/docs/en/zos/3.1.0?topic=jobs-symbolic-parameters)

---

## ✅ Conclusão da Parte 11

O uso adequado de variáveis e parâmetros simbólicos no JCL é **essencial para padronizar, simplificar e escalar a execução de jobs**, especialmente em ambientes complexos como os que envolvem **DB2 for z/OS**. Saber usar parâmetros de forma consciente melhora a **manutenibilidade**, reduz erros humanos e favorece a automação.

---

# 🧩 JCL - Parte 12: Uso de Condicionais no JCL (IF / THEN / ELSE / ENDIF)

## 📘 Objetivo

Controlar o fluxo de execução de steps com base no **código de retorno (RC)** de steps anteriores, utilizando comandos condicionais no JCL.

Muito útil para:
- Executar um step apenas se o anterior foi bem-sucedido (ex: RC = 0)
- Pular utilitários se uma carga falhou
- Redirecionar execução para steps de correção ou análise
- Evitar reruns desnecessários

---

## 🧩 Estrutura básica

```jcl
//STEP01   EXEC PGM=PGMA
//IF (STEP01.RC = 0) THEN
//STEP02   EXEC PGM=PGMB
//ELSE
//ERROSTEP EXEC PGM=IEBGENER
//ENDIF
```

**Notas:**
- `STEP01.RC` refere-se ao Return Code do step anterior.
- As comparações podem ser: `=`, `>`, `<`, `>=`, `<=`, `¬=`
- Você pode usar nomes simbólicos de step (como `PGMCHECK.RC`)

---

## 🧪 Exemplo prático com programa DB2

```jcl
//PGMCHECK EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(DSN1)
  RUN PROGRAM(VALIDA) PLAN(PLANVAL) -
      LIB('USR.LOAD')
  END
/*
//SYSTSPRT DD SYSOUT=*

//IF (PGMCHECK.RC = 0) THEN
//DB2LOAD  EXEC PGM=IKJEFT01
//STEPLIB  DD DSN=DB2.V13.RUNLIB.LOAD,DISP=SHR
//SYSTSIN  DD *
  DSN SYSTEM(DSN1)
  RUN PROGRAM(CARGA) PLAN(PLANCARGA) -
      LIB('USR.LOAD')
  END
/*
//SYSTSPRT DD SYSOUT=*

//ELSE
//ERROMSG EXEC PGM=IKJEFT01
//SYSTSIN  DD *
  ECHO PROBLEMA NA VALIDAÇÃO! NÃO EXECUTAR CARGA.
/*
//SYSTSPRT DD SYSOUT=*
//ENDIF
```

---

## 🔄 Comparações válidas

| Sintaxe           | Significado                  |
|------------------|------------------------------|
| `=`, `EQ`         | Igual                        |
| `¬=`, `NE`, `^=`  | Diferente                    |
| `<`, `LT`         | Menor                        |
| `>`, `GT`         | Maior                        |
| `<=`, `LE`        | Menor ou igual               |
| `>=`, `GE`        | Maior ou igual               |

---

## ⚠️ Boas práticas

- Sempre use nomes de steps significativos (`VALIDA`, `CARGA`, `ERROMSG` etc.)
- Documente as decisões do fluxo condicional
- Evite lógicas confusas com muitos níveis aninhados
- Utilize condicionais para evitar reruns de jobs inteiros
- Sempre teste as condições em ambiente controlado antes da produção

---

## 🔗 Referência oficial IBM

🔹 [IBM z/OS MVS JCL Reference – IF/THEN/ELSE/ENDIF Statements](https://www.ibm.com/docs/en/zos/3.1.0?topic=statements-if-then-else-endif)

---

## ✅ Conclusão da Parte 12

O uso de `IF/THEN/ELSE` no JCL é essencial para controlar logicamente a execução de jobs, reagir a falhas, automatizar rotinas corretivas e garantir a execução somente quando as condições certas forem atendidas. É amplamente utilizado em ambientes DB2, especialmente para controlar sequência de programas e utilitários com base nos RCs.

---
