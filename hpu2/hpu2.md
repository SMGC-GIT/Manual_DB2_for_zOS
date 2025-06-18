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
- [1️⃣1️⃣ Parâmetros Gerais do HPU](#1️⃣1️⃣-parâmetros-gerais-do-hpu)
- [1️⃣2️⃣ Detalhamento dos Parâmetros](#1️⃣2️⃣-detalhamento-dos-parâmetros)
- [1️⃣3️⃣ Restrições e Recursos no SELECT](#1️⃣3️⃣-restrições-e-recursos-no-select)
- [1️⃣4️⃣ Referência IBM](#1️⃣4️⃣-referência-ibm)
- [1️⃣5️⃣ Exemplo completo de JCL com parâmetros](#1️⃣5️⃣-exemplo-completo-de-jcl-com-parâmetros)
- [1️⃣6️⃣ Consistência DB2 LOCK e QUIESCE](#1️⃣6️⃣-consistência-db2-lock-e-quiesce)

  
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

### 1️⃣1️⃣ Parâmetros Gerais do HPU

| Parâmetro       | Descrição |
|-----------------|-----------|
| `DB2`           | Especifica o subsistema DB2. Ex: `DB2(DB2A)` |
| `LOCK`          | Define uso de bloqueio durante leitura |
| `QUIESCE`       | Define uso de QUIESCE para consistência |
| `OUTDDN`        | Nome lógico para o DD de saída |
| `FORMAT`        | Tipo de formato do unload |
| `DELIMITER`     | Caractere separador para arquivos delimitados |
| `NULL INDICATOR`| Como tratar valores nulos na saída |
| `ORDER BY`      | Ordenação dos dados exportados |
| `MAXRC`         | Código máximo de retorno permitido |
| `INCLUDE HEADER`| Adiciona linha com nome das colunas |

---

### 1️⃣2️⃣ Detalhamento dos Parâmetros

#### 🔹 `DB2(DB2A)`
Indica o nome do subsistema DB2 no qual o HPU deve se conectar. Obrigatório.

```text
DB2(DB2A)
```

#### 🔹 `LOCK YES | NO`
Controla se os dados devem ser bloqueados durante o unload.  
- `YES`: impede que outros processos atualizem dados simultaneamente  
- `NO`: mais rápido, mas pode gerar inconsistência se houver alterações

```text
LOCK YES
```

#### 🔹 `QUIESCE YES | NO`
Garante que o unload ocorra após um ponto de consistência (QUIESCE).  
Evita dados inconsistentes entre páginas.

```text
QUIESCE YES
```

#### 🔹 `OUTDDN(SAIDA1)`
Define o nome lógico do `DD` onde o unload será gravado.  
Esse nome deve existir no JCL.

```text
OUTDDN(SAIDA1)
```

#### 🔹 `FORMAT DELIMITED | INTERNAL | CSV`
Indica o formato do arquivo de saída.  
- `DELIMITED`: dados separados por delimitador definido  
- `INTERNAL`: formato binário interno do DB2  
- `CSV`: similar ao delimitado, com vírgula e aspas por padrão

```text
FORMAT DELIMITED
```

#### 🔹 `DELIMITER ','`
Define o caractere separador quando `FORMAT` for `DELIMITED` ou `CSV`.

```text
DELIMITER ','
```

#### 🔹 `NULL INDICATOR 'NULL'`
Define como os campos nulos são representados no arquivo de saída.

```text
NULL INDICATOR 'NULL'
```

#### 🔹 `ORDER BY COLUNA1`
Permite ordenar os registros descarregados. Pode impactar performance.

```text
ORDER BY COD_CLIENTE
```

#### 🔹 `MAXRC 8`
Define o maior código de retorno aceito para considerar o job como concluído com sucesso.

```text
MAXRC 8
```

#### 🔹 `INCLUDE HEADER`
Inclui uma linha no início com os nomes das colunas.

```text
INCLUDE HEADER
```

---

### 1️⃣3️⃣ Restrições e Recursos no SELECT

O HPU suporta apenas um subconjunto da linguagem SQL.

#### ✅ Permitido

- `SELECT coluna1, coluna2`
- `SELECT *`
- `WHERE` simples
- `ORDER BY`
- `AND` / `OR`

#### ❌ Não Suportado

| Restrição                     | Motivo |
|------------------------------|--------|
| `JOIN` entre tabelas         | Apenas uma tabela por vez |
| `GROUP BY`, `HAVING`         | Agregações não são suportadas |
| `UNION`, `INTERSECT`         | Combinação de conjuntos é proibida |
| `CASE`, `COALESCE`           | Funções complexas não aceitas |
| Subqueries (`SELECT dentro`) | Proibido |
| `CAST`, `CONCAT`, `SUBSTR`   | Funções de string restritas |

💡 **Dica:** Utilize o unload para descarregar tudo e aplique filtros complexos fora do DB2 (Python, shell script, SQL Server, etc.)

---

### 1️⃣4️⃣ Referência IBM

- 🔗 [IBM HPU SYSIN Syntax](https://www.ibm.com/docs/en/db2-hpu/5.1?topic=commands-sysin-control-statements)
- 🔗 [IBM HPU SQL Support](https://www.ibm.com/docs/en/db2-hpu/5.1?topic=statements-sql-control)

---

### 1️⃣5️⃣ Exemplo completo de JCL com parâmetros

```jcl
//HPUJOB  JOB ...
//STEP01  EXEC PGM=IBMHPU,PARM='DB2A'
//STEPLIB DD DSN=IBM.HPU.LOADLIB,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSUDUMP DD SYSOUT=*
//OUTDD   DD DSN=USR.UNLOAD.CLIENTES.CSV,DISP=(NEW,CATLG,DELETE),
//          SPACE=(CYL,(20,5)),UNIT=SYSDA
//SYSIN   DD *
 UNLOAD
   FROM TABLESPACE DB2DB01.CLIENTES
   TO OUTDDN(OUTDD)
   FORMAT CSV
   DELIMITER ','
   NULL INDICATOR 'NULL'
   INCLUDE HEADER
   DB2 FORCE
   LOCK YES
   QUIESCE FORCE
   COPYDDN LAST_IC
   INDEXSCAN
   ORDER BY ID_CLIENTE
   MAXRC 8
/*
```

---

### 1️⃣6️⃣ Consistência DB2 LOCK e QUIESCE

A consistência dos dados durante o processo de *unload* com o IBM HPU depende diretamente da combinação dos parâmetros `DB2`, `LOCK` e `QUIESCE`. Esses parâmetros definem o **nível de concorrência**, **tipo de acesso** e **garantia de integridade transacional** durante a extração de dados.

Abaixo está uma tabela seguida de explicações detalhadas sobre cada combinação possível e seus impactos.

| Combinação                      | Resultado |
|----------------------------------|-----------|
| DB2 NO + LOCK NO                 | Máximo desempenho, risco de inconsistência |
| DB2 NO + LOCK YES                | Lock local só em I/O físico |
| DB2 YES + LOCK YES               | Garantia de consistência via LOCK |
| DB2 YES + QUIESCE YES            | Consistência forte com QUIESCE |
| DB2 FORCE + QUIESCE/LOCK YES     | Total segurança, possivelmente offline |

---

#### 🔍 Tabela Resumo

| DB2       | LOCK   | QUIESCE | Garantia de Consistência | Desempenho | Observações Técnicas |
|-----------|--------|---------|---------------------------|------------|-----------------------|
| NO        | NO     | -       | ❌ Nenhuma                | 🔋 Máxima   | Sem uso de catálogo DB2, sem locks. Risco de dados inconsistentes se houver alterações simultâneas. |
| NO        | YES    | -       | ⚠️ Parcial (I/O Level)    | 🔋 Alta     | Lock é feito em nível de leitura física. Pode evitar alterações simultâneas no disco, mas sem consistência lógica transacional. |
| YES       | NO     | NO      | ⚠️ Mínima (SQL Snapshot)  | ⚖️ Média    | Usa SQL via DB2 sem lock nem quiesce. Pode haver inconsistências se os dados forem alterados após a geração do plano. |
| YES       | YES    | NO      | ✅ Boa (Transação DB2)    | ⚖️ Média    | Lock via SQL. Garante consistência lógica transacional para as páginas acessadas, sem quiesce. |
| YES       | -      | YES     | ✅ Alta (Quiesce ativo)   | ⚖️ Baixa    | Interrompe momentaneamente updates/inserts. Ideal para dados críticos. |
| YES       | -      | FORCE   | ✅ Alta (STOP/START)      | ⚠️ Baixa    | Se QUIESCE falhar, executa STOP/START. Pode impactar disponibilidade. |
| YES       | -      | TRY     | ⚠️ Parcial (Melhor esforço) | ⚖️ Média | Tenta QUIESCE. Se falhar, prossegue conforme LOCK. Sem erro fatal. |
| FORCE     | YES    | FORCE   | ✅🔒 Máxima (Offline)     | ❗ Muito Baixa | Garante integridade total via parada da instância ou TS. Altamente intrusivo, só recomendado em janelas de manutenção. |

---

#### 📘 Explicações Detalhadas

##### 🔸 DB2(NO)
- O HPU opera em modo **nativo**, sem chamar o DB2.
- Acesso direto ao VSAM (dataset físico).
- Alta performance, mas nenhuma verificação de integridade relacional é feita.
- Pode capturar dados em meio a transações abertas → risco de "dirty reads".

##### 🔸 DB2(YES)
- O HPU utiliza **acesso lógico via SQL** e consulta o catálogo do DB2.
- O acesso é semelhante ao de uma aplicação tradicional (por exemplo, uma query via SPUFI).
- Permite aplicação de **locks lógicos** e uso de QUIESCE.

##### 🔸 DB2(FORCE)
- Obriga o uso do DB2, mesmo que o unload direto seja possível.
- Ideal para situações onde consistência é prioritária.

---

##### 🔸 LOCK(NO)
- Nenhuma tentativa de aplicar lock é feita.
- O unload pode ocorrer em paralelo com atualizações (inserts, deletes, updates).

##### 🔸 LOCK(YES)
- O HPU tenta aplicar locks em nível de página ou objeto.
- Se DB2=NO, o lock é tentado no nível de sistema de arquivos (I/O).
- Se DB2=YES ou FORCE, os locks são aplicados via comandos SQL (mais confiáveis).

---

##### 🔸 QUIESCE(NO)
- Não há tentativa de pausar a atividade do objeto/tablespace.
- Risco de inconsistência caso os dados sejam alterados durante o unload.

##### 🔸 QUIESCE(YES)
- O HPU executa o comando `QUIESCE` para garantir que o objeto esteja em estado consistente.
- Interrompe temporariamente transações abertas ou impede novos acessos.
- Mais seguro para unloads críticos.

##### 🔸 QUIESCE(FORCE)
- Caso QUIESCE falhe, o HPU executa `STOP DATABASE` e `START DATABASE` no objeto alvo.
- Pode indisponibilizar temporariamente a base para outras aplicações.
- Requer autoridade SYSADM e planejamento de janela.

##### 🔸 QUIESCE(TRY)
- Tenta o QUIESCE. Se falhar, o comportamento depende do parâmetro LOCK.
  - LOCK=NO → prossegue com warning.
  - LOCK=YES → pode gerar erro ou fallback para locks leves.

---

### ✅ Boas Práticas

- **Para consistência transacional sem downtime**, use `DB2(YES)` + `LOCK(YES)`.
- **Para extrações críticas**, use `DB2(YES)` + `QUIESCE(YES)` ou `FORCE`.
- **Evite `DB2(NO) LOCK(NO)` em bases com alta concorrência**.
- Sempre valide se os objetos estão ativos antes de usar `QUIESCE(FORCE)`.

---

### 🔗 Referência IBM

- [HPU - Parâmetros DB2, LOCK e QUIESCE](https://www.ibm.com/docs/en/dhpufz/5.1.0?topic=commands-sysin-control-statements)
- [QUIESCE DB2 Command](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-quiesce)
- [STOP/START DB2 Command](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-stop-database)


