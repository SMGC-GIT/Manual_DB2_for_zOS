# CHECK CONSTRAINT e Constraints no DB2 for z/OS

---

## Índice clicável

- [1. Introdução e Contexto](#1-introdução-e-contexto)  
- [2. Tipos de Constraints no DB2 for z/OS](#2-tipos-de-constraints-no-db2-for-zos)  
- [3. CHECK CONSTRAINT: Definição, Importância e Detalhes Técnicos](#3-check-constraint-definição-importância-e-detalhes-técnicos)  
- [4. Sintaxe e Exemplos de Criação de CHECK CONSTRAINT](#4-sintaxe-e-exemplos-de-criação-de-check-constraint)  
- [5. Regras, Restrições e Boas Práticas](#5-regras-restrições-e-boas-práticas)  
- [6. Como Consultar Constraints no Catálogo DB2](#6-como-consultar-constraints-no-catalógo-db2)  
- [7. Diferenças Técnicas entre CHECK, TRIGGER e FOREIGN KEY](#7-diferenças-técnicas-entre-check-trigger-e-foreign-key)  
- [8. Impacto na Performance](#8-impacto-na-performance)  
- [9. Visualização e Gestão no PowerDesigner](#9-visualização-e-gestão-no-powerdesigner)  
- [10. Exemplos Práticos Completo com Resultados](#10-exemplos-práticos-completo-com-resultados)  
  - [10.1 CHECK CONSTRAINT](#101-check-constraint)  
  - [10.2 PRIMARY KEY](#102-primary-key)  
  - [10.3 FOREIGN KEY](#103-foreign-key)  
  - [10.4 UNIQUE](#104-unique)  
  - [10.5 NOT NULL](#105-not-null)  
- [11. Checklist para Implementação de Constraints](#11-checklist-para-implementação-de-constraints)  
- [12. Fontes Oficiais e Referências IBM](#12-fontes-oficiais-e-referências-ibm)  

---

## 1. Introdução e Contexto

No ambiente de banco de dados DB2 for z/OS, a integridade dos dados é um requisito fundamental para garantir confiabilidade, consistência e segurança das informações armazenadas. As **constraints** são as principais ferramentas declarativas para assegurar que os dados atendam às regras de negócio estabelecidas, funcionando diretamente no banco, sem depender das aplicações.

Entre essas constraints, a **CHECK CONSTRAINT** desempenha papel vital ao impor condições específicas e personalizadas sobre os valores das colunas, prevenindo inserções ou atualizações que violem as regras estabelecidas.

Este manual visa fornecer um conteúdo técnico avançado, didático e confiável, contemplando desde conceitos básicos, sintaxe, consulta no catálogo, impactos técnicos, até exemplos detalhados e resultados práticos.

---

## 2. Tipos de Constraints no DB2 for z/OS

No DB2 for z/OS, temos os seguintes tipos de constraints:

| Constraint     | Função Principal                               | Observações                      |
|----------------|-----------------------------------------------|---------------------------------|
| PRIMARY KEY    | Identificação única e obrigatória da linha    | Implica UNIQUE + NOT NULL        |
| FOREIGN KEY    | Garantia de integridade referencial entre tabelas | Restringe valores conforme tabela pai |
| UNIQUE         | Garante valores únicos em uma ou mais colunas | Permite NULL, exceto se NOT NULL |
| NOT NULL       | Impede valores nulos na coluna                 | Essencial para dados obrigatórios |
| CHECK          | Impõe condições lógicas definidas pelo usuário | Restrito a colunas da mesma linha |

---

## 3. CHECK CONSTRAINT: Definição, Importância e Detalhes Técnicos

### Definição

Uma **CHECK CONSTRAINT** é uma restrição declarativa que define uma expressão lógica que deve ser satisfeita para que uma linha seja inserida ou atualizada no banco. Caso contrário, a operação é rejeitada com erro.

### Por que usar?

- Centraliza regras de negócio diretamente no banco.
- Garante integridade de domínio (exemplo: idade mínima, valores permitidos).
- Evita duplicação e inconsistência ao delegar validação somente para aplicações.
- Facilita auditoria e manutenção das regras.
- Melhora a qualidade e segurança dos dados.

### Detalhes Técnicos Importantes

- Avaliada em cada operação DML (INSERT, UPDATE).
- Expressão restrita a colunas da mesma linha (não aceita subqueries, joins, funções não determinísticas).
- Limitações: não permite funções como CURRENT DATE, RAND(), etc.
- Para regras mais complexas, use triggers.

---

## 4. Sintaxe e Exemplos de Criação de CHECK CONSTRAINT

### Criação na criação da tabela:

```sql
CREATE TABLE CLIENTE (
  ID INT NOT NULL,
  IDADE SMALLINT NOT NULL,
  RENDA DECIMAL(10,2),
  CONSTRAINT CK_IDADE_MINIMA CHECK (IDADE >= 18),
  CONSTRAINT CK_RENDA_VALIDA CHECK (RENDA >= 0)
);
```

### Adicionando `CHECK` em tabela já existente:

```sql
ALTER TABLE CLIENTE
  ADD CONSTRAINT CK_TIPO_CLIENTE CHECK (TIPO_CLIENTE IN ('PF', 'PJ'));
```

### Observações:

- Use nomes descritivos para facilitar manutenção (`CK_<TABELA>_<DESCRICAO>`).
- Múltiplas constraints podem ser aplicadas à mesma tabela.

---

## 5. Regras, Restrições e Boas Práticas

- **Restrito à linha atual:** Não pode usar dados de outras linhas ou tabelas.
- **Sem subqueries ou joins:** Apenas colunas da tabela.
- **Evitar funções não determinísticas:** Ex: CURRENT DATE não pode ser usado em CHECK.
- **Sintaxe simples:** Priorize legibilidade e simplicidade para facilitar entendimento e manutenção.
- **Nomenclatura padronizada:** Facilita scripts, documentação e troubleshooting.
- **Testes rigorosos:** Valide dados válidos e inválidos em ambiente de desenvolvimento.
- **Documentação:** Registre a regra no modelo lógico e físico, e em documentação oficial.

---

## 6. Como Consultar Constraints no Catálogo DB2

Para consultar constraints no DB2, utilize as tabelas do catálogo abaixo:

| Tabela               | Uso principal                               |
|----------------------|---------------------------------------------|
| SYSIBM.SYSCHECKS     | Definições e textos das CHECK CONSTRAINTS  |
| SYSIBM.SYSCHECKDEP   | Colunas envolvidas nos CHECKs               |
| SYSIBM.SYSCST        | Todas as constraints da tabela              |
| SYSIBM.SYSCSTCOL     | Colunas das constraints                      |
| SYSIBM.SYSTABLES     | Informações sobre tabelas                    |
| SYSIBM.SYSCOLUMNS    | Informações sobre colunas                    |

### Consultas típicas

```sql
-- Listar CHECK constraints de uma tabela
SELECT NAME AS CHECK_NAME, TBNAME, TEXT AS CONDITION
FROM SYSIBM.SYSCHECKS
WHERE TBNAME = 'CLIENTE';

-- Colunas associadas a cada CHECK constraint
SELECT CHKNAME, TBNAME, NAME AS COLUMN_NAME
FROM SYSIBM.SYSCHECKDEP
WHERE TBNAME = 'CLIENTE';
```

---

## 7. Diferenças Técnicas entre CHECK, TRIGGER e FOREIGN KEY

| Constraint  | Pode acessar outras tabelas? | Quando usar                          | Complexidade | Tipos de validação típicas                |
|-------------|------------------------------|------------------------------------|--------------|-------------------------------------------|
| CHECK       | Não                          | Validação simples na mesma linha   | Baixa        | Condições lógicas entre colunas da linha  |
| TRIGGER     | Sim                          | Regras complexas e ações automáticas| Alta         | Verificação cross-tabela, lógica complexa |
| FOREIGN KEY | Sim                          | Integridade referencial entre tabelas | Média     | Consistência entre tabelas relacionadas   |

---

## 8. Impacto na Performance

- `CHECK` é avaliado a cada `INSERT` ou `UPDATE`.
- Regras simples têm impacto mínimo.
- Regras complexas ou múltiplas `CHECK` podem afetar a performance.
- Evite sobrecarregar a tabela com várias constraints redundantes.
- Testar em ambiente com carga representativa.
- Otimize expressões para minimizar custos de avaliação.

---

## 9. Visualização e Gestão no PowerDesigner

### No Modelo Lógico (LDM)

- Clique na entidade → propriedades → aba "Rules" ou "Constraints".
- Adicione regras do tipo **Check Rule** com expressão da constraint.
- Use Extended Attributes para padronizar nomeações.

### No Modelo Físico (PDM)

- Selecione tabela → propriedades → aba "Check Constraints".
- Adicione nova constraint com nome e expressão.
- Marque a opção para geração automática via DDL.
- Sincronize modelo e banco para garantir alinhamento.

---

## 10. Exemplos Práticos Completo com Resultados

### 10.1 CHECK CONSTRAINT

```sql
CREATE TABLE CLIENTE (
  ID INT,
  IDADE SMALLINT,
  CONSTRAINT CK_IDADE CHECK (IDADE >= 18)
);
```

| Operação                                      | Resultado Esperado                                     |
|----------------------------------------------|-------------------------------------------------------|
| `INSERT INTO CLIENTE VALUES (1, 25);`        | Sucesso                                              |
| `INSERT INTO CLIENTE VALUES (2, 16);`        | Erro SQLCODE -545: violação do CHECK `CK_IDADE`      |

---

### 10.2 PRIMARY KEY

```sql
CREATE TABLE DEPARTAMENTO (
  COD_DEP INT NOT NULL,
  NOME VARCHAR(40),
  CONSTRAINT PK_DEP PRIMARY KEY (COD_DEP)
);
```

| Operação                                        | Resultado Esperado                                |
|------------------------------------------------|-------------------------------------------------|
| `INSERT INTO DEPARTAMENTO VALUES (1, 'TI');`   | Sucesso                                         |
| `INSERT INTO DEPARTAMENTO VALUES (1, 'RH');`   | Erro SQLCODE -803: violação da chave primária  |

---

### 10.3 FOREIGN KEY

```sql
CREATE TABLE FUNCIONARIO (
  MATRICULA INT NOT NULL,
  NOME VARCHAR(40),
  COD_DEP INT,
  CONSTRAINT FK_FUNC_DEP FOREIGN KEY (COD_DEP) 
    REFERENCES DEPARTAMENTO(COD_DEP)
);
```

| Operação                                            | Resultado Esperado                               |
|----------------------------------------------------|-------------------------------------------------|
| `INSERT INTO FUNCIONARIO VALUES (101, 'Ana', 1);` | Sucesso                                        |
| `INSERT INTO FUNCIONARIO VALUES (102, 'Bruno', 99);` | Erro SQLCODE -530: violação da FK `FK_FUNC_DEP`|

---

### 10.4 UNIQUE

```sql
CREATE TABLE USUARIO (
  ID INT,
  EMAIL VARCHAR(100),
  CONSTRAINT UNQ_EMAIL UNIQUE (EMAIL)
);
```

| Operação                                        | Resultado Esperado                               |
|------------------------------------------------|-------------------------------------------------|
| `INSERT INTO USUARIO VALUES (1, 'ana@abc.com');`| Sucesso                                        |
| `INSERT INTO USUARIO VALUES (2, 'ana@abc.com');`| Erro SQLCODE -803: violação UNIQUE              |

---

### 10.5 NOT NULL

```sql
CREATE TABLE PRODUTO (
  ID INT,
  NOME VARCHAR(30) NOT NULL
);
```

| Operação                                      | Resultado Esperado                             |
|----------------------------------------------|-----------------------------------------------|
| `INSERT INTO PRODUTO VALUES (1, 'Copo');`    | Sucesso                                      |
| `INSERT INTO PRODUTO VALUES (2, NULL);`      | Erro SQLCODE -407: inserção de NULL não permitida |

---

## 11. Checklist para Implementação de Constraints

- [ ] Nomear constraints com padrão claro e consistente (ex: `CK_CLIENTE_IDADE`)  
- [ ] Documentar regras nos modelos lógico e físico  
- [ ] Realizar testes com dados válidos e inválidos  
- [ ] Validar impacto de performance em ambiente representativo  
- [ ] Verificar alinhamento com equipe de desenvolvimento e aplicação  
- [ ] Planejar rollback e controle de versões para alterações  
- [ ] Registrar constraints em documentação formal do projeto  

---

## 12. Fontes Oficiais e Referências IBM

- [IBM DB2 for z/OS SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/12?topic=reference-sql-statements)  
- [IBM Knowledge Center – SYSCHECKS Table](https://www.ibm.com/docs/en/db2-for-zos/12?topic=catalog-syschecks)  
- [IBM Redbooks – DB2 for z/OS: The Database Administrator’s Guide](https://www.redbooks.ibm.com/)  
- [IBM DB2 Performance and Tuning Guidelines](https://www.ibm.com/support/pages/db2-zos-performance-tuning-guidelines)  

---

### Observações:

- Certifique-se de que a opção **"Generate CHECK constraints"** esteja habilitada nas configurações do modelo.
- As constraints também podem ser visualizadas em scripts gerados via *Database Generation* ou *DDL Preview*.
- Use **Extended Attributes** para padronizar nomeação de regras entre diferentes tabelas e domínios.

---
