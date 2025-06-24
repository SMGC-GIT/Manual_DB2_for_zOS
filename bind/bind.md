# 📘 Guia Completo: BIND no DB2 for z/OS

> Elaborado para atuação sênior em ambientes corporativos de missão crítica, com base na documentação oficial da IBM.

---

## 📑 Índice

- [📌 1. Visão Geral](#1-visão-geral)
- [🧱 2. Estrutura do BIND](#2-estrutura-do-bind)
- [📎 3. Sintaxe do BIND](#3-sintaxe-do-bind)
- [🔍 4. Parâmetros Explicados](#4-parâmetros-explicados)
- [📈 5. Quando Atualizar o BIND](#5-quando-atualizar-o-bind)
- [🔁 6. REBIND: Atualizando sem Recompilar](#6-rebind-atualizando-sem-recompilar)
- [📊 7. Boas Práticas em Ambientes Críticos](#7-boas-práticas-em-ambientes-críticos)
- [📚 8. Tabelas do Catálogo Relacionadas](#8-tabelas-do-catálogo-relacionadas)
- [🧪 9. Exemplo Prático](#9-exemplo-prático)
- [📘 10. Glossário Técnico](#10-glossário-técnico)
- [🔗 11. Fontes Oficiais IBM](#11-fontes-oficiais-ibm)
- [🛠️ 12. Consultas SQL Úteis para Gestão de Packages](#12-consultas-sql-úteis-para-gestão-de-packages)

---

## 📌 1. Visão Geral

O **comando BIND** transforma instruções SQL compiladas (armazenadas em **DBRMs**) em **packages** ou **plans** executáveis pelo DB2. Ele define como e sob quais condições essas instruções serão executadas.

Além disso, o BIND:

- Associa o SQL compilado a um ambiente (usuário, esquema, estratégia de acesso).
- Garante controle de segurança, isolamento de transações e versionamento.
- Permite atualização de estratégias de acesso sem recompilação do programa.

---

## 🧱 2. Estrutura do BIND

### Objetos envolvidos:

| Objeto      | Função no processo                           |
|-------------|----------------------------------------------|
| **DBRM**    | Contém o SQL compilado (pré-compilador DB2)  |
| **PACKAGE** | Unidade de execução modular (moderna)        |
| **PLAN**    | Agregador de DBRMs (modelo legado)           |
| **COLLECTION** | Conjunto lógico de packages                |

---

## 📎 3. Sintaxe do BIND

### ✅ BIND PACKAGE

```sql
BIND PACKAGE('COLECAO') MEMBER('PROGRAMA')
  QUALIFIER(MYSCHEMA)
  OWNER(DBAUSR)
  VALIDATE(BIND)
  ISOLATION(CS)
  RELEASE(COMMIT)
  EXPLAIN(YES)
  ACQUIRE(USE)
  APPLCOMPAT(V12R1M510)
```

### ✅ BIND PLAN (legado)

```sql
BIND PLAN(MYPLAN)
  PKLIST(COLECAO.PROGRAMA1 COLECAO.PROGRAMA2)
  VALIDATE(RUN)
  ISOLATION(RR)
  EXPLAIN(NO)
```

---

## 🔍 4. Parâmetros Explicados

| Parâmetro       | Descrição                                                                                                                                   |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `QUALIFIER`      | Define o schema padrão para objetos SQL. Permite usar SQL sem prefixo de esquema. Ex: `SELECT * FROM CLIENTES` → usa `MYSCHEMA.CLIENTES`. |
| `OWNER`          | Define o proprietário do package (controla permissões de execução e REBIND).                                                                |
| `VALIDATE`       | `BIND`: checa permissões no momento do bind. `RUN`: posterga para o runtime (pode causar falhas futuras).                                  |
| `ISOLATION`      | Define como os dados são bloqueados: `CS` (Cursor Stability), `RR` (Repeatable Read), `UR` (Uncommitted Read), `RS` (Read Stability).      |
| `RELEASE`        | `COMMIT`: libera locks ao final da transação. `DEALLOCATE`: mantém recursos até a thread encerrar (mais eficiente para programas longos).  |
| `EXPLAIN`        | `YES`: armazena o plano de acesso na `PLAN_TABLE`. Fundamental para análise de performance.                                                 |
| `ACQUIRE`        | `USE`: aloca locks quando necessário. `ALLOCATE`: aloca tudo no início (usado com `RELEASE DEALLOCATE`).                                   |
| `APPLCOMPAT`     | Define compatibilidade com versões específicas do DB2 (e.g., `V12R1M510`). Crucial para evitar regressões após upgrade.                    |

---

## 📈 5. Quando Atualizar o BIND

Atualizar um BIND é necessário quando há **mudanças estruturais** ou **estratégicas** que afetam a execução do SQL. Exemplos:

### ➕ Alterações que exigem BIND/REBIND:

- Alteração em **índices ou colunas** de uma tabela (DDL)
- Atualização de **estatísticas** via `RUNSTATS`
- Modificação de **views**, **sinônimos** ou **triggers**
- Alterações em **autorizações** de objetos referenciados
- Atualização de **versão do DB2** (`APPLCOMPAT`)
- Correções ou melhorias em parâmetros como `ISOLATION`, `RELEASE`, etc.

### 🛠️ Importância:

Se o BIND não for atualizado:

- O plano de acesso pode se tornar ineficiente
- O programa pode falhar na execução (SQLCODE -805, -818, -805)
- Pode ocorrer regressão de performance em produção

---

## 🔁 6. REBIND: Atualizando sem Recompilar

O comando `REBIND PACKAGE` permite **recompilar o plano de acesso** de um package existente, sem alterar o código-fonte nem recompilar o DBRM.

### ✅ Sintaxe

```sql
REBIND PACKAGE('COLECAO') MEMBER('PROGRAMA')
  EXPLAIN(YES)
  APPLCOMPAT(V12R1M510)
  VALIDATE(BIND)
```

### 📍 Quando usar:

- Após atualização de estatísticas (RUNSTATS)
- Após mudanças de estrutura (índices, colunas)
- Para forçar reavaliação da estratégia do otimizador
- Após instalação de manutenção (PTFs)
- Ao adotar nova política de compatibilidade (`APPLCOMPAT`)

---

## 📊 7. Boas Práticas em Ambientes Críticos

- **Padronizar QUALIFIER e OWNER** por sistema, ambiente e aplicação
- Sempre utilizar `EXPLAIN(YES)` para monitorar e auditar estratégias de acesso
- Usar `RELEASE(DEALLOCATE)` em programas reutilizáveis ou threads CICS
- Controlar permissões de `REBIND` via perfis e roles DB2 (ex: DBADM, BINDADD)
- Criar políticas de versionamento com `COLLECTION` por release
- Validar com `VALIDATE(BIND)` em ambientes de homologação e produção
- Analisar pacotes antigos com `SYSPACKAGE.LASTUSED` e considerar free ou cleanup

---

## 📚 8. Tabelas do Catálogo Relacionadas

| Tabela                    | Descrição                                                             |
|---------------------------|----------------------------------------------------------------------|
| `SYSIBM.SYSPACKAGE`       | Metadados do package (flags, data do bind, isolation, etc.)          |
| `SYSIBM.SYSPACKDEP`       | Dependências do package (tabelas, views, aliases, funções)           |
| `SYSIBM.SYSPACKAUTH`      | Autorizações concedidas para execução, bind, etc.                    |
| `SYSIBM.SYSPACKSTMT`      | Instruções SQL individuais compiladas no package                     |
| `SYSIBM.SYSPACKCOPY`      | Histórico de versões anteriores mantidas com `COPY` para fallback    |

---

## 🧪 9. Exemplo Prático

### 📂 Situação:

Foi realizada uma alteração de índices e atualizadas as estatísticas da tabela `TRANSACOES_FINANCEIRAS`.

### ✅ Ação:

```sql
REBIND PACKAGE('PKGTRANSACOES') MEMBER('PG001')
  EXPLAIN(YES)
  VALIDATE(BIND)
  APPLCOMPAT(V12R1M510)
```

### 🎯 Resultado:

- Novo plano de acesso otimizado
- SQLs ajustadas à nova estrutura e estatísticas
- Análise de performance disponível na `PLAN_TABLE`

---

## 📘 10. Glossário Técnico

| Termo           | Definição |
|------------------|----------|
| **BIND**         | Processo que converte o DBRM em um package executável no DB2 |
| **DBRM**         | Módulo de solicitação de banco de dados, gerado na pré-compilação |
| **PACKAGE**      | Unidade modular de execução no DB2, mais moderno que PLAN |
| **PLAN**         | Objeto legado que agregava DBRMs para execução |
| **COLLECTION**   | Conjunto lógico de packages agrupados sob um nome comum |
| **QUALIFIER**    | Esquema substituto usado em tempo de execução no SQL |
| **APPLCOMPAT**   | Compatibilidade da aplicação com versão do DB2 |
| **EXPLAIN**      | Recurso que grava o plano de acesso para análise de performance |
| **RELEASE**      | Define quando os recursos são liberados (COMMIT ou DEALLOCATE) |
| **ISOLATION**    | Nível de isolamento de transações SQL (CS, RR, UR, RS) |
| **RUNSTATS**     | Coleta estatísticas das tabelas para o otimizador do DB2 |
| **REBIND**       | Reprocessa o package para gerar novo plano de acesso |
| **OWNER**        | Usuário proprietário do objeto BIND |
| **VALIDATE**     | Modo de verificação de permissões: BIND ou RUN |
| **LASTUSED**     | Campo que indica a última execução do package |

---

## 🔗 11. Fontes Oficiais IBM

- 📖 [BIND PACKAGE - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-bind-package)
- 📖 [REBIND PACKAGE - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-rebind-package)
- 📖 [SYSPACKAGE Catalog - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-syspackage)
- 📖 [APPLCOMPAT - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=reference-applcompat)

---

## 🛠️ 12. Consultas SQL Úteis para Gestão de Packages

### 🔎 12.1. Pacotes utilizados recentemente

```sql
SELECT COLLID, NAME, VERSION, LASTUSED
FROM SYSIBM.SYSPACKAGE
WHERE LASTUSED IS NOT NULL
ORDER BY LASTUSED DESC;
```

### 💤 12.2. Pacotes não utilizados nos últimos 90 dias

```sql
SELECT COLLID, NAME, VERSION, LASTUSED
FROM SYSIBM.SYSPACKAGE
WHERE LASTUSED < CURRENT DATE - 90 DAYS;
```

### 🚫 12.3. Pacotes com status inválido

```sql
SELECT COLLID, NAME, VALID
FROM SYSIBM.SYSPACKAGE
WHERE VALID = 'N';
```

### 🧵 12.4. Listar programas associados a um plano (PKLIST)

```sql
SELECT *
FROM SYSIBM.SYSPLANDEP
WHERE BNAME = 'NOME_DO_PLAN';
```

### 🔗 12.5. Ver dependências de um package

```sql
SELECT * 
FROM SYSIBM.SYSPACKDEP
WHERE COLLID = 'COLECAO' AND NAME = 'PROGRAMA';
```

### 🔐 12.6. Ver permissões concedidas em packages

```sql
SELECT *
FROM SYSIBM.SYSPACKAUTH
WHERE COLLID = 'COLECAO';
```

---

> Este guia está preparado para uso em treinamentos, operações de produção, ou auditoria de qualidade em ambientes DB2 for z/OS. Pode ser expandido com temas como: análise de EXPLAIN, automação de REBIND, gestão de versionamento de packages e mais.

