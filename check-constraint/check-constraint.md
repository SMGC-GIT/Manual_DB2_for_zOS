# CHECK CONSTRAINT no DB2 for z/OS

## Índice

- [1. Conceito](#1-conceito)
- [2. Por que usar CHECK CONSTRAINT?](#2-por-que-usar-check-constraint)
- [3. Estrutura da Sintaxe](#3-estrutura-da-sintaxe)
- [4. Regras e Limitações Importantes](#4-regras-e-limitações-importantes)
- [5. Como verificar se um campo tem um CHECK CONSTRAINT?](#5-como-verificar-se-um-campo-tem-um-check-constraint)
- [6. Exemplos Didáticos](#6-exemplos-didáticos)
- [7. Testando um CHECK](#7-testando-um-check)
- [8. Considerações para Ambientes de Produção](#8-considerações-para-ambientes-de-produção)
- [9. Performance](#9-performance)
- [10. Script Gerador de CHECK Constraints](#10-script-gerador-de-check-constraints)
- [11. Diferença entre CHECK, TRIGGER e FOREIGN KEY](#11-diferença-entre-check-trigger-e-foreign-key)
- [12. Fontes Oficiais IBM](#12-fontes-oficiais-ibm)
- [13. Checklist de Implementação](#13-checklist-de-implementação)
- [14. Como visualizar CHECK CONSTRAINT no PowerDesigner](#14-como-visualizar-check-constraint-no-powerdesigner)

---

## 1. Conceito

`CHECK CONSTRAINT` é uma restrição declarativa usada para garantir que os valores inseridos ou atualizados em uma tabela satisfaçam uma determinada condição lógica definida pelo usuário.

- Atua no **nível da linha**.
- Garante que os dados estejam **dentro dos limites de negócio** aceitos.
- Evita inconsistências diretamente no banco de dados.

---

## 2. Por que usar CHECK CONSTRAINT?

- **Integridade de Dados:** Impede gravações inválidas.
- **Centralização de Regras:** Evita replicação de lógica nas aplicações.
- **Redução de Erros:** Protege contra falhas humanas ou de sistema.
- **Governança:** Permite rastreabilidade e documentação clara das regras de dados.

---

## 3. Estrutura da Sintaxe

### Durante a criação da tabela:

```sql
CREATE TABLE CLIENTE (
  CPF CHAR(11) NOT NULL,
  IDADE SMALLINT NOT NULL,
  RENDA DECIMAL(10,2),
  CONSTRAINT CK_IDADE_MINIMA CHECK (IDADE >= 18)
);
```

### Após a criação da tabela:

```sql
ALTER TABLE CLIENTE
  ADD CONSTRAINT CK_RENDA_POSITIVA CHECK (RENDA >= 0);
```

---

## 4. Regras e Limitações Importantes

- Apenas condições que envolvam **colunas da mesma linha**.
- Não permite:
  - Subqueries
  - JOINs
  - UDFs ou Stored Procedures
- Não aceita funções como `CURRENT DATE`, `RAND()`, etc.
- Para lógicas mais complexas, considere o uso de `TRIGGER`.

---

## 5. Como verificar se um campo tem um CHECK CONSTRAINT?

### Catálogo DB2 for z/OS:

#### a) `SYSIBM.SYSCHECKS`
Contém as definições dos CHECK constraints.

#### b) `SYSIBM.SYSCHECKDEP`
Relaciona as colunas envolvidas nos CHECKs.

### Exemplo:

```sql
-- Lista os CHECKS de uma tabela
SELECT NAME, TBNAME, TEXT
FROM SYSIBM.SYSCHECKS
WHERE TBNAME = 'CLIENTE';
```

#### Resultado esperado:

| NAME              | TBNAME   | TEXT                   |
|-------------------|----------|------------------------|
| CK_IDADE_MINIMA   | CLIENTE  | (IDADE >= 18)          |
| CK_RENDA_POSITIVA | CLIENTE  | (RENDA >= 0)           |

---

```sql
-- Lista colunas associadas a cada CHECK
SELECT CHKNAME, TBNAME, NAME AS COLUNA
FROM SYSIBM.SYSCHECKDEP
WHERE TBNAME = 'CLIENTE';
```

#### Resultado esperado:

| CHKNAME           | TBNAME   | COLUNA         |
|-------------------|----------|----------------|
| CK_IDADE_MINIMA   | CLIENTE  | IDADE          |
| CK_RENDA_POSITIVA | CLIENTE  | RENDA          |

---

## 6. Exemplos Didáticos

### Regra de idade mínima:

```sql
CHECK (IDADE >= 18)
```

### Conjunto de valores válidos:

```sql
CHECK (TIPO_CLIENTE IN ('PF', 'PJ'))
```

### Validação condicional entre colunas:

```sql
CHECK (
  (TIPO_CONTA = 'CORRENTE' AND LIMITE_CREDITO IS NOT NULL)
  OR
  (TIPO_CONTA = 'POUPANCA' AND LIMITE_CREDITO IS NULL)
)
```

---

## 7. Testando um CHECK

```sql
-- OK
INSERT INTO CLIENTE (CPF, IDADE, RENDA)
VALUES ('12345678901', 25, 5000.00);
```

```sql
-- Violação da constraint
INSERT INTO CLIENTE (CPF, IDADE, RENDA)
VALUES ('12345678901', 16, 5000.00);
```

#### Resultado esperado:

| Mensagem técnica                                     |
|-----------------------------------------------------|
| SQLCODE = -543                                      |
| SQLSTATE = 23513                                    |
| Motivo: Violation of CHECK constraint 'CK_IDADE_MINIMA' |

---

## 8. Considerações para Ambientes de Produção

- Nomeie constraints com padrão empresarial: `CK_<TABELA>_<REGRA>`
- Documente cada CHECK no modelo lógico/físico.
- Tenha plano de rollback antes de alterar/remover constraints.
- Realize testes com dados reais.
- Evite regras ambíguas ou difíceis de manter.

---

## 9. Performance

- Avaliação feita em tempo de execução de DML (INSERT/UPDATE).
- Impacto de performance é baixo, se usados corretamente.
- Evite criar CHECKs com expressões complexas ou múltiplas colunas desnecessariamente.

---

## 10. Script Gerador de CHECK Constraints

```sql
SELECT C.TBNAME, C.NAME AS CHECK_NAME, C.TEXT AS DEFINITION
FROM SYSIBM.SYSCHECKS C
WHERE C.TYPE = 'C'
ORDER BY C.TBNAME, C.NAME;
```

#### Resultado esperado (exemplo):

| TBNAME   | CHECK_NAME         | DEFINITION          |
|----------|--------------------|---------------------|
| CLIENTE  | CK_IDADE_MINIMA    | (IDADE >= 18)       |
| CLIENTE  | CK_RENDA_POSITIVA  | (RENDA >= 0)        |

---

## 11. Diferença entre CHECK, TRIGGER e FOREIGN KEY

| Tipo        | Função Principal                      | Multitabela? | Quando usar                        |
|-------------|----------------------------------------|--------------|------------------------------------|
| CHECK       | Regras simples por linha               | ❌ Não       | Validação direta de valores        |
| TRIGGER     | Lógica complexa ou procedural          | ✅ Sim       | Validação com lógica condicional   |
| FOREIGN KEY | Integridade entre tabelas              | ✅ Sim       | Manter integridade referencial     |

---

## 12. Fontes Oficiais IBM

- [IBM DB2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/12?topic=reference-sql-statements)
- [IBM Knowledge Center - SYSCHECKS](https://www.ibm.com/docs/en/db2-for-zos/12?topic=catalog-syschecks)
- [IBM Redbooks – DB2 for z/OS: The Database Administrator’s Guide](https://www.redbooks.ibm.com/)

---

## 13. Checklist de Implementação

| Item                                                    | Status |
|----------------------------------------------------------|--------|
| A constraint representa claramente a regra de negócio?   | ✅     |
| Nome segue padrão corporativo e documentado?             | ✅     |
| A regra está registrada no modelo lógico e físico?       | ✅     |
| As aplicações estão cientes da regra?                    | ✅     |
| Testes foram realizados com dados válidos e inválidos?   | ✅     |

---

## 14. Como visualizar CHECK CONSTRAINT no PowerDesigner

### No Modelo Lógico (LDM):

1. Abra o modelo LDM no PowerDesigner.
2. Clique com o botão direito sobre a entidade desejada → *Properties*.
3. Acesse a aba **"Rules"** ou **"Constraints"**.
4. Adicione uma **Check Rule** com a condição desejada (ex: `IDADE >= 18`).

### No Modelo Físico (PDM):

1. No PDM, selecione a tabela → *Properties*.
2. Vá até a aba **"Check Constraints"**.
3. Clique em **Add** e preencha:
   - **Name**: `CK_CLIENTE_IDADE`
   - **Constraint**: `IDADE >= 18`
   - Marque a opção **Generate**.

### Observações:

- Certifique-se de que a opção **"Generate CHECK constraints"** esteja habilitada nas configurações do modelo.
- As constraints também podem ser visualizadas em scripts gerados via *Database Generation* ou *DDL Preview*.
- Use **Extended Attributes** para padronizar nomeação de regras entre diferentes tabelas e domínios.

---
