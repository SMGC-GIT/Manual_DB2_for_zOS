# Utilitário DEFUTIL - DB2 for z/OS

---

## Índice

- [1. O que é o utilitário DEFUTIL](#1-o-que-é-o-utilitário-defutil)
- [2. Quando usar o DEFUTIL](#2-quando-usar-o-defutil)
- [3. Como funciona o DEFUTIL](#3-como-funciona-o-defutil)
- [4. Sintaxe e parâmetros](#4-sintaxe-e-parâmetros)
- [5. Exemplo de JCL](#5-exemplo-de-jcl)
- [6. Exemplos práticos de uso](#6-exemplos-práticos-de-uso)
- [7. Cuidados e riscos ao utilizar](#7-cuidados-e-riscos-ao-utilizar)
- [8. Consultas na SYSUTIL para diagnóstico](#8-consultas-na-sysutil-para-diagnóstico)
- [9. Referências oficiais IBM](#9-referências-oficiais-ibm)

---

## 1. O que é o utilitário DEFUTIL

O `DEFUTIL` (Define Utility) é um utilitário administrativo interno do DB2 for z/OS que permite gerenciar manualmente as entradas da tabela `SYSIBM.SYSUTIL`.

Essa tabela controla as execuções ativas dos utilitários DB2, como `REORG`, `COPY`, `LOAD`, entre outros. O DEFUTIL é útil para atualizar ou remover registros que podem bloquear futuras execuções desses utilitários.

---

## 2. Quando usar o DEFUTIL

O uso do DEFUTIL é indicado em situações como:

- A execução de um utilitário falhou e deixou entradas "presas" na `SYSUTIL`, bloqueando reexecuções;
- A necessidade de forçar o status de uma `UTILID` para "STOPPED", liberando o recurso;
- Situações de teste, onde se deseja simular uma execução ativa (status "STARTED");
- Correção administrativa de inconsistências no catálogo após falhas no job.

---

## 3. Como funciona o DEFUTIL

O DEFUTIL atualiza diretamente a tabela `SYSIBM.SYSUTIL`, manipulando os metadados que representam a execução ativa de utilitários.

### Estrutura da SYSUTIL

| Coluna           | Descrição                                     |
|------------------|-----------------------------------------------|
| UTILID           | Nome lógico da execução do utilitário         |
| UTDBID           | ID do database envolvido                      |
| UTPSID           | ID do tablespace (Page Set ID)                |
| UTPROC           | Tipo de utilitário (REORG, LOAD, COPY, etc.)  |
| UTSTATUS         | Status da execução: STARTED ou STOPPED        |

---

## 4. Sintaxe e parâmetros

```plaintext
DEFUTIL
  UTILID(nome-da-utilid)
  DBID(id-do-database)
  PSID(id-do-page-set)
  UTPROC(tipo-de-utilitario)
  STATUS(STARTED | STOPPED)
```

### Explicação dos parâmetros

| Parâmetro | Requerido | Descrição |
|-----------|-----------|-----------|
| UTILID    | Sim       | Identificador único da execução do utilitário |
| DBID      | Opcional  | Database ID (caso aplicável) |
| PSID      | Opcional  | Page Set ID (identifica o TS ou IX) |
| UTPROC    | Sim       | Tipo do utilitário: REORG, COPY, LOAD, etc. |
| STATUS    | Sim       | Define o estado: `STARTED` ou `STOPPED` |

> ⚠️ Recomenda-se que o par `DBID` e `PSID` sejam utilizados quando disponíveis, para garantir precisão na operação.

---

## 5. Exemplo de JCL

```jcl
//DEFUTIL  JOB (ACCT),'DEFUTIL JOB',CLASS=A,MSGCLASS=X
//STEP01   EXEC PGM=DSNUTILB,PARM='DB2P,DEFUTIL'
//STEPLIB  DD DSN=DSNEXIT,DISP=SHR
//         DD DSN=DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  DEFUTIL
    UTILID(REORG_CLIENTES_001)
    DBID(0091)
    PSID(0022)
    UTPROC(REORG)
    STATUS(STOPPED)
/*
```

---

## 6. Exemplos práticos de uso

### 6.1 Parar execução "presa"

```plaintext
DEFUTIL
  UTILID(LOAD_CLIENTES)
  STATUS(STOPPED)
```

### 6.2 Simular execução ativa

```plaintext
DEFUTIL
  UTILID(FAKE_REORG_TESTE)
  DBID(0099)
  PSID(0010)
  UTPROC(REORG)
  STATUS(STARTED)
```

### 6.3 Corrigir utilitário interrompido

```plaintext
DEFUTIL
  UTILID(COPY_DIA01)
  DBID(0001)
  PSID(0005)
  UTPROC(COPY)
  STATUS(STOPPED)
```

---

## 7. Cuidados e riscos ao utilizar

- Use **apenas com autorização e conhecimento técnico**;
- Verifique antes a tabela `SYSIBM.SYSUTIL` com `SELECT` para não sobrescrever entradas válidas;
- O uso incorreto pode causar falhas em futuros utilitários ou travamentos;
- Não utilize em ambientes produtivos sem análise prévia de impacto.

---

## 8. Consultas na SYSUTIL para diagnóstico

### Verificar todas as utilid pendentes:

```sql
SELECT UTILID, UTSTATUS, UTDBID, UTPSID, UTPROC
FROM SYSIBM.SYSUTIL
WHERE UTSTATUS <> 'STOPPED';
```

### Diagnóstico de conflitos

```sql
SELECT *
FROM SYSIBM.SYSUTIL
WHERE UTILID = 'NOME_DA_UTILID';
```

> 💡 Dica: A falta de `STOPPED` impede a execução subsequente de utilitários com o mesmo `UTILID`.

---

## 9. Referências oficiais IBM

- [DEFUTIL Utility - IBM v13](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-defutil-utility)
- [SYSIBM.SYSUTIL - DB2 Catalog Table](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-sysutil)

---
