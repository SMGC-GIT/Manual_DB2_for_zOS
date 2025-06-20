# 🚛 Tipos de LOAD no DB2 for z/OS

---

## 📚 Índice

- [🔹 LOAD REPLACE](#-load-replace)
- [🔹 LOAD RESUME YES](#-load-resume-yes)
- [🔹 LOAD SHRLEVEL NONE](#-load-shrlevel-none)
- [🔹 LOAD SHRLEVEL CHANGE](#-load-shrlevel-change)
- [🔹 LOAD WITH SHRLEVEL REFERENCE](#-load-with-shrlevel-reference)
- [🔹 Comparativo Geral dos Tipos](#-comparativo-geral-dos-tipos)
- [🔹 Considerações e Dicas Técnicas](#-considerações-e-dicas-técnicas)
- [🔹 Erros Frequentes e Soluções](#-erros-frequentes-e-soluções)
- [🔹 Templates Prontos de JCL](#-templates-prontos-de-jcl)
- [🔹 Casos de Uso por Ambiente](#-casos-de-uso-por-ambiente)
- [🔹 Planilha Técnica de Decisão](#-planilha-técnica-de-decisão)
- [🔹 Referências IBM](#-referências-ibm)

---

## 🔹 LOAD REPLACE

### 📌 Descrição
Substitui todos os dados da tabela. Ideal para cargas iniciais ou reinicializações periódicas.

### ✅ Quando Usar
- Reset de ambiente.
- Garantia de consistência total dos dados.

### 🚫 Restrições
- Remove todos os dados existentes.
- Requer cuidado com foreign keys e triggers.

### 💻 Exemplo de JCL

```jcl
//LOADREPL JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=DB2.WORK.LOAD1,UNIT=SYSDA,SPACE=(CYL,(10,5))
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       REPLACE
       INTO TABLE DB2P.CLIENTES
       (ID POSITION(1:5) INTEGER,
        NOME POSITION(6:25) CHAR)
       LOG NO
/*
```

---

## 🔹 LOAD RESUME YES

### 📌 Descrição
Acrescenta dados à tabela sem apagar os existentes. Suporta cargas frequentes.

### ✅ Quando Usar
- Cargas parciais.
- Cargas incrementais (ETL, integração).

### 🚫 Restrições
- Pode gerar duplicidade.
- Necessário cuidado com integridade de índices.

### 💻 Exemplo de JCL

```jcl
//LOADRESM JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DSN=DB2.WORK.LOAD2,UNIT=SYSDA,SPACE=(CYL,(10,5))
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       RESUME YES
       INTO TABLE DB2P.PEDIDOS
       (ID_PEDIDO POSITION(1:5) INTEGER,
        VALOR    POSITION(6:15) DECIMAL)
       LOG YES
/*
```

---

## 🔹 LOAD SHRLEVEL NONE

### 📌 Descrição
Exclusividade total do objeto. Máxima performance, nenhum acesso permitido durante LOAD.

### 💻 Exemplo de JCL

```jcl
//LOADSHRN JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       REPLACE
       SHRLEVEL NONE
       INTO TABLE DB2P.CONTAS
       (CONTA_ID POSITION(1:5) CHAR,
        SALDO    POSITION(6:15) DECIMAL)
       LOG NO
/*
```

---

## 🔹 LOAD SHRLEVEL CHANGE

### 📌 Descrição
Permite acesso simultâneo controlado (read/write). Balanceia disponibilidade e performance.

### 💻 Exemplo de JCL

```jcl
//LOADCHNG JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       RESUME YES
       SHRLEVEL CHANGE
       INTO TABLE DB2P.VENDAS
       (ID_VENDA POSITION(1:5) INTEGER,
        VALOR    POSITION(6:15) DECIMAL)
       LOG YES
/*
```

---

## 🔹 LOAD WITH SHRLEVEL REFERENCE

### 📌 Descrição
Permite somente leituras durante o LOAD. Ideal para ambientes críticos.

### 💻 Exemplo de JCL

```jcl
//LOADREF JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       SHRLEVEL REFERENCE
       RESUME YES
       INTO TABLE DB2P.PRODUTOS
       (COD POSITION(1:5) CHAR,
        DESC POSITION(6:25) CHAR)
       LOG YES
/*
```

---

## 🔹 Comparativo Geral dos Tipos

| Tipo de LOAD             | Apaga Dados? | Permite Acesso?   | Performance | Ideal Para                         |
|--------------------------|---------------|--------------------|-------------|-------------------------------------|
| REPLACE                 | Sim           | Não                | Alta        | Reset ou reinicialização            |
| RESUME YES              | Não           | Sim                | Média       | Carga incremental                   |
| SHRLEVEL NONE           | -             | Não                | Muito Alta  | Janela de carga exclusiva           |
| SHRLEVEL CHANGE         | -             | Sim (parcial)      | Média       | Manutenção com disponibilidade      |
| SHRLEVEL REFERENCE      | -             | Leitura somente    | Alta        | Ambientes críticos de leitura       |

---

## 🔹 Considerações e Dicas Técnicas

- Use `LOG NO` para dados temporários ou reprocessáveis.
- Utilize `DISCARDDN` para capturar registros inválidos.
- `KEEPDICTIONARY YES` reduz overhead ao usar compressão.
- Use `INLINE STATISTICS` para evitar execução posterior de `RUNSTATS`.
- Após `LOAD`, considere `REBUILD INDEX` para garantir acesso eficiente.

---

## 🔹 Erros Frequentes e Soluções

| Código/Erro               | Causa Comum                                         | Solução Sugerida                                     |
|---------------------------|------------------------------------------------------|------------------------------------------------------|
| DSNU334I                  | Incompatibilidade de tipos no SYSREC                 | Verifique formatação e posições dos dados            |
| DSNU1237I                 | Tabela está em CHECK PENDING                         | Execute `CHECK DATA` ou `REPAIR`                     |
| RC=8 após LOAD            | Dados inconsistentes ou inválidos                    | Use `DISCARDDN` para isolar registros problemáticos  |
| LOAD trava                | SHRLEVEL inadequado para workload                    | Ajuste para `REFERENCE` ou `CHANGE`                  |
| DSNU016I - ABEND=04E      | Falha de bufferpool ou dataset                       | Revise espaço disponível e permissões                |

---

## 🔹 Templates Prontos de JCL

### ✅ LOAD com DISCARD

```jcl
//LOADDIS JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//DISCARD  DD DSN=DB2P.PEDIDOS.DISCARD,DISP=(NEW,CATLG),
//            UNIT=SYSDA,SPACE=(TRK,(10,5))
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       DISCARDMAX 100
       DISCARDDN(DISCARD)
       RESUME YES
       SHRLEVEL CHANGE
       INTO TABLE DB2P.PEDIDOS
       (ID_PEDIDO POSITION(1:5) INTEGER,
        VALOR POSITION(6:15) DECIMAL)
/*
```

### ✅ LOAD com INLINE STATISTICS

```jcl
//LOADST JOB ...
//STEP01 EXEC PGM=DSNUTILB,PARM='DB2P,LOAD'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LOAD DATA INDDN(SYSREC)
       SHRLEVEL NONE
       REPLACE
       INTO TABLE DB2P.LOGINS
       (LOGIN_ID POSITION(1:5) CHAR,
        ACESSO   POSITION(6:25) TIMESTAMP)
       STATISTICS YES
       INDEXSTAT YES
/*
```

---

## 🔹 Casos de Uso por Ambiente

| Ambiente | Estratégia Recomendada                      | Observações |
|----------|---------------------------------------------|-------------|
| DEV      | REPLACE com LOG NO                          | Reprocessável, ideal para testes |
| QA       | RESUME YES + DISCARD + INLINE STATISTICS    | Validação de dados e performance |
| PROD     | SHRLEVEL REFERENCE com LOG YES              | Alta disponibilidade e segurança |

---

## 🔹 Planilha Técnica de Decisão

| Critério                       | Baixo Volume | Médio Volume | Alto Volume |
|-------------------------------|--------------|---------------|-------------|
| Disponibilidade necessária    | REPLACE      | SHRLEVEL CHANGE | SHRLEVEL REFERENCE |
| Dados reprocessáveis?         | LOG NO       | LOG NO        | LOG NO      |
| Dados críticos e únicos?      | LOG YES      | DISCARD/LOG   | DISCARD/LOG |
| Tabela com triggers/constraints | Verificar dependências antes do LOAD |
| Índices existentes?           | INLINE STATISTICS ou REBUILD INDEX após carga |
| Janela de manutenção          | SHRLEVEL NONE preferido se permitido downtime |

---

## 🔹 Referências IBM

- 📘 [LOAD Utility - IBM Docs](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-load-utility)  
- 📘 [LOAD Considerations](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-load-utility-considerations)  
- 📘 [Db2 for z/OS Utility Reference](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-utility-statements)  

