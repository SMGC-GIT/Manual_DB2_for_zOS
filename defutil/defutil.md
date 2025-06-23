# Utilitário DEFUTIL - DB2 for z/OS

---

## 📚 Índice

- [1. O que é o utilitário DEFUTIL](#1-o-que-é-o-utilitário-defutil)
- [2. Quando usar o DEFUTIL](#2-quando-usar-o-defutil)
- [3. Como funciona o DEFUTIL](#3-como-funciona-o-defutil)
- [4. Sintaxe e parâmetros](#4-sintaxe-e-parâmetros)
- [5. Exemplo de JCL](#5-exemplo-de-jcl)
- [6. Exemplos práticos de uso](#6-exemplos-práticos-de-uso)
- [7. Cuidados e riscos ao utilizar](#7-cuidados-e-riscos-ao-utilizar)
- [8. Consultas na SYSUTIL para diagnóstico](#8-consultas-na-sysutil-para-diagnóstico)
- [9. Campos adicionais da SYSUTIL (controle interno)](#9-campos-adicionais-da-sysutil-controle-interno)
- [10. Relação com DISPLAY, MODIFY e UTILID](#10-relação-com-display-modify-e-utilid)
- [11. Referências oficiais IBM](#11-referências-oficiais-ibm)

---

## 1. O que é o utilitário DEFUTIL

O `DEFUTIL` (Define Utility) é um utilitário administrativo interno do DB2 for z/OS que permite criar, alterar ou encerrar manualmente entradas na tabela `SYSIBM.SYSUTIL`, responsável por controlar execuções ativas de utilitários como `REORG`, `COPY`, `LOAD`, entre outros.

---

## 2. Quando usar o DEFUTIL

- **Execução de utilitário falhou** e a entrada permaneceu como "ativa"
- **Bloqueio de novos jobs** com mesmo `UTILID`
- **Ambiente de testes** que simula jobs ativos
- **Correções administrativas** pós-falha

---

## 3. Como funciona o DEFUTIL

O DEFUTIL **altera diretamente** a tabela `SYSIBM.SYSUTIL`, usando parâmetros como `UTILID`, `UTPROC` e `STATUS` para atualizar o estado de execução de um utilitário.

### Exemplo de estrutura de execução:

| Coluna     | Descrição                                     |
|------------|-----------------------------------------------|
| UTILID     | Nome da execução do utilitário                |
| UTSTATUS   | STARTED / STOPPED                             |
| UTPROC     | Tipo do utilitário (REORG, COPY, etc.)        |
| UTDBID     | Database ID                                   |
| UTPSID     | Page Set ID (TS ou IX)                        |
| UTSTRT     | Timestamp de início                           |
| UTSTOP     | Timestamp de parada (se STOPPED)              |
| UTUTIME    | Tempo total de execução (em microssegundos)   |

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

### Descrição dos parâmetros

| Parâmetro | Obrigatório | Descrição |
|-----------|-------------|-----------|
| UTILID    | ✅           | Nome lógico da execução |
| DBID      | ⚠️ Opcional | ID do banco de dados (quando aplicável) |
| PSID      | ⚠️ Opcional | Page Set ID (TS ou IX) |
| UTPROC    | ✅           | Tipo: REORG, COPY, LOAD, etc. |
| STATUS    | ✅           | STARTED ou STOPPED |

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

### Parar execução presa

```plaintext
DEFUTIL
  UTILID(LOAD_CLIENTES)
  STATUS(STOPPED)
```

### Simular execução ativa

```plaintext
DEFUTIL
  UTILID(FAKE_REORG_TESTE)
  DBID(0099)
  PSID(0010)
  UTPROC(REORG)
  STATUS(STARTED)
```

---

## 7. Cuidados e riscos ao utilizar

- **Evite uso sem diagnóstico prévio** da `SYSUTIL`
- **Risco de sobrescrever registros válidos**
- **Pode travar futuros utilitários se mal configurado**
- **Ambientes produtivos requerem extrema cautela**

---

## 8. Consultas na SYSUTIL para diagnóstico

```sql
-- Exibir utilitários ativos
SELECT UTILID, UTSTATUS, UTPROC, UTSTRT
FROM SYSIBM.SYSUTIL
WHERE UTSTATUS = 'STARTED';

-- Verificar execuções pendentes
SELECT *
FROM SYSIBM.SYSUTIL
WHERE UTSTATUS <> 'STOPPED';
```

---

## 9. Campos adicionais da SYSUTIL (controle interno)

| Campo     | Descrição                                     |
|-----------|-----------------------------------------------|
| UTSTRT    | Timestamp de início                           |
| UTSTOP    | Timestamp de parada                           |
| UTPROC    | Nome do utilitário                            |
| UTUTIME   | Duração (em microssegundos)                   |
| UTPROCID  | ID do processo responsável                    |
| UTLASTS   | Última atualização da entrada                 |

> ⚠️ **Esses campos não podem ser modificados diretamente. Use DEFUTIL com cautela para STATUS apenas.**

---

## 10. Relação com DISPLAY, MODIFY e UTILID

| Comando           | Função                                                |
|-------------------|--------------------------------------------------------|
| `DISPLAY UTILITY` | Exibe utilitários em execução com detalhes técnicos    |
| `MODIFY UTILITY`  | Força parada de utilitário preso                       |
| `DEFUTIL`         | Altera status lógico de uma execução (manual)          |

### Exemplo `DISPLAY UTILITY`

```plaintext
-DIS UTIL(*)
```

### Exemplo `MODIFY UTILITY`

```plaintext
-MODIFY UTILITY (UTILID(REORG_CLIENTES_001)) TERM
```

---

## 11. Referências oficiais IBM

- [DEFUTIL Utility - IBM v13](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-defutil-utility)
- [SYSUTIL - IBM Catalog Table](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-sysutils-table)
- [MODIFY UTILITY - IBM Command](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-modify-utility)
- [DISPLAY UTILITY - IBM Command](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-display-utility)

---

