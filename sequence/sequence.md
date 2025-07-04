# Manual Técnico: SEQUENCE no DB2 for z/OS

## Índice

- [1. Conceito e Objetivo da SEQUENCE](#1-conceito-e-objetivo-da-sequence)
- [2. Sintaxe e Criação de uma SEQUENCE](#2-sintaxe-e-criação-de-uma-sequence)
- [3. Propriedades Técnicas da SEQUENCE](#3-propriedades-técnicas-da-sequence)
- [4. Como Consultar e Monitorar SEQUENCES](#4-como-consultar-e-monitorar-sequences)
- [5. Como Saber se um Campo Está Associado a uma SEQUENCE](#5-como-saber-se-um-campo-está-associado-a-uma-sequence)
- [6. Utilização em Programas COBOL, SQL Dinâmico e SP](#6-utilização-em-programas-cobol-sql-dinâmico-e-sp)
- [7. Testes Práticos em Ambiente DEV](#7-testes-práticos-em-ambiente-dev)
- [8. Cuidados com LOAD e RESTART](#8-cuidados-com-load-e-restart)
- [9. Boas Práticas e Estratégias de Produção](#9-boas-práticas-e-estratégias-de-produção)
- [10. Referências Oficiais IBM](#10-referências-oficiais-ibm)

---

## 1. Conceito e Objetivo da SEQUENCE

Uma **SEQUENCE** é um objeto do DB2 for z/OS criado para gerar números sequenciais únicos, típicos para uso como **identificadores primários** (PK) em tabelas. Substitui com vantagens o `IDENTITY COLUMN`, oferecendo **maior controle, reuso entre tabelas, persistência e escalabilidade**.

### Por que usar SEQUENCE?
- Evita concorrência e gargalos com `MAX(COL)+1`.
- É reutilizável por várias tabelas.
- Permite controle detalhado do intervalo e incremento.
- Suporte a ambientes de **alta concorrência** e **grandes volumes**.

---

## 2. Sintaxe e Criação de uma SEQUENCE

```sql
CREATE SEQUENCE DBATEST.SEQ_CLIENTE_ID
    AS BIGINT
    START WITH 1
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 9999999999
    CYCLE NO
    CACHE 100
    ORDER;
```

### Explicação dos Parâmetros:
- `START WITH`: valor inicial.
- `INCREMENT BY`: incremento entre valores.
- `MINVALUE` / `MAXVALUE`: limite inferior/superior.
- `CYCLE`: se sim, reinicia ao atingir `MAXVALUE`.
- `CACHE`: pré-alocação em buffer para performance.
- `ORDER`: garante ordem em ambientes paralelos (custo maior).

---

## 3. Propriedades Técnicas da SEQUENCE

| Propriedade         | Descrição |
|---------------------|-----------|
| Persistência        | Objeto persistente no catálogo. |
| Independência       | Não ligada diretamente a nenhuma tabela. |
| Concorrência        | Totalmente segura para múltiplas sessões. |
| Bufferização (CACHE)| Melhora performance, mas pode causar gaps em caso de falha. |
| Semaforização       | Internamente gerenciada, segura para uso simultâneo. |

---

## 4. Como Consultar e Monitorar SEQUENCES

Você pode usar as views do catálogo:

```sql
SELECT * 
FROM SYSIBM.SYSSEQUENCES 
WHERE NAME = 'SEQ_CLIENTE_ID';
```

Outras tabelas úteis:
- `SYSIBM.SYSSEQUENCESDEP`: dependências indiretas.
- `SYSIBM.SYSSTMT`: verificar programas que usam `NEXT VALUE FOR`.

---

## 5. Como Saber se um Campo Está Associado a uma SEQUENCE

Como a SEQUENCE é independente, a associação não é explícita no catálogo. Você pode:

### A) Verificar Triggers ou SPs
```sql
SELECT TEXT 
FROM SYSIBM.SYSROUTINES 
WHERE TEXT LIKE '%NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID%';
```

### B) Examinar fontes de programas COBOL
- Pesquisar `NEXT VALUE FOR` em fontes, triggers ou instruções SQL estáticas.

### C) Verificar Bind Packages:
```sql
SELECT * 
FROM SYSIBM.SYSPACKSTMT 
WHERE TEXT LIKE '%NEXT VALUE FOR%';
```

---

## 6. Utilização em Programas COBOL, SQL Dinâmico e SP

### A) Em SQL nativo:
```sql
INSERT INTO CLIENTE (ID, NOME) 
VALUES (NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID, 'SILVIA');
```

### B) Em programas COBOL (Embedded SQL):
```cobol
EXEC SQL
    SELECT NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID
    INTO :WS-ID
END-EXEC.
```

### C) Em Stored Procedure:
```sql
SET V_ID = NEXT VALUE FOR DBATEST.SEQ_CLIENTE_ID;
```

---

## 7. Testes Práticos em Ambiente DEV

### 1. Criar SEQUENCE
```sql
CREATE SEQUENCE DBATEST.SEQ_TESTE_ID START WITH 100;
```

### 2. Verificar valor inicial:
```sql
SELECT NEXT VALUE FOR DBATEST.SEQ_TESTE_ID FROM SYSIBM.SYSDUMMY1;
```

### 3. Repetir várias vezes, observar incremento:
```sql
VALUES (NEXT VALUE FOR DBATEST.SEQ_TESTE_ID);
```

### 4. Reiniciar SEQUENCE:
```sql
ALTER SEQUENCE DBATEST.SEQ_TESTE_ID RESTART WITH 1;
```

### 5. Simular falha com CACHE e comparar gaps:
```sql
-- Criar com CACHE:
CREATE SEQUENCE DBATEST.SEQ_CACHE_EX
    START WITH 1 INCREMENT BY 1 CACHE 10;

-- Simular falha de instância e verificar gaps nos valores.
```

---

## 8. Cuidados com LOAD e RESTART

### A) Durante LOAD com REPLACE
- Se houver chave gerada via SEQUENCE, o `LOAD` precisa considerar o maior valor existente após a operação.
- Caso contrário, o próximo valor da SEQUENCE poderá gerar conflito com PK duplicada.

### Solução:
```sql
ALTER SEQUENCE DBATEST.SEQ_CLIENTE_ID 
RESTART WITH (SELECT MAX(ID) + 1 FROM CLIENTE);
```

### B) Cuidados em RESTART de JOBs
- Jobs com `NEXT VALUE FOR` não são idempotentes.
- Um `RESTART` pode gerar novo valor mesmo sem novo `INSERT`.

### Boas práticas:
- Capturar valor de SEQUENCE em variável antes de qualquer erro potencial.
- Controlar via checkpoint ou staging temporário.

---

## 9. Boas Práticas e Estratégias de Produção

- **Use CACHE sempre que possível** (exceto se gaps forem inaceitáveis).
- **Evite ORDER** em ambientes OLTP, a não ser que a ordem exata seja vital.
- **Crie padrão de nomenclatura**, por exemplo: `SEQ_TABELA_CAMPO`.
- **Audite o uso em programas** — SEQUENCE pode ser usada por qualquer sessão sem restrição.
- **Monitore crescimento** — verifique se `MAXVALUE` está próximo do limite.
- **Não use em UPDATEs ou DELETES** — é desperdício de número sequencial.

---

## 10. Referências Oficiais IBM

- 📘 [IBM DB2 13 for z/OS SQL Reference - SEQUENCE](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-sequence)
- 📘 [IBM Knowledge Center - SYSIBM.SYSSEQUENCES](https://www.ibm.com/docs/en/db2-for-zos/13?topic=catalog-syssequences)
- 📘 [DB2 for z/OS Programming Guide](https://www.ibm.com/docs/en/db2-for-zos/13?topic=guide-using-sequence-objects)

---

