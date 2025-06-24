# 📘 Guia Completo: BIND no DB2 for z/OS

> Elaborado para atuação sênior em ambientes corporativos de missão crítica, com base na documentação oficial da IBM.

---

- [1. Visão Geral](#1-visão-geral)
- [2. Estrutura do BIND](#2-estrutura-do-bind)
- [3. Sintaxe do BIND](#3-sintaxe-do-bind)
- [4. Parâmetros Explicados](#4-parâmetros-explicados)
- [5. Quando Atualizar o BIND](#5-quando-atualizar-o-bind)
- [6. REBIND: Atualizando sem Recompilar](#6-rebind-atualizando-sem-recompilar)
- [7. Boas Práticas em Ambientes Críticos](#7-boas-práticas-em-ambientes-críticos)
- [8. Tabelas do Catálogo Relacionadas](#8-tabelas-do-catálogo-relacionadas)
- [9. Exemplo Prático](#9-exemplo-prático)
- [10. Glossário Técnico](#10-glossário-técnico)
- [11. Fontes Oficiais IBM](#11-fontes-oficiais-ibm)
- [12. Consultas SQL Úteis para Gestão de Packages](#12-consultas-sql-úteis-para-gestão-de-packages)
- [13. Script Automatizado para REBIND em Lote](#13-script-automatizado-para-rebind-em-lote)
- [14. COPY PACKAGE e Estratégias de Fallback](#14-copy-package-e-estratégias-de-fallback)
- [15. FREE PACKAGE e Limpeza de Pacotes Obsoletos](#15-free-package-e-limpeza-de-pacotes-obsoletos)
- [16. Análise de Performance com EXPLAIN e PLAN_TABLE](#16-análise-de-performance-com-explain-e-plan_table)
- [17. Estratégias de Controle com VERSION](#17-estratégias-de-controle-com-version)
- [18. Erros Comuns Relacionados ao BIND](#18-erros-comuns-relacionados-ao-bind)
- [19. Checklist de Diagnóstico de Pacotes Inválidos](#19-checklist-de-diagnóstico-de-pacotes-inválidos)
- [20. Playbook de REBIND Emergencial](#20-playbook-de-rebind-emergencial)

---

## 1. Visão Geral

O **comando BIND** transforma instruções SQL compiladas (armazenadas em **DBRMs**) em **packages** ou **plans** executáveis pelo DB2. Ele define como e sob quais condições essas instruções serão executadas.

Além disso, o BIND:

- Associa o SQL compilado a um ambiente (usuário, esquema, estratégia de acesso).
- Garante controle de segurança, isolamento de transações e versionamento.
- Permite atualização de estratégias de acesso sem recompilação do programa.

---

## 2. Estrutura do BIND

### Objetos envolvidos:

| Objeto      | Função no processo                           |
|-------------|----------------------------------------------|
| **DBRM**    | Contém o SQL compilado (pré-compilador DB2)  |
| **PACKAGE** | Unidade de execução modular (moderna)        |
| **PLAN**    | Agregador de DBRMs (modelo legado)           |
| **COLLECTION** | Conjunto lógico de packages                |

---

## 3. Sintaxe do BIND

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

## 4. Parâmetros Explicados

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

## 5. Quando Atualizar o BIND

Atualizar um BIND é necessário quando há **mudanças estruturais** ou **estratégicas** que afetam a execução do SQL. Exemplos:

### ➕ Alterações que exigem BIND/REBIND:

- Alteração em **índices ou colunas** de uma tabela (DDL)
- Atualização de **estatísticas** via `RUNSTATS`
- Modificação de **views**, **sinônimos** ou **triggers**
- Alterações em **autorizações** de objetos referenciados
- Atualização de **versão do DB2** (`APPLCOMPAT`)
- Correções ou melhorias em parâmetros como `ISOLATION`, `RELEASE`, etc.
- Alterações em tabelas, índices, views ou triggers
- Modificação de permissões
- Mudanças no plano de execução por performance

### 🛠️ Importância:

Se o BIND não for atualizado:

- O plano de acesso pode se tornar ineficiente
- O programa pode falhar na execução (SQLCODE -805, -818, -805)
- Pode ocorrer regressão de performance em produção

---

## 6. REBIND: Atualizando sem Recompilar

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

## 7. Boas Práticas em Ambientes Críticos

- **Padronizar QUALIFIER e OWNER** por sistema, ambiente e aplicação
- Sempre utilizar `EXPLAIN(YES)` para monitorar e auditar estratégias de acesso
- Usar `RELEASE(DEALLOCATE)` em programas reutilizáveis ou threads CICS
- Controlar permissões de `REBIND` via perfis e roles DB2 (ex: DBADM, BINDADD)
- Criar políticas de versionamento com `COLLECTION` por release
- Validar com `VALIDATE(BIND)` em ambientes de homologação e produção
- Analisar pacotes antigos com `SYSPACKAGE.LASTUSED` e considerar free ou cleanup

---

## 8. Tabelas do Catálogo Relacionadas

| Tabela                    | Descrição                                                             |
|---------------------------|----------------------------------------------------------------------|
| `SYSIBM.SYSPACKAGE`       | Metadados do package (flags, data do bind, isolation, etc.)          |
| `SYSIBM.SYSPACKDEP`       | Dependências do package (tabelas, views, aliases, funções)           |
| `SYSIBM.SYSPACKAUTH`      | Autorizações concedidas para execução, bind, etc.                    |
| `SYSIBM.SYSPACKSTMT`      | Instruções SQL individuais compiladas no package                     |
| `SYSIBM.SYSPACKCOPY`      | Histórico de versões anteriores mantidas com `COPY` para fallback    |

---

## 9. Exemplo Prático

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

## 10. Glossário Técnico

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

## 11. Fontes Oficiais IBM

- 📖 [BIND PACKAGE - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-bind-package)
- 📖 [REBIND PACKAGE - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=commands-rebind-package)
- 📖 [SYSPACKAGE Catalog - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-syspackage)
- 📖 [APPLCOMPAT - IBM](https://www.ibm.com/docs/en/db2-for-zos/13?topic=reference-applcompat)

---

## 12. Consultas SQL Úteis para Gestão de Packages

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

## 13. Script Automatizado para REBIND em Lote

Automatizar o REBIND para pacotes antigos ou afetados por mudanças de RUNSTATS ou DDL pode ser crucial para performance e estabilidade. Abaixo, um exemplo de **script gerador de REBINDs dinâmicos**, baseado na tabela `SYSPACKAGE`.

### 🎯 Objetivo:

Gerar dinamicamente comandos `REBIND PACKAGE` apenas para pacotes válidos e com `LASTUSED` recente.

---

### 📋 13.1. Query SQL para gerar comandos REBIND

```sql
SELECT 
  'REBIND PACKAGE(''' || COLLID || ''') MEMBER(''' || NAME || ''') ' ||
  'APPLCOMPAT(V12R1M510) EXPLAIN(YES) VALIDATE(BIND);' AS REBIND_CMD
FROM SYSIBM.SYSPACKAGE
WHERE VALID = 'Y'
  AND LASTUSED >= CURRENT DATE - 90 DAYS
ORDER BY COLLID, NAME;
```

> 💡 **Dica:** Execute a query em um ambiente controlado (test/homolog) e avalie os REBINDs gerados antes de aplicar em produção.

---

### ⚠️ 13.2. Adaptação para REBIND em lote via JCL (Exemplo)

```jcl
//REBINDPK JOB (ACCT),'REBIND PACKAGES',
//         CLASS=A,MSGCLASS=X,NOTIFY=&SYSUID
//STEP1    EXEC PGM=IKJEFT01,DYNAMNBR=50
//SYSTSPRT DD  SYSOUT=*
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
REBIND PACKAGE(COL1) MEMBER(PGMA) APPLCOMPAT(V12R1M510)
EXPLAIN(YES) VALIDATE(BIND);
REBIND PACKAGE(COL2) MEMBER(PGMB) APPLCOMPAT(V12R1M510)
EXPLAIN(YES) VALIDATE(BIND);
/*
//SYSTSIN  DD  *
DSN SYSTEM(DB2X)
/*
```

---

### ✅ 13.3. Filtrar pacotes de uma aplicação específica

```sql
SELECT 
  'REBIND PACKAGE(''' || COLLID || ''') MEMBER(''' || NAME || ''') ' ||
  'APPLCOMPAT(V12R1M510) EXPLAIN(YES);' AS REBIND_CMD
FROM SYSIBM.SYSPACKAGE
WHERE COLLID LIKE 'APP01%'
  AND VALID = 'Y';
```

---

### 📌 Considerações:

- A execução em massa deve ser monitorada, com logs ativados.
- Pacotes com dependências inválidas podem falhar no REBIND.
- Mantenha um `BACKUP` com `COPY PACKAGE` ou `DISPLAY` antes de REBIND.
- Ideal executar em janela de manutenção com suporte online.

---

> Este processo é recomendado para ambientes onde o volume de pacotes torna inviável o REBIND manual. Avaliações periódicas com base em `LASTUSED`, `VALID`, e `RUNSTATS` devem fazer parte da governança de packages no DB2.

---

## 14. COPY PACKAGE e Estratégias de Fallback

O comando `COPY PACKAGE` permite **criar uma cópia de segurança de um package** em sua forma binária. Isso é extremamente útil antes de realizar `REBIND`, especialmente em produção, para permitir **rollback seguro** em caso de degradação de performance.

### 14.1. Sintaxe do COPY PACKAGE

```sql
COPY PACKAGE(COLECAO.PROGRAMA) 
  COPYID('BKP001');
```

- `COPYID` define uma versão identificável da cópia.
- Pode-se manter múltiplas cópias por package.

### 14.2. Restaurando com REBIND COPY

```sql
REBIND PACKAGE(COLECAO.PROGRAMA) 
  COPY(BKP001);
```

> 🔐 **Recomenda-se executar `COPY PACKAGE` antes de qualquer REBIND em produção.** Assim, é possível voltar ao plano anterior sem nova compilação.

---

## 15. FREE PACKAGE e Limpeza de Pacotes Obsoletos

Pacotes que não são mais utilizados devem ser removidos para liberar recursos e manter o catálogo limpo.

### 15.1. Verificando pacotes antigos

```sql
SELECT COLLID, NAME, LASTUSED
FROM SYSIBM.SYSPACKAGE
WHERE LASTUSED < CURRENT DATE - 180 DAYS;
```

### 15.2. FREE PACKAGE

```sql
FREE PACKAGE(COLECAO.PROGRAMA);
```

> ⚠️ Se o pacote estiver em uso por algum plan, a exclusão pode falhar.

### 15.3. Limpeza completa

```sql
FREE PACKAGE(COLECAO.PROGRAMA)
  PLAN(PLANO) ACTION(REMOVE);
```

---

## 16. Análise de Performance com EXPLAIN e PLAN_TABLE

O parâmetro `EXPLAIN(YES)` gera informações sobre o plano de acesso que o DB2 usará para executar o SQL, armazenadas na `PLAN_TABLE`.

### 16.1. Gerando dados com BIND/REBIND

```sql
REBIND PACKAGE(COLECAO.PROGRAMA) 
  EXPLAIN(YES);
```

### 16.2. Consulta básica na PLAN_TABLE

```sql
SELECT QUERYNO, METHOD, TABNO, ACCESSNAME, MATCHCOLS, PREFETCH
FROM PLAN_TABLE
WHERE QUERYNO = 1;
```

### 16.3. Campos importantes

- `ACCESSNAME`: nome do índice utilizado
- `MATCHCOLS`: colunas usadas como match
- `METHOD`: tipo de join
- `PREFETCH`: técnica de pré-busca de páginas

> 🔎 Use essas informações para detectar scans, joins ineficientes e ausência de índice.

---

## 17. Estratégias de Controle com VERSION

O uso de `VERSION` no `BIND PACKAGE` permite manter **múltiplas versões de um mesmo programa**, úteis em:

- Homologação vs. Produção
- Blue/Green Deployment
- Retenção de histórico para fallback

### 17.1. Criando versão nomeada

```sql
BIND PACKAGE(COLECAO) MEMBER(PROGRAMA)
  VERSION(V001)
  ISOLATION(CS)
  EXPLAIN(YES);
```

### 17.2. Rebind de uma versão específica

```sql
REBIND PACKAGE(COLECAO) MEMBER(PROGRAMA) VERSION(V001)
  EXPLAIN(YES);
```

### 17.3. Remoção de versões antigas

```sql
FREE PACKAGE(COLECAO.PROGRAMA) VERSION(V001);
```

> 🧩 Combine `VERSION` com `COPY PACKAGE` para implementar uma estratégia robusta de fallback por versão.


---

## 18. Erros Comuns Relacionados ao BIND

Abaixo estão os erros mais recorrentes relacionados ao ciclo de vida dos packages. Cada um inclui a mensagem, causa técnica, explicação aprofundada e uma ou mais soluções eficazes.

---

### 18.1. **-805: Package not found**

**Mensagem:**
```
DSNT408I SQLCODE = -805 
THE PACKAGE 'COLLID.PROGRAMA.VERSION' WAS NOT FOUND
```

**Causa:**
- O programa executável está chamando um `PACKAGE` que não existe no catálogo `SYSPACKAGE`.
- Isso ocorre normalmente após deploy de uma nova versão sem executar o BIND correspondente.
- Também pode ocorrer se o plano (`PLAN`) estiver com `PKLIST` incorreta.

**Explicação Técnica:**
Durante a execução, o DB2 tenta localizar o package referenciado no precompilado via `COLLID.PROGRAMA.VERSION`. Se não encontrar, a execução falha. Esse erro costuma aparecer em ambientes de produção logo após um deploy incompleto.

**Soluções:**
1. Verifique se o package foi bindado:
   ```sql
   SELECT * FROM SYSIBM.SYSPACKAGE 
   WHERE NAME = 'PROGRAMA' AND COLLID = 'COLLID';
   ```
2. Caso não exista, gere novamente o DBRM e execute:
   ```sql
   BIND PACKAGE(COLLID) MEMBER(PROGRAMA) VERSION(...) ...
   ```
3. Verifique se o plano (`PLAN`) inclui o `PKLIST` correto.

---

### 18.2. **-818: Timestamp mismatch**

**Mensagem:**
```
THE PRECOMPILER GENERATED TIMESTAMP x IN THE LOAD MODULE 
DOES NOT MATCH THE BIND TIMESTAMP y IN THE DBRM
```

**Causa:**
- O load module (.LOAD) e o package referenciam timestamps diferentes.
- Isso ocorre quando se recompila o programa mas não se faz REBIND.

**Explicação Técnica:**
O DB2 associa um timestamp único a cada compilação (DBRM) e compara com o do load. Se houver divergência, o runtime entende que o programa e o plano de acesso estão inconsistentes.

**Soluções:**
- Recompile o programa e rebinde imediatamente.
- Em pipelines de deploy, nunca separar compilação e BIND.

---

### 18.3. **-922: Authorization Failure**

**Mensagem:**
```
DSNT408I SQLCODE = -922 
AUTHORIZATION FAILURE: error-type ERROR
```

**Causa:**
- O usuário que executa o programa não possui `EXECUTE` no package.
- O OWNER do BIND pode não ter GRANT adequado.

**Explicação Técnica:**
O controle de acesso no DB2 está vinculado à execução do package. Se o usuário final não estiver autorizado via `SYSPACKAUTH`, a execução falha.

**Soluções:**
```sql
GRANT EXECUTE ON PACKAGE COLLID.PROGRAMA TO USER USUARIO;
```
Ou conceda via ROLE ou grupo autorizado.

---

### 18.4. **DSNT201I - Package was invalidated**

**Mensagem:**
```
DSNT201I - PACKAGE 'COLLID.PROG' WAS INVALIDATED BY DDL CHANGE
```

**Causa:**
- Alterações de estrutura (DDL) nas tabelas referenciadas pelo package.

**Explicação Técnica:**
O catálogo detecta que o plano de acesso está desatualizado e invalida o package automaticamente para garantir consistência.

**Soluções:**
- Executar REBIND imediatamente após ALTER TABLE, DROP INDEX, etc.
- Use:
```sql
REBIND PACKAGE(COLLID) MEMBER(PROGRAMA) ...
```

---

### 18.5. **-530: Constraint Violation após alteração de estrutura**

**Mensagem:**
```
FOREIGN KEY VIOLATION DURING INSERT OR UPDATE
```

**Causa:**
- Mudanças em constraints que afetam pacotes que manipulam essas tabelas.
- Pode haver impacto no plano de acesso.

**Soluções:**
- Verifique se o programa está respeitando a nova constraint.
- REBIND do pacote envolvido pode ser necessário.

---

### 18.6. **Falha de REBIND por falta de estatísticas**

**Sintoma:**
- Plano de acesso inesperado, uso excessivo de TABLE SCAN, join ineficiente
- `PLAN_TABLE` mostra CARD = -1

**Explicação Técnica:**
O otimizador depende de estatísticas atualizadas para gerar o melhor plano de acesso. Sem elas, assume defaults ineficientes.

**Solução:**
```sql
RUNSTATS TABLESPACE DB.TS TABLE(ALL) INDEX(ALL)
REBIND PACKAGE(COLLID.PROGRAMA) EXPLAIN(YES)
```

---

## 19. Checklist de Diagnóstico de Pacotes Inválidos

### 🎯 Objetivo:
Identificar e tratar pacotes inválidos no catálogo `SYSIBM.SYSPACKAGE`, que estão impedindo a execução correta de programas no DB2. Pacotes inválidos ocorrem geralmente após alterações de tabelas, recompile sem REBIND, deploy incorreto, problemas de permissão ou estatísticas desatualizadas.

---

### ✅ Etapas de Diagnóstico

---

#### 🔹 **Passo 1 – Listar pacotes inválidos**

```sql
SELECT COLLID, NAME, VERSION, VALID 
FROM SYSIBM.SYSPACKAGE 
WHERE VALID = 'N';
```

**O que estou fazendo aqui?**  
Você está localizando todos os packages que estão **marcados como inválidos (VALID = 'N')**, ou seja, que precisam obrigatoriamente de um REBIND para funcionar.

---

#### 🔹 **Passo 2 – Verificar se o pacote ainda é utilizado**

```sql
SELECT COLLID, NAME, LASTUSED 
FROM SYSIBM.SYSPACKAGE 
WHERE VALID = 'N';
```

**Por quê isso importa?**  
Se `LASTUSED` estiver muito antiga (ex: mais de 180 dias), o pacote pode ser considerado **obsoleto** e removido com `FREE PACKAGE`. Caso contrário, é um forte indício de que há impacto real em produção e é necessário atuar com urgência.

---

#### 🔹 **Passo 3 – Analisar dependências do pacote**

```sql
SELECT * 
FROM SYSIBM.SYSPACKDEP 
WHERE COLLID = 'COLLID' AND NAME = 'PROGRAMA';
```

**Por que isso é importante?**  
Mostra quais objetos o pacote depende (tabelas, índices, views). Se algum deles foi alterado (ex: `ALTER TABLE`), o pacote pode ter sido invalidado automaticamente e precisa de `REBIND`.

---

#### 🔹 **Passo 4 – Validar permissões associadas**

```sql
SELECT * 
FROM SYSIBM.SYSPACKAUTH 
WHERE COLLID = 'COLLID' AND NAME = 'PROGRAMA';
```

**Motivo:**  
Verifica se o usuário que está executando o programa tem `EXECUTE` no package. Erros como `-922` podem ocorrer por ausência de autorização, mesmo que o pacote esteja válido.

---

#### 🔹 **Passo 5 – Confirmar plano de acesso (EXPLAIN)**

Após o REBIND, execute:
```sql
EXPLAIN PACKAGE(COLLID.PROGRAMA)
```
ou rebind com EXPLAIN:

```sql
REBIND PACKAGE(COLLID) MEMBER(PROGRAMA) 
EXPLAIN(YES);
```

E analise:
```sql
SELECT QUERYNO, METHOD, MATCHCOLS, PREFETCH 
FROM PLAN_TABLE 
WHERE COLLID = 'COLLID';
```

**Para quê serve isso?**  
Garante que o plano de acesso está otimizado e reflete as estatísticas mais recentes. Pode revelar uso de TABLE SCAN indesejado, joins ruins, ausência de índices, etc.

---

### 🧩 Conclusão

Este checklist permite:

- Localizar pacotes inválidos
- Priorizar REBIND conforme uso real
- Entender por que o pacote foi invalidado
- Garantir que permissões e dependências estejam corretas
- Verificar se o novo plano gerado após REBIND é eficiente

> 💡 **Dica extra**: Automatize esse checklist via stored procedure, REXX ou script em JCL para rodar em ambientes grandes regularmente.


---

## 20. Playbook de REBIND Emergencial

### 🎯 Objetivo:
Orientar o DBA no tratamento rápido e seguro de falhas de execução de programas causadas por packages inválidos, ausentes, com permissões incorretas ou fora de sincronia com o executável.

Este playbook é voltado para **ambientes de produção**, onde tempo e assertividade são críticos.

---

### 📌 Situações típicas onde o REBIND é necessário com urgência:

- Após deploy, usuários recebem erros `-805`, `-818`, `DSNT201I`, ou o programa trava sem retorno.
- Pacote foi invalidado após `ALTER TABLE`, `DROP INDEX`, etc.
- Load module recompilado sem atualização do DBRM.
- Package foi FREE sem REBIND posterior.
- Permissão de execução foi revogada acidentalmente.

---

### 🧰 Etapas de Resolução

---

#### 🔹 **Passo 1 – Verifique se o package existe no catálogo**

```sql
SELECT COLLID, NAME, VERSION 
FROM SYSIBM.SYSPACKAGE 
WHERE NAME = 'PROGRAMA';
```

**Para quê?**  
Confirma se o package foi bindado corretamente. Se não existir, o erro mais provável será `-805`.

---

#### 🔹 **Passo 2 – Verifique se o package está inválido**

```sql
SELECT COLLID, NAME, VALID 
FROM SYSIBM.SYSPACKAGE 
WHERE NAME = 'PROGRAMA' AND VALID = 'N';
```

**Para quê?**  
Um package inválido geralmente resulta em falhas silenciosas ou `DSNT201I`.

---

#### 🔹 **Passo 3 – Efetue o REBIND com parâmetros apropriados**

```sql
REBIND PACKAGE(COLLID) MEMBER(PROGRAMA) 
    EXPLAIN(YES) VALIDATE(BIND);
```

**Detalhes:**
- `EXPLAIN(YES)` gera novo plano na `PLAN_TABLE`.
- `VALIDATE(BIND)` força validação completa no momento do REBIND.

---

#### 🔹 **Passo 4 – Se falhar, recompile e rebinde**

**Procedimento completo:**

1. Compile o programa fonte (ex: COBOL ou PL/I).
2. Gere novo DBRM.
3. Execute:
   ```sql
   BIND PACKAGE(COLLID) MEMBER(PROGRAMA) VERSION(V001) ...
   ```

**Dica:**  
Garanta que o `VERSION` usado na compilação seja compatível com o BIND.

---

#### 🔹 **Passo 5 – Corrija permissões de execução**

```sql
GRANT EXECUTE ON PACKAGE COLLID.PROGRAMA 
TO USER USUARIO;
```

**Para quê?**  
Erros como `-922` indicam que o executor perdeu permissão no package. Isso pode ocorrer após DROP/REBIND, ou alteração de OWNER.

---

#### 🔹 **Passo 6 – Restaure versão anterior, se necessário**

```sql
REBIND PACKAGE(COLLID.PROGRAMA) 
COPY(BKP001);
```

**Para quê?**  
Restaura uma versão funcional previamente salva com `COPY PACKAGE`. Ideal para rollback imediato.

---

#### 🔹 **Passo 7 – Teste funcional após REBIND**

Execute uma transação real (ou job batch) com tracing ativo e monitore:

- Se o erro original foi resolvido
- Se o plano de acesso gerado é eficiente (`EXPLAIN`)
- Se estatísticas estão em uso correto (`MATCHCOLS`, `METHOD`)

---

### 📌 Checklist Final:

| Verificação                                 | Status Esperado     |
|--------------------------------------------|---------------------|
| Package existe no catálogo?                | Sim                 |
| VALID = 'Y'?                                | Sim                 |
| Último REBIND recente?                     | Sim ou justificável |
| Permissões de execução conferidas?         | Sim                 |
| Plano de acesso revisado via EXPLAIN?      | Sim                 |
| COPY PACKAGE de segurança disponível?      | Sim (em produção)   |

---

### 💡 Dicas Profissionais

- Sempre execute `COPY PACKAGE` antes de rebinde em produção:
  ```sql
  COPY PACKAGE(COLLID.PROGRAMA) COPYID('PRE-REBIND');
  ```

- Para rebind em lote, use `DSNTPSMP` ou JCL com `DSNTIAD`.

- Automatize o REBIND com base em eventos de invalidação (`SYSPACKAGE.VALID = 'N'`).

---

> ⚠️ Um REBIND emergencial pode restaurar a funcionalidade, mas deve ser seguido de análise pós-ocorrência para identificar causas-raiz (ex: deploy incompleto, ausência de RUNSTATS, ordem de operações errada).


---

> 💡 Dica: Padronize a criação de `COPY PACKAGE` após cada BIND/REBIND crítico para permitir rollback imediato em produção.

---

> ✅ Esta seção pode ser expandida com SQLCODEs adicionais sob demanda e correlacionada com logs de falha em produção, como `DSNT376I`, `DSNT500I`, entre outros.

---

> Este guia está preparado para uso em treinamentos, operações de produção, ou auditoria de qualidade em ambientes DB2 for z/OS. Pode ser expandido com temas como: análise de EXPLAIN, automação de REBIND, gestão de versionamento de packages e mais.

