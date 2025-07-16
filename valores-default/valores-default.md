# DEFAULT em Colunas no DB2 for z/OS

---

## 📑 Índice

- [📘 1. O que é o DEFAULT no DB2](#-1-o-que-é-o-default-no-db2)
- [⚙️ 2. Funcionamento Geral](#️-2-funcionamento-geral)
- [🧬 3. Tipos de DEFAULT por Data Type](#-3-tipos-de-default-por-data-type)
- [✅ 4. Vantagens](#-4-vantagens)
- [⚠️ 5. Cuidados e Desvantagens](#️-5-cuidados-e-desvantagens)
- [📈 6. Performance e Impactos](#-6-performance-e-impactos)
- [🔄 7. Inclusão, Alteração e Drop](#-7-inclusão-alteração-e-drop)
- [📌 8. Adendo: NOT NULL WITH DEFAULT vs NULL WITH DEFAULT](#-8-adendo-not-null-with-default-vs-null-with-default)
- [📚 9. Referências Oficiais IBM](#-9-referências-oficiais-ibm)
- [✅ 10. Considerações Finais](#-10-considerações-finais)

---

## 📘 1. O que é o DEFAULT no DB2

O `DEFAULT` define um **valor padrão** atribuído automaticamente a uma coluna quando um valor não é fornecido durante a execução de um `INSERT`. Ele garante preenchimento automático de campos e padronização de valores quando ausentes.

---

## ⚙️ 2. Funcionamento Geral

- O DEFAULT pode ser especificado no momento do `CREATE TABLE` ou com `ALTER TABLE`.
- Pode conter:
  - Um valor **constante literal** (`0`, `'A'`, `'2025-01-01'`)
  - Uma função especial (`CURRENT DATE`, `CURRENT TIME`, `CURRENT TIMESTAMP`)
  - Ou a palavra `NULL`, se a coluna for nullable
- Só é aplicado em `INSERT` quando o valor da coluna for omitido
- Não se aplica a `UPDATE`, a menos que explicitamente controlado

```sql
CREATE TABLE FUNCIONARIOS (
  ID         INTEGER       NOT NULL,
  NOME       VARCHAR(100)  NOT NULL,
  ATIVO      CHAR(1)       DEFAULT 'S',
  ADMISSAO   DATE          DEFAULT CURRENT DATE
);
```

---

## 🧬 3. Tipos de DEFAULT por Data Type

| Tipo de Dado                         | Valor Default Implícito | Pode Definir Customizado? | Observações |
|--------------------------------------|--------------------------|----------------------------|-------------|
| `SMALLINT`, `INTEGER`, `BIGINT`      | `0`                      | ✅ Sim                     | Numéricos inteiros |
| `DECIMAL`, `FLOAT`, `REAL`, `DOUBLE` | `0.0`                    | ✅ Sim                     | Numéricos com ponto flutuante |
| `CHAR`, `VARCHAR`                    | String vazia (`''`)      | ✅ Sim                     | Pode ser qualquer literal |
| `DATE`                               | `NULL`                   | ✅ Sim                     | Pode usar `CURRENT DATE` |
| `TIME`                               | `NULL`                   | ✅ Sim                     | Pode usar `CURRENT TIME` |
| `TIMESTAMP`                          | `NULL`                   | ✅ Sim                     | Pode usar `CURRENT TIMESTAMP` |
| `BLOB`, `CLOB`, `DBCLOB`             | `NULL`                   | ✅ Sim                     | Exige tamanho compatível |
| `ROWID`                              | Automático               | ❌ Não                     | Não aceita valor default definido |

---

## ✅ 4. Vantagens

- Simplifica `INSERT` de dados em colunas opcionais
- Evita erros por omissão de colunas
- Garante valores consistentes
- Facilita manutenção de código e evolução de tabelas
- Ideal para colunas como `STATUS`, `DATA_CRIACAO`, `USUARIO_CRIADOR`, etc.
- Melhora a clareza e padronização de regras de negócio

---

## ⚠️ 5. Cuidados e Desvantagens

- ❗ **Não se aplica a registros já existentes:**  
  O valor DEFAULT só é considerado durante a execução de comandos `INSERT` em que a coluna não é explicitamente referenciada. Dados já existentes na tabela **não são alterados retroativamente**, mesmo após uma mudança no valor default. Portanto, **não espere que os registros antigos sejam atualizados automaticamente** — qualquer alteração exigirá um `UPDATE` manual.

- ⚠️ **Uso indiscriminado pode ocultar falhas de lógica na aplicação:**  
  Aplicar defaults sem validação pode **mascarar erros** de entrada de dados. Por exemplo, uma coluna `STATUS CHAR(1) DEFAULT 'A'` pode acabar armazenando `'A'` indevidamente em casos onde o status deveria ter sido informado pela lógica do sistema. Isso pode comprometer a integridade de processos e relatórios.

- ❗ **Pode gerar falsa sensação de que o campo foi preenchido intencionalmente:**  
  Quando se consulta um campo com valor default preenchido automaticamente, é difícil saber se o valor foi realmente fornecido pela aplicação ou se foi apenas herdado por omissão. Isso pode afetar a **auditoria de dados** e a **compreensão do comportamento dos usuários ou sistemas**.

- ⚠️ **Alterações de DEFAULT em tabelas com muitos dados devem ser planejadas:**  
  Embora a alteração de um valor default não modifique registros existentes, a instrução `ALTER TABLE` pode causar **impactos em tempo de execução**, especialmente em ambientes com grande volume de dados ou alta concorrência. Dependendo da estrutura da tabela, tipo de tablespace e versionamento, o comando pode:
  - exigir uma nova versão da tabela;
  - bloquear recursos;
  - demandar rebinds de packages que utilizam a tabela.

- 🚫 **Não aceita expressões complexas nem funções definidas pelo usuário (UDF):**  
  O DB2 for z/OS restringe os valores default a **literais constantes** ou **funções built-in permitidas** (como `CURRENT DATE`, `CURRENT TIMESTAMP`). **Não é permitido** usar expressões aritméticas (`SALARIO * 1.1`), `CASE`, `COALESCE`, subqueries ou funções criadas pelo usuário (`UDFs`). Isso limita a lógica embutida nos defaults e exige que tais cálculos sejam feitos na aplicação ou via trigger.

- ❗ **Em instruções MERGE, o DEFAULT pode não ser aplicado conforme esperado:**  
  No contexto de um `MERGE INTO`, o uso de DEFAULT pode ser inconsistente, especialmente se os campos não forem explicitamente omitidos no `INSERT` dentro da cláusula `WHEN NOT MATCHED`. É essencial **testar cuidadosamente** a lógica do `MERGE` para garantir que o default será aplicado corretamente, caso a inserção ocorra.

---

## 📈 6. Performance e Impactos

- ✅ O uso de DEFAULT **não afeta negativamente a performance de leitura**
- ✅ Melhora a performance de `INSERT` em comparação a código de aplicação que tenta preencher valores manualmente
- ❗ ALTER TABLE com novo DEFAULT pode requerer `conversion phase` em tablespace particionado
- ✅ Não interfere diretamente nos índices
- ✅ Pode ser usado com `GENERATED ALWAYS` ou `GENERATED BY DEFAULT` em colunas geradas

---

## 🔄 7. Inclusão, Alteração e Drop

### ➕ Adicionar coluna com DEFAULT

```sql
ALTER TABLE CLIENTES ADD COLUMN SITUACAO CHAR(1) DEFAULT 'A';
```

### 📝 Alterar valor default existente

```sql
ALTER TABLE CLIENTES ALTER COLUMN SITUACAO SET DEFAULT 'I';
```

### ❌ Remover valor default

```sql
ALTER TABLE CLIENTES ALTER COLUMN SITUACAO DROP DEFAULT;
```

📌 Essas ações **não atualizam os dados já existentes** na tabela. Só afetam novos `INSERT`.

---

## 📌 8. Adendo: NOT NULL WITH DEFAULT vs NULL WITH DEFAULT

### 🔹 `NOT NULL WITH DEFAULT`

- A coluna **nunca aceita NULL** e **sempre terá um valor padrão** se nenhum for informado.
- Usado para garantir **consistência rígida** nos dados.
- O valor default será automaticamente inserido se o campo for omitido.
- **Ideal para:** flags (`'S'`/`'N'`), indicadores (`'A'`, `'I'`), datas de controle (`CURRENT DATE`).

### 🔹 `NULL WITH DEFAULT`

- A coluna **aceita NULLs**, mas pode receber um valor default **caso omitida no INSERT**.
- Se explicitamente informada como `NULL`, ela **permanece como NULL**.
- **Ideal para:** campos que podem ser preenchidos parcialmente ou futuramente (ex: `data_demissao`, `observacoes`).

### 🧾 Comportamento por Tipo com `NOT NULL WITH DEFAULT`

| Tipo         | Valor Default Implícito |
|--------------|--------------------------|
| INTEGER      | `0`                      |
| DECIMAL(10,2)| `0.00`                   |
| CHAR(1)      | `' '`                    |
| VARCHAR(50)  | `''` (string vazia)      |
| DATE         | `'0001-01-01'` (dependendo do SGBD) ou erro |
| TIMESTAMP    | `CURRENT TIMESTAMP` se explicitado |
| TIME         | `'00:00:00'` se suportado |

### 🧾 Comportamento por Tipo com `NULL WITH DEFAULT`

| Tipo         | Valor Default se omitido | Valor se informado NULL |
|--------------|--------------------------|--------------------------|
| INTEGER      | `0`                      | `NULL`                  |
| CHAR(1)      | `' '`                    | `NULL`                  |
| DATE         | `CURRENT DATE` (se definido) | `NULL`             |
| VARCHAR      | `''`                     | `NULL`                  |

📍 **Resumo prático:**  
- `NOT NULL WITH DEFAULT`: proteção + consistência  
- `NULL WITH DEFAULT`: flexibilidade + controle opcional

---

## 📚 9. Referências Oficiais IBM

- 🔗 [CREATE TABLE - IBM DB2 13 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-table)
- 🔗 [ALTER TABLE - IBM DB2 13 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-alter-table)
- 🔗 [DEFAULT Clause - DB2 SQL Reference](https://www.ibm.com/docs/en/db2-for-zos/13?topic=specifications-default-clause)
- 🔗 [DB2 13 SQL Reference Guide](https://www.ibm.com/docs/en/db2-for-zos/13?topic=reference-sql-statements)

---

## ✅ 10. Considerações Finais

Utilize `DEFAULT` para definir comportamentos padronizados de negócio, facilitando inserções e garantindo consistência. Porém, planeje bem seu uso para evitar que campos críticos fiquem com valores não intencionais. Analise sempre a necessidade de `NOT NULL WITH DEFAULT` versus `NULL WITH DEFAULT`, considerando a semântica da aplicação.

---
```
