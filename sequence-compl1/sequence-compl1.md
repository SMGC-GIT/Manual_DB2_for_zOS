# Manual Técnico: SEQUENCE no DB2 for z/OS

## Índice

1. [Conceito e Objetivo da SEQUENCE](#1-conceito-e-objetivo-da-sequence)
2. [Sintaxe e Criação de uma SEQUENCE](#2-sintaxe-e-criação-de-uma-sequence)
3. [Propriedades Técnicas da SEQUENCE](#3-propriedades-técnicas-da-sequence)
4. [Como Consultar e Monitorar SEQUENCES](#4-como-consultar-e-monitorar-sequences)
5. [Como Saber se um Campo Está Associado a uma SEQUENCE](#5-como-saber-se-um-campo-está-associado-a-uma-sequence)
6. [Utilização em Programas COBOL, SQL Dinâmico e SP](#6-utilização-em-programas-cobol-sql-dinâmico-e-sp)
7. [Testes Práticos em Ambiente DEV](#7-testes-práticos-em-ambiente-dev)
8. [Cuidados com LOAD e RESTART](#8-cuidados-com-load-e-restart)
9. [Boas Práticas e Estratégias de Produção](#9-boas-práticas-e-estratégias-de-produção)
10. [Gerenciamento de Memória e Cache](#10-gerenciamento-de-memória-e-cache)
11. [Troubleshooting e Resolução de Problemas](#11-troubleshooting-e-resolução-de-problemas)
12. [Migração de IDENTITY para SEQUENCE](#12-migração-de-identity-para-sequence)
13. [Referências Oficiais IBM](#13-referências-oficiais-ibm)

---

## 1. Conceito e Objetivo da SEQUENCE

Uma SEQUENCE é um objeto do DB2 for z/OS criado para gerar números sequenciais únicos, típicos para uso como identificadores primários (PK) em tabelas. Substitui com vantagens o IDENTITY COLUMN, oferecendo maior controle, reuso entre tabelas, persistência e escalabilidade.

### Por que usar SEQUENCE?

- **Evita concorrência e gargalos** com `MAX(COL)+1`
- **É reutilizável** por várias tabelas
- **Permite controle detalhado** do intervalo e incremento
- **Suporte a ambientes de alta concorrência** e grandes volumes
- **Melhor performance** em comparação com métodos tradicionais
- **Flexibilidade** para alterações sem impacto na estrutura da tabela

### Diferenças entre SEQUENCE e IDENTITY COLUMN

| Característica | SEQUENCE | IDENTITY COLUMN |
|----------------|----------|-----------------|
| **Escopo** | Independente, pode ser usada em múltiplas tabelas | Vinculada a uma única coluna |
| **Flexibilidade** | Permite alterações sem DDL da tabela | Requer ALTER TABLE para modificações |
| **Reuso** | Pode ser compartilhada entre tabelas | Específica para uma tabela |
| **Controle** | Controle granular sobre quando gerar valores | Automático no INSERT |
| **Cache** | Configurável individualmente | Dependente das configurações da tabela |

---

## 2. Sintaxe e Criação de uma SEQUENCE

### Sintaxe Completa

```sql
CREATE SEQUENCE [schema.]sequence-name
    [AS data-type]
    [START WITH numeric-constant]
    [INCREMENT BY numeric-constant]
    [MINVALUE numeric-constant | NO MINVALUE]
    [MAXVALUE numeric-constant | NO MAXVALUE]
    [CYCLE | NO CYCLE]
    [CACHE integer-constant | NO CACHE]
    [ORDER | NO ORDER];
```

### Exemplo Prático

```sql
CREATE SEQUENCE DBATEST.SEQ_CLIENTE_ID
    AS BIGINT
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9999999999
    NO CYCLE
    CACHE 100
    ORDER;
```

### Explicação Detalhada dos Parâmetros

| Parâmetro | Descrição | Padrão | Considerações |
|-----------|-----------|---------|---------------|
| **AS data-type** | Tipo de dados (INTEGER, BIGINT, SMALLINT, DECIMAL) | INTEGER | BIGINT recomendado para alta volumetria |
| **START WITH** | Valor inicial da sequência | 1 ou -1 | Deve estar entre MINVALUE e MAXVALUE |
| **INCREMENT BY** | Incremento entre valores consecutivos | 1 | Pode ser negativo para sequências decrescentes |
| **MINVALUE** | Valor mínimo permitido | 1 para positivo, -999...999 para negativo | Define o limite inferior |
| **MAXVALUE** | Valor máximo permitido | 999...999 para positivo, -1 para negativo | Define o limite superior |
| **CYCLE/NO CYCLE** | Se deve reiniciar após atingir limite | NO CYCLE | CYCLE pode causar duplicações |
| **CACHE** | Quantidade de valores pré-alocados | 20 | Melhora performance, mas pode causar gaps |
| **ORDER/NO ORDER** | Garante ordem em ambientes paralelos | NO ORDER | ORDER impacta performance |

### Tipos de Dados Suportados

```sql
-- INTEGER (32-bit): -2,147,483,648 a 2,147,483,647
CREATE SEQUENCE SEQ_INT AS INTEGER;

-- BIGINT (64-bit): -9,223,372,036,854,775,808 a 9,223,372,036,854,775,807
CREATE SEQUENCE SEQ_BIGINT AS BIGINT;

-- SMALLINT (16-bit): -32,768 a 32,767
CREATE SEQUENCE SEQ_SMALLINT AS SMALLINT;

-- DECIMAL
CREATE SEQUENCE SEQ_DECIMAL AS DECIMAL(15,0);
```

---

## 3. Propriedades Técnicas da SEQUENCE

### Características Fundamentais

| Propriedade | Descrição | Impacto |
|-------------|-----------|---------|
| **Persistência** | Objeto persistente no catálogo DB2 | Sobrevive a reinicializações |
| **Independência** | Não ligada diretamente a nenhuma tabela | Flexibilidade de uso |
| **Concorrência** | Thread-safe para múltiplas sessões | Segurança em ambientes paralelos |
| **Bufferização (CACHE)** | Pré-alocação em memória para performance | Melhora throughput, pode causar gaps |
| **Semaforização** | Controle interno de acesso concorrente | Garante unicidade |
| **Transacional** | Não participa de ROLLBACK | Valores não são "devolvidos" |

### Comportamento Transacional

**Importante**: Os valores de SEQUENCE são **não-transacionais**. Uma vez obtido um valor com `NEXT VALUE FOR`, ele não retorna ao pool mesmo se a transação for revertida com ROLLBACK.

```sql
-- Exemplo de comportamento não-transacional
BEGIN;
    INSERT INTO CLIENTE VALUES (NEXT VALUE FOR SEQ_CLIENTE_ID, 'TESTE');
ROLLBACK;
-- O valor da SEQUENCE foi consumido mesmo com ROLLBACK
```

### Gerenciamento de Memória

O DB2 mantém informações da SEQUENCE em diferentes áreas:
- **Catálogo**: Definição persistente
- **Cache do Buffer Pool**: Valores pré-alocados
- **Memória Compartilhada**: Controle de concorrência

---

## 4. Como Consultar e Monitorar SEQUENCES

### Catálogo Principal - SYSIBM.SYSSEQUENCES

```sql
-- Consulta completa de uma SEQUENCE
SELECT NAME,
       SCHEMA,
       DATATYPEID,
       SOURCETYPEID,
       LENGTH,
       SCALE,
       START,
       INCREMENT,
       MINVALUE,
       MAXVALUE,
       CYCLE,
       CACHE,
       ORDER,
       CREATEDBY,
       CREATEDTS,
       ALTEREDBY,
       ALTEREDTS
FROM SYSIBM.SYSSEQUENCES 
WHERE NAME = 'SEQ_CLIENTE_ID'
  AND SCHEMA = 'DBATEST';
```

### Consulta de Valor Atual

```sql
-- Obter o próximo valor sem consumi-lo (DB2 12+)
SELECT PREVIOUS VALUE FOR DBATEST.SEQ_CLIENTE_ID 
FROM SYSIBM.SYSDUMMY1;

-- Verificar valor atual via catálogo
SELECT SEQCACHE,
       SEQCACHEHWM
FROM SYSIBM.SYSSEQUENCES
WHERE NAME = 'SEQ_CLIENTE_ID';
```

### Monitoramento de Uso

```sql
-- Verificar dependências
SELECT * 
FROM SYSIBM.SYSSEQUENCESDEP 
WHERE SEQSCHEMA = 'DBATEST' 
  AND SEQNAME = 'SEQ_CLIENTE_ID';

-- Identificar statements que usam a SEQUENCE
SELECT COLLID,
       NAME,
       SEQNO,
       SECTNO,
       TEXT
FROM SYSIBM.SYSPACKSTMT 
WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%';
```

### Análise de Performance

```sql
-- Verificar estatísticas de uso (DB2 13+)
SELECT SEQSCHEMA,
       SEQNAME,
       REQUESTS,
       CACHEHITS,
       CACHEMISSES,
       AVGCACHEHITTIME,
       AVGCACHEMISSTIME
FROM SYSIBM.SYSSEQUENCESTATS
WHERE SEQSCHEMA = 'DBATEST';
```

---

## 5. Como Saber se um Campo Está Associado a uma SEQUENCE

Como a SEQUENCE é independente, a associação não é explícita no catálogo. Você pode investigar através de múltiplas abordagens:

### A) Verificar Triggers que usam SEQUENCE

```sql
-- Pesquisar em triggers
SELECT SCHEMA,
       NAME,
       TBNAME,
       TBCREATOR,
       TEXT
FROM SYSIBM.SYSTRIGGERS 
WHERE TEXT LIKE '%NEXT VALUE FOR%'
  AND TEXT LIKE '%DBATEST.SEQ_CLIENTE_ID%';
```

### B) Examinar Stored Procedures

```sql
-- Pesquisar em stored procedures
SELECT SCHEMA,
       NAME,
       TEXT
FROM SYSIBM.SYSROUTINES 
WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%'
  AND ROUTINETYPE = 'P';
```

### C) Verificar Packages de Programas

```sql
-- Verificar programas COBOL/PL/I que usam a SEQUENCE
SELECT DISTINCT COLLID,
       NAME,
       CREATOR,
       TIMESTAMP
FROM SYSIBM.SYSPACKSTMT 
WHERE TEXT LIKE '%NEXT VALUE FOR%'
  AND TEXT LIKE '%SEQ_CLIENTE_ID%';
```

### D) Analisar Default Values

```sql
-- Verificar se há DEFAULT usando SEQUENCE
SELECT TBCREATOR,
       TBNAME,
       NAME,
       DEFAULTVALUE
FROM SYSIBM.SYSCOLUMNS
WHERE DEFAULTVALUE LIKE '%NEXT VALUE FOR%'
  AND DEFAULTVALUE LIKE '%SEQ_CLIENTE_ID%';
```

### E) Script de Auditoria Completa

```sql
-- Script para encontrar todos os usos de uma SEQUENCE
WITH SEQUENCE_USAGE AS (
    -- Triggers
    SELECT 'TRIGGER' AS TIPO,
           SCHEMA || '.' || NAME AS OBJETO,
           'TRIGGER em ' || TBCREATOR || '.' || TBNAME AS CONTEXTO
    FROM SYSIBM.SYSTRIGGERS 
    WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%'
    
    UNION ALL
    
    -- Stored Procedures
    SELECT 'PROCEDURE' AS TIPO,
           SCHEMA || '.' || NAME AS OBJETO,
           'STORED PROCEDURE' AS CONTEXTO
    FROM SYSIBM.SYSROUTINES 
    WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%'
    
    UNION ALL
    
    -- Programs
    SELECT 'PROGRAM' AS TIPO,
           COLLID || '.' || NAME AS OBJETO,
           'PROGRAM PACKAGE' AS CONTEXTO
    FROM SYSIBM.SYSPACKSTMT 
    WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%'
    
    UNION ALL
    
    -- Default Values
    SELECT 'DEFAULT' AS TIPO,
           TBCREATOR || '.' || TBNAME || '.' || NAME AS OBJETO,
           'DEFAULT VALUE' AS CONTEXTO
    FROM SYSIBM.SYSCOLUMNS
    WHERE DEFAULTVALUE LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%'
)
SELECT * FROM SEQUENCE_USAGE
ORDER BY TIPO, OBJETO;
```

---

## 6. Utilização em Programas COBOL, SQL Dinâmico e SP

### A) SQL Nativo (estático)

```sql
-- INSERT direto
INSERT INTO CLIENTE (ID, NOME, EMAIL) 
VALUES (NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID, 'SILVIA', 'silvia@teste.com');

-- INSERT com subquery
INSERT INTO CLIENTE (ID, NOME, EMAIL)
SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID,
       'JOÃO',
       'joao@teste.com'
FROM SYSIBM.SYSDUMMY1;

-- Múltiplos INSERTs
INSERT INTO CLIENTE (ID, NOME, EMAIL) 
VALUES (NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID, 'MARIA', 'maria@teste.com'),
       (NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID, 'PEDRO', 'pedro@teste.com');
```

### B) Programas COBOL (Embedded SQL)

```cobol
WORKING-STORAGE SECTION.
01 WS-CLIENTE.
   05 WS-ID                PIC 9(10).
   05 WS-NOME              PIC X(50).
   05 WS-EMAIL             PIC X(100).

01 SQLCA.
   COPY SQLCA.

PROCEDURE DIVISION.
MAIN-PROCESS.
    MOVE 'CLIENTE TESTE' TO WS-NOME
    MOVE 'teste@email.com' TO WS-EMAIL
    
    * Obter próximo ID da SEQUENCE
    EXEC SQL
        SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID
        INTO :WS-ID
        FROM SYSIBM.SYSDUMMY1
    END-EXEC
    
    IF SQLCODE = 0
        EXEC SQL
            INSERT INTO CLIENTE (ID, NOME, EMAIL)
            VALUES (:WS-ID, :WS-NOME, :WS-EMAIL)
        END-EXEC
        
        IF SQLCODE = 0
            EXEC SQL COMMIT END-EXEC
            DISPLAY 'CLIENTE INSERIDO COM ID: ' WS-ID
        ELSE
            EXEC SQL ROLLBACK END-EXEC
            DISPLAY 'ERRO NO INSERT: ' SQLCODE
        END-IF
    ELSE
        DISPLAY 'ERRO AO OBTER SEQUENCE: ' SQLCODE
    END-IF.
```

### C) SQL Dinâmico

```sql
-- Exemplo em PL/SQL ou procedimento
DECLARE
    stmt VARCHAR(500);
    cliente_id BIGINT;
BEGIN
    -- Obter ID da SEQUENCE
    GET DIAGNOSTICS cliente_id = ROW_COUNT;
    
    -- Construir statement dinâmico
    SET stmt = 'INSERT INTO CLIENTE (ID, NOME, EMAIL) VALUES (' 
            || 'NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID' || ', '
            || '''CLIENTE DINAMICO''' || ', '
            || '''dinamico@teste.com''' || ')';
    
    -- Executar
    EXECUTE IMMEDIATE stmt;
END;
```

### D) Stored Procedures

```sql
CREATE PROCEDURE DBATEST.SP_INSERIR_CLIENTE (
    IN P_NOME VARCHAR(50),
    IN P_EMAIL VARCHAR(100),
    OUT P_ID BIGINT,
    OUT P_SQLCODE INTEGER,
    OUT P_MENSAGEM VARCHAR(200)
)
LANGUAGE SQL
DYNAMIC RESULT SETS 0
BEGIN
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS P_SQLCODE = DB2_RETURNED_SQLCODE,
                       P_MENSAGEM = MESSAGE_TEXT;
        ROLLBACK;
    END;
    
    -- Obter próximo ID
    SET P_ID = NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID;
    
    -- Inserir cliente
    INSERT INTO CLIENTE (ID, NOME, EMAIL)
    VALUES (P_ID, P_NOME, P_EMAIL);
    
    -- Sucesso
    SET P_SQLCODE = 0;
    SET P_MENSAGEM = 'CLIENTE INSERIDO COM SUCESSO';
    
    COMMIT;
END;
```

### E) Uso com DEFAULT VALUE

```sql
-- Definir DEFAULT na tabela
ALTER TABLE CLIENTE 
ALTER COLUMN ID SET DEFAULT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID;

-- INSERT sem especificar ID
INSERT INTO CLIENTE (NOME, EMAIL) 
VALUES ('CLIENTE COM DEFAULT', 'default@teste.com');
```

---

## 7. Testes Práticos em Ambiente DEV

### 1. Criação e Teste Básico

```sql
-- Criar SEQUENCE de teste
CREATE SEQUENCE DBATEST.SEQ_TESTE_ID 
    START WITH 100 
    INCREMENT BY 1
    CACHE 10;

-- Verificar valor inicial
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID 
FROM SYSIBM.SYSDUMMY1;

-- Resultado: 100
```

### 2. Teste de Incremento

```sql
-- Obter múltiplos valores
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;

-- Resultados: 101, 102, 103
```

### 3. Teste de Concorrência

```sql
-- Simular múltiplas sessões simultâneas
-- Sessão 1:
BEGIN;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
-- Aguardar...
COMMIT;

-- Sessão 2 (executar simultaneamente):
BEGIN;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
-- Não há conflito, valores únicos
COMMIT;
```

### 4. Teste de RESTART

```sql
-- Reiniciar SEQUENCE
ALTER SEQUENCE DBATEST.SEQ_TESTE_ID RESTART WITH 1;

-- Verificar reinício
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
-- Resultado: 1
```

### 5. Teste com Diferentes Increments

```sql
-- Criar SEQUENCE com incremento diferente
CREATE SEQUENCE DBATEST.SEQ_TESTE_INC5 
    START WITH 1000 
    INCREMENT BY 5
    CACHE 20;

-- Testar incremento
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_INC5 FROM SYSIBM.SYSDUMMY1;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_INC5 FROM SYSIBM.SYSDUMMY1;
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_INC5 FROM SYSIBM.SYSDUMMY1;

-- Resultados: 1000, 1005, 1010
```

### 6. Simulação de Falha com CACHE

```sql
-- Criar SEQUENCE com CACHE para testar gaps
CREATE SEQUENCE DBATEST.SEQ_CACHE_TEST
    START WITH 1 
    INCREMENT BY 1 
    CACHE 10;

-- Obter alguns valores
SELECT NEXT VALUE FOR DBATEST.SEQ_CACHE_TEST FROM SYSIBM.SYSDUMMY1; -- 1
SELECT NEXT VALUE FOR DBATEST.SEQ_CACHE_TEST FROM SYSIBM.SYSDUMMY1; -- 2
SELECT NEXT VALUE FOR DBATEST.SEQ_CACHE_TEST FROM SYSIBM.SYSDUMMY1; -- 3

-- Simular reinício do DB2 (ou restart da aplicação)
-- O cache é perdido, próximo valor pode ser 11 (não 4)
```

### 7. Teste de Limites

```sql
-- Criar SEQUENCE com limites baixos
CREATE SEQUENCE DBATEST.SEQ_LIMITE_TEST
    START WITH 95
    INCREMENT BY 1
    MAXVALUE 100
    NO CYCLE;

-- Testar até o limite
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 95
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 96
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 97
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 98
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 99
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- 100
SELECT NEXT VALUE FOR DBATEST.SEQ_LIMITE_TEST FROM SYSIBM.SYSDUMMY1; -- ERRO!
```

### 8. Teste de Performance

```sql
-- Comparar performance com e sem CACHE
-- Sem CACHE
CREATE SEQUENCE DBATEST.SEQ_NO_CACHE
    START WITH 1
    NO CACHE;

-- Com CACHE
CREATE SEQUENCE DBATEST.SEQ_WITH_CACHE
    START WITH 1
    CACHE 100;

-- Teste de velocidade (executar múltiplas vezes e medir tempo)
-- Sem cache será mais lento devido ao I/O do catálogo
```

---

## 8. Cuidados com LOAD e RESTART

### A) Situações Críticas com LOAD

#### Problema: Dessincronia após LOAD REPLACE

```sql
-- Cenário problemático
-- 1. Tabela CLIENTE com ID usando SEQUENCE
-- 2. Último ID na tabela: 1000
-- 3. Próximo valor da SEQUENCE: 1001
-- 4. LOAD REPLACE com dados que têm ID até 2000
-- 5. Próximo NEXT VALUE FOR ainda será 1001 (conflito!)

-- Solução: Ajustar SEQUENCE após LOAD
ALTER SEQUENCE DBATEST.SEQ_CLIENTE_ID 
RESTART WITH (SELECT COALESCE(MAX(ID), 0) + 1 FROM CLIENTE);
```

#### Script Seguro para LOAD com SEQUENCE

```sql
-- Antes do LOAD
-- 1. Verificar valor atual da SEQUENCE
SELECT SEQCACHE, SEQCACHEHWM 
FROM SYSIBM.SYSSEQUENCES 
WHERE SCHEMA = 'DBATEST' 
  AND NAME = 'SEQ_CLIENTE_ID';

-- 2. Executar LOAD
LOAD DATA REPLACE INTO CLIENTE 
FROM '/path/to/data.txt' 
MODIFIED BY COLDEL| 
MESSAGES '/path/to/messages.txt';

-- 3. Verificar máximo valor após LOAD
SELECT MAX(ID) FROM CLIENTE;

-- 4. Ajustar SEQUENCE
ALTER SEQUENCE DBATEST.SEQ_CLIENTE_ID 
RESTART WITH (SELECT COALESCE(MAX(ID), 0) + 1 FROM CLIENTE);

-- 5. Verificar sincronização
SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID FROM SYSIBM.SYSDUMMY1;
```

#### Automação com Stored Procedure

```sql
CREATE PROCEDURE DBATEST.SP_ADJUST_SEQUENCE_AFTER_LOAD(
    IN P_SCHEMA VARCHAR(30),
    IN P_SEQUENCE_NAME VARCHAR(30),
    IN P_TABLE_SCHEMA VARCHAR(30),
    IN P_TABLE_NAME VARCHAR(30),
    IN P_COLUMN_NAME VARCHAR(30),
    OUT P_OLD_VALUE BIGINT,
    OUT P_NEW_VALUE BIGINT,
    OUT P_RESULT VARCHAR(200)
)
LANGUAGE SQL
BEGIN
    DECLARE v_max_value BIGINT DEFAULT 0;
    DECLARE v_sql VARCHAR(500);
    
    -- Obter valor atual da SEQUENCE
    SELECT COALESCE(SEQCACHE, 0) INTO P_OLD_VALUE
    FROM SYSIBM.SYSSEQUENCES
    WHERE SCHEMA = P_SCHEMA AND NAME = P_SEQUENCE_NAME;
    
    -- Obter máximo valor da tabela
    SET v_sql = 'SELECT COALESCE(MAX(' || P_COLUMN_NAME || '), 0) FROM ' 
                || P_TABLE_SCHEMA || '.' || P_TABLE_NAME;
    
    -- Executar query dinâmica (simplificado)
    SELECT MAX(ID) INTO v_max_value FROM CLIENTE; -- Adaptar para dinâmico
    
    -- Calcular novo valor
    SET P_NEW_VALUE = v_max_value + 1;
    
    -- Ajustar SEQUENCE se necessário
    IF P_NEW_VALUE > P_OLD_VALUE THEN
        SET v_sql = 'ALTER SEQUENCE ' || P_SCHEMA || '.' || P_SEQUENCE_NAME 
                   || ' RESTART WITH ' || CHAR(P_NEW_VALUE);
        
        EXECUTE IMMEDIATE v_sql;
        SET P_RESULT = 'SEQUENCE AJUSTADA DE ' || CHAR(P_OLD_VALUE) || ' PARA ' || CHAR(P_NEW_VALUE);
    ELSE
        SET P_RESULT = 'SEQUENCE JÁ ESTÁ SINCRONIZADA';
    END IF;
END;
```

### B) Cuidados em RESTART de Jobs

#### Problema: Jobs Não-Idempotentes

```cobol
* Exemplo problemático em COBOL
PROCEDURE DIVISION.
MAIN-PROCESS.
    * Obter ID da SEQUENCE
    EXEC SQL
        SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID
        INTO :WS-ID
    END-EXEC
    
    * Se job falhar aqui, o valor da SEQUENCE foi "perdido"
    * No RESTART, um novo valor será obtido
    
    EXEC SQL
        INSERT INTO CLIENTE (ID, NOME)
        VALUES (:WS-ID, :WS-NOME)
    END-EXEC.
```

#### Solução: Checkpoint Strategy

```cobol
WORKING-STORAGE SECTION.
01 WS-CHECKPOINT-FILE.
   05 WS-LAST-ID           PIC 9(10).
   05 WS-RESTART-FLAG      PIC X(1).

PROCEDURE DIVISION.
MAIN-PROCESS.
    * Verificar se é restart
    IF WS-RESTART-FLAG = 'Y'
        * Usar ID do checkpoint
        MOVE WS-LAST-ID TO WS-ID
    ELSE
        * Obter novo ID da SEQUENCE
        EXEC SQL
            SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID
            INTO :WS-ID
        END-EXEC
        
        * Salvar em checkpoint
        MOVE WS-ID TO WS-LAST-ID
        MOVE 'Y' TO WS-RESTART-FLAG
        * Gravar checkpoint file
    END-IF
    
    * Processar com ID garantido
    EXEC SQL
        INSERT INTO CLIENTE (ID, NOME)
        VALUES (:WS-ID, :WS-NOME)
    END-EXEC
    
    * Limpar checkpoint após sucesso
    MOVE 'N' TO WS-RESTART-FLAG.
```

#### Estratégia com Tabela de Controle

```sql
-- Tabela de controle para jobs
CREATE TABLE DBATEST.JOB_CONTROL (
    JOB_NAME VARCHAR(30) NOT NULL,
    SEQUENCE_NAME VARCHAR(30) NOT NULL,
    LAST_ID BIGINT NOT NULL,
    STATUS CHAR(1) NOT NULL, -- P=Processing, C=Complete
    TIMESTAMP TIMESTAMP NOT NULL,
    PRIMARY KEY (JOB_NAME, SEQUENCE_NAME)
);

-- Stored procedure para controle seguro
CREATE PROCEDURE DBATEST.SP_GET_SAFE_SEQUENCE_ID(
    IN P_JOB_NAME VARCHAR(30),
    IN P_SEQUENCE_NAME VARCHAR(30),
    OUT P_ID BIGINT,
    OUT P_IS_RESTART CHAR(1)
)
LANGUAGE SQL
BEGIN
    -- Verificar se job está em restart
    SELECT LAST_ID, 'Y'
    INTO P_ID, P_IS_RESTART
    FROM DBATEST.JOB_CONTROL
    WHERE JOB_NAME = P_JOB_NAME
      AND SEQUENCE_NAME = P_SEQUENCE_NAME
      AND STATUS = 'P';
    
    -- Se não encontrou, é primeira execução
    IF P_ID IS NULL THEN
        -- Obter novo ID da SEQUENCE
        EXECUTE IMMEDIATE 'SELECT NEXT VALUE FOR ' || P_SEQUENCE_NAME || ' FROM SYSIBM.SYSDUMMY1'
        INTO P_ID;
        
        -- Registrar controle
        INSERT INTO DBATEST.JOB_CONTROL 
        VALUES (P_JOB_NAME, P_SEQUENCE_NAME, P_ID, 'P', CURRENT_TIMESTAMP);
        
        SET P_IS_RESTART = 'N';
    END IF;
END;
```

---


### 9. Boas Práticas e Estratégias de Produção

- **Use sempre um nome padronizado para SEQUENCEs**, como `SEQ_<TABELA>_<COLUNA>` para facilitar rastreabilidade.
- **Evite o uso de `NO CYCLE` sem planejamento.** Isso pode causar falha silenciosa quando o valor máximo for atingido.
- **Avalie o uso de CACHE com cautela.** Pode melhorar performance, mas em caso de falha, valores são "perdidos" (não gerados novamente).
- **Documente o SEQUENCE associado à tabela e coluna.** Isso é importante pois SEQUENCE é um objeto separado da tabela.

#### Exemplo de criação com padrão recomendado:

```sql
CREATE SEQUENCE SEQ_CLIENTE_ID
  START WITH 1
  INCREMENT BY 1
  NO CYCLE
  CACHE 20;
```

#### Estratégia para gerar valor e inserir:

```sql
INSERT INTO CLIENTES (ID, NOME)
VALUES (NEXT VALUE FOR SEQ_CLIENTE_ID, 'SILVIA');
```

---

### 10. Gerenciamento de Memória e Cache

#### Cache no SEQUENCE: o que é?

O **CACHE** permite ao DB2 manter valores de SEQUENCE em memória, evitando I/Os para cada chamada.

```sql
CREATE SEQUENCE SEQ_PEDIDO_ID
  START WITH 1000
  INCREMENT BY 1
  CACHE 50;
```

#### Benefícios:
- Aumenta a performance em operações intensivas.
- Reduz contention entre tarefas.

#### Cuidados:
- Se o DB2 for reiniciado ou a instância falhar, os valores no cache são **perdidos**, mas isso **não compromete a unicidade**.
- Para ambientes críticos de auditoria ou com exigência de valores sequenciais exatos, prefira `NOCACHE`.

#### Simulação de perda de valores:

1. Crie com CACHE:
```sql
CREATE SEQUENCE SEQ_EXEMPLO CACHE 10;
```

2. Gere valores até o 5º:
```sql
SELECT NEXT VALUE FOR SEQ_EXEMPLO FROM SYSIBM.SYSDUMMY1;
```

3. Simule um *crash*. Na volta, o valor continua do 11 (caso os 10 estivessem em cache e DB2 não persistiu os intermediários).

---

### 11. Troubleshooting e Resolução de Problemas

#### Verificar o status de uma SEQUENCE:

```sql
SELECT *
FROM SYSIBM.SYSSEQUENCES
WHERE NAME = 'SEQ_CLIENTE_ID';
```

#### Problemas comuns:
- **Erro -803 (duplicate key)** ao usar SEQUENCE com `INSERT` automático → pode indicar falha de sincronismo SEQUENCE vs dados existentes.
- **SEQUENCE pulando valores:** normal com CACHE ou rollback de transações após `NEXT VALUE`.
- **Valor final atingido (`MAXVALUE`)** → Verificar na `SYSSEQUENCES` os limites definidos.

#### Correção prática: reiniciar SEQUENCE

```sql
ALTER SEQUENCE SEQ_CLIENTE_ID RESTART WITH 10000;
```

---

### 12. Migração de IDENTITY para SEQUENCE

#### Por que migrar?
- Maior controle.
- Possibilidade de reutilizar o mesmo SEQUENCE entre tabelas.
- Compatível com replicações, particionamentos e scripts multiambiente.

#### Comparação rápida:

| Recurso                   | IDENTITY     | SEQUENCE       |
|---------------------------|--------------|----------------|
| Associação à Tabela       | Implícita     | Independente   |
| Reutilização              | Não           | Sim            |
| Cache configurável        | Sim           | Sim            |
| Alteração posterior       | Limitada      | Ampla          |
| Utilização em múltiplas tabelas | Não     | Sim            |

#### Exemplo de migração:

1. Tabela com IDENTITY:
```sql
CREATE TABLE CLIENTES (
  ID INTEGER GENERATED ALWAYS AS IDENTITY,
  NOME VARCHAR(100)
);
```

2. Novo modelo com SEQUENCE:
```sql
CREATE SEQUENCE SEQ_CLIENTE_ID START WITH 1 INCREMENT BY 1;

CREATE TABLE CLIENTES (
  ID INTEGER NOT NULL,
  NOME VARCHAR(100)
);
```

3. Uso:
```sql
INSERT INTO CLIENTES (ID, NOME)
VALUES (NEXT VALUE FOR SEQ_CLIENTE_ID, 'MARIA');
```

---

### 13. Referências Oficiais IBM

- 📘 [IBM DB2 for z/OS - CREATE SEQUENCE (SQL)](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-sequence)
- 📘 [IBM DB2 for z/OS - SYSIBM.SYSSEQUENCES catalog](https://www.ibm.com/docs/en/db2-for-zos/13?topic=catalogs-syssequences)
- 📘 [IBM Redbooks - DB2 12 for z/OS Technical Overview](https://www.redbooks.ibm.com/redbooks/pdfs/sg248383.pdf)
- 📘 [IBM Documentation Navigator](https://www.ibm.com/docs/en)

---

