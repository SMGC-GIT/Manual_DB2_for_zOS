# 💽 HPU – High Performance Unload

## 📚 Índice

- [1️⃣ O que é o HPU](#1️⃣-o-que-é-o-hpu)
- [2️⃣ Quando e Por Que Usar](#2️⃣-quando-e-por-que-usar)
- [3️⃣ Características do HPU](#3️⃣-características-do-hpu)
- [4️⃣ Exemplo Básico de JCL com HPU](#4️⃣-exemplo-básico-de-jcl-com-hpu)
- [5️⃣ Sintaxe e Parâmetros do SYSIN](#5️⃣-sintaxe-e-parâmetros-do-sysin)
- [6️⃣ Exemplos Avançados](#6️⃣-exemplos-avançados)
- [7️⃣ Dicas de Performance e Boas Práticas](#7️⃣-dicas-de-performance-e-boas-práticas)
- [8️⃣ Erros Comuns e Soluções](#8️⃣-erros-comuns-e-soluções)
- [9️⃣ Referências Oficiais IBM](#9️⃣-referências-oficiais-ibm)

---

## 1️⃣ O que é o HPU

**High Performance Unload (HPU)** é uma ferramenta IBM para descarregar dados do DB2 for z/OS diretamente dos datasets VSAM (tabelas organizadas fisicamente), sem passar pelo subsistema do DB2.

### ✳️ Benefícios:
- Desempenho muito superior ao UNLOAD tradicional.
- Evita overhead do subsistema DB2.
- Ideal para grandes volumes de dados.

---

## 2️⃣ Quando e Por Que Usar

- 💡 Quando precisa descarregar grandes quantidades de dados rapidamente.
- 🔐 Quando o subsistema DB2 está indisponível.
- 📦 Para exportar dados para outros ambientes ou para arquivamento.
- 🔄 Como parte de rotinas de backup lógico.

---

## 3️⃣ Características do HPU

| Característica            | Detalhe |
|---------------------------|---------|
| **Acesso direto**         | Lê os dados diretamente dos datasets físicos (VSAM) |
| **Independente do DB2**   | Não depende do subsistema estar ativo |
| **Alta performance**      | Pode ser 3 a 10 vezes mais rápido que UNLOAD |
| **Multiplataforma**       | Permite exportar dados para uso em outras bases |

---

## 4️⃣ Exemplo Básico de JCL com HPU

```jcl
//UNLOAD   JOB (ACCT),'HPU UNLOAD JOB',CLASS=A,MSGCLASS=X
//STEP01   EXEC PGM=INZUTILB,REGION=0M,
//             PARM=('DB2A','HPUUTIL')
//STEPLIB  DD DSN=DSNEXIT,DISP=SHR
//         DD DSN=DSNLOAD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
 UNLOAD
   DATA DDNAME(SAIDA)
   TABLESPACE DB2DB01.TSCLIENTE
   SELECT *
/*
//SAIDA    DD DSN=MY.OUTPUT.FILE,
//            DISP=(NEW,CATLG,DELETE),
//            UNIT=SYSDA,SPACE=(CYL,(50,10)),
//            DCB=(RECFM=FB,LRECL=1000,BLKSIZE=10000)
```

---

## 5️⃣ Sintaxe e Parâmetros do SYSIN

### Principais comandos:

- `UNLOAD`: inicia a operação.
- `TABLESPACE`: tablespace a ser descarregado.
- `DATA DDNAME(...)`: nome DD de saída.
- `SELECT`: cláusula para selecionar colunas/linhas.

### Exemplo com WHERE e ordenação:
```plaintext
 UNLOAD
   DATA DDNAME(SAIDA)
   TABLESPACE DB2DB01.TSCLIENTE
   SELECT CPF, NOME, CIDADE
   WHERE ESTADO = 'SP'
   ORDER BY CIDADE
```

---

## 6️⃣ Exemplos Avançados

### 🧾 Exemplo com saída CSV:

```plaintext
 UNLOAD
   DATA DDNAME(SAIDA)
   FORMAT DELIMITED
   DELIMITER ','
   NULL INDICATOR 'NULL'
   SELECT *
```

### 🔍 Exemplo descarregando apenas registros ativos (com filtro):
```plaintext
 UNLOAD
   DATA DDNAME(SAIDA)
   SELECT * 
   WHERE STATUS = 'A'
```

### 🧱 Exemplo por índice específico:
```plaintext
 UNLOAD
   INDEX DB2DB01.IXCLIENTE1
   DATA DDNAME(SAIDA)
   SELECT *
```

---

## 7️⃣ Dicas de Performance e Boas Práticas

- ✅ Utilize `FORMAT INTERNAL` para máxima performance (somente para reimportação no mesmo sistema).
- ✅ Separe cargas grandes em múltiplos jobs em paralelo.
- ✅ Sempre defina `LRECL` e `BLKSIZE` otimizados na DD de saída.
- ✅ Evite `ORDER BY` se não for necessário.
- 🔄 Atualize estatísticas com `RUNSTATS` após grandes unloads.

---

## 8️⃣ Erros Comuns e Soluções

| Código / Mensagem       | Causa Provável                                     | Solução |
|--------------------------|---------------------------------------------------|---------|
| `DSNU020I`               | Problema com espaço em disco                      | Aumentar allocation da DD |
| `INZU010I`               | Syntax error no SYSIN                             | Revisar a cláusula UNLOAD |
| `IEC030I`                | Arquivo não encontrado ou sem permissão           | Verifique DDNAME e permissões RACF |
| `DATASET IN USE`         | Dataset em uso por outro job                      | Liberar lock com -CANCEL THREAD |
| `RC 08` com erro de INDEX| Índice não encontrado ou não disponível           | Checar definição da tabela e tablespace |

---

## 9️⃣ Referências Oficiais IBM

- 📄 [IBM HPU for Db2 for z/OS 5.1 - Overview](https://www.ibm.com/docs/en/db2-hpu/5.1?topic=zos-high-performance-unload-db2)
- 📄 [HPU SYSIN Syntax - IBM Documentation](https://www.ibm.com/docs/en/db2-hpu/5.1?topic=commands-sysin-control-statements)

---
