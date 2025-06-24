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

### 🎯 Objetivo:
Entender os principais objetos envolvidos no processo de BIND no DB2 for z/OS e suas respectivas funções — desde a compilação do código-fonte até a execução do SQL no ambiente de produção.

---

### 🧱 Objetos envolvidos no ciclo de BIND

| Objeto        | Função no processo                                                         |
|---------------|-----------------------------------------------------------------------------|
| **DBRM**      | (*Database Request Module*) — Resultado do pré-compilador. Contém instruções SQL compiladas e metadados necessários para geração do plano de acesso. |
| **PACKAGE**   | Unidade modular de execução que encapsula o plano de acesso de um DBRM. Permite versionamento, REBIND isolado e controle granular. |
| **PLAN**      | Contêiner que agrega um ou mais packages. Necessário na execução em ambientes legados. Atualmente, cada PLAN referencia pacotes via `PKLIST`. |
| **COLLECTION**| Conjunto lógico que agrupa packages relacionados. Usada como namespace para facilitar organização, deploy, rollback e controle de versões. |

---

### 🔎 Explicação técnica de cada componente

---

#### 📄 **DBRM (Database Request Module)**

- Gerado pelo pré-compilador DB2 a partir do código fonte (COBOL, PL/I, C, etc.).
- Contém as instruções SQL em formato intermediário, além de informações de contexto (tabelas envolvidas, tipos de dados, etc.).
- **Não é executável** — serve de insumo para gerar o **PACKAGE**.

> 💡 Um novo DBRM é gerado toda vez que você recompila o programa-fonte.

---

#### 📦 **PACKAGE**

- É o resultado do comando `BIND PACKAGE`, que transforma um DBRM em um plano de acesso executável.
- Contém o plano otimizado pelo otimizador SQL, respeitando estatísticas, índices e configurações da época do bind.
- Suporta `VERSION`, permitindo múltiplas versões do mesmo programa coexistirem em produção.

**Vantagens do uso de PACKAGE:**
- Isolamento de alterações: é possível rebinder apenas um programa sem afetar os demais.
- Performance: permite reavaliação do plano (via `REBIND`) sem recompilar.
- Facilidade de rollback: possível usar `COPY PACKAGE` para restaurar versões anteriores.

---

#### 📋 **PLAN**

- Contêiner utilizado no momento da execução do programa.
- Refere-se aos packages através de `PKLIST` ou `PKLIST(*)`.
- Embora o uso de `PLAN` seja legado, ele **ainda é necessário** para vincular as transações CICS, batch ou TSO ao DB2.

> ⚠️ Não há mais `BIND PLAN` com `DBRMs` diretamente em sistemas modernos — o correto é `BIND PLAN` referenciando `PACKAGES`.

---

#### 📁 **COLLECTION**

- Nome lógico definido no comando `BIND PACKAGE`, que serve como agrupador de packages.
- Funciona como um "nome de pasta" para os pacotes — usado no momento de execução (chamada pelo plano).
- A collection define o escopo em que o programa será executado:
  
```sql
BIND PACKAGE(COL01) MEMBER(PROGRAMA) ...
```

- Permite deploys paralelos:
  - `COL01` → Produção atual
  - `COL02` → Versão em homologação
  - `COL_BKP` → Backup de versão estável

---

### ✅ Exemplo de fluxo típico com objetos envolvidos

1. Fonte COBOL é pré-compilado → gera DBRM
2. Executa `BIND PACKAGE(COLID)` → gera PACKAGE
3. Executa `BIND PLAN` com `PKLIST(COLID.PROGRAMA)`
4. Em tempo de execução, o DB2 localiza o plano → busca o package na collection → executa o SQL

---

### 🧠 Observações para DBAs modernos

- A IBM recomenda o uso de **PACKAGES + COLLECTIONS + BIND PLAN por PKLIST** como modelo padrão.
- Em ambientes atuais (V12+), os `DBRMs` **nunca** devem ser bindados diretamente em `BIND PLAN`.
- Versões e REBINDs são gerenciados diretamente por `PACKAGE`, sem recompilar fontes.

---

> 🎓 Dominar a estrutura do BIND é essencial para gerenciar performance, deploy seguro e troubleshooting em ambientes de missão crítica.

---

## 3. Sintaxe do BIND

### 🎯 Objetivo:
Apresentar a estrutura geral dos comandos `BIND PACKAGE` e `BIND PLAN`, explicando seus elementos obrigatórios e opcionais, com foco em boas práticas de uso para ambientes corporativos com múltiplas versões de programas.

---

### 📘 Comando `BIND PACKAGE`

Usado para gerar ou atualizar um **package** a partir de um **DBRM** previamente gerado na compilação do programa-fonte.

```sql
BIND PACKAGE('COLECAO') 
    MEMBER('NOME_PROGRAMA') 
    VALIDATE(BIND) 
    EXPLAIN(YES) 
    ISOLATION(CS) 
    RELEASE(COMMIT) 
    QUALIFIER('ESQUEMA_PADRAO') 
    OWNER('USUARIO') 
    APPLCOMPAT(V12R1M510);
```

---

### 🧩 Explicação dos elementos

| Parte             | Significado |
|------------------|-------------|
| `PACKAGE('COLID')` | Nome lógico do agrupador de packages (Collection ID). Organiza os pacotes por ambiente, sistema, release, etc. |
| `MEMBER('PROGRAMA')` | Nome do módulo fonte. Geralmente é o nome do programa compilado. Deve coincidir com o nome do DBRM. |
| `VALIDATE(BIND)`   | Garante que permissões e objetos sejam checados no momento do bind. |
| `EXPLAIN(YES)`     | Gera plano de acesso na PLAN_TABLE. |
| `ISOLATION(CS)`    | Define a política de bloqueio para cursores. |
| `RELEASE(COMMIT)`  | Libera recursos após o COMMIT. |
| `QUALIFIER('SCHEMA')` | Define o schema padrão de resolução de nomes SQL. |
| `OWNER('USUARIO')` | Define o dono do pacote no catálogo. |
| `APPLCOMPAT(...)`  | Garante compatibilidade com determinada versão do DB2. |

---

### 🧪 Exemplo real em ambiente de produção

```sql
BIND PACKAGE('FATURAMENTO') 
    MEMBER('REL_MENSAL') 
    QUALIFIER('CORP') 
    OWNER('DBA_USUARIO') 
    VALIDATE(BIND) 
    ISOLATION(CS) 
    RELEASE(COMMIT) 
    EXPLAIN(YES) 
    APPLCOMPAT(V12R1M510);
```

Este comando:
- Cria ou substitui o package `REL_MENSAL` na collection `FATURAMENTO`.
- Usa o schema `CORP` como padrão para todos os objetos SQL.
- Garante que tudo esteja correto no momento do bind (`VALIDATE(BIND)`).
- Gera o plano na `PLAN_TABLE` para posterior análise.
- Define a política de acesso e compatibilidade com `DB2 V12R1M510`.

---

### 📘 Comando `BIND PLAN`

Usado para criar um plano de execução (`PLAN`) que referencia um ou mais packages previamente bindados.

```sql
BIND PLAN('NOME_PLAN') 
    PKLIST(COLECAO1.MEMBRO1, COLECAO2.MEMBRO2) 
    VALIDATE(BIND) 
    ACTION(REPLACE) 
    ISOLATION(CS) 
    RELEASE(COMMIT) 
    OWNER('DBA_USUARIO');
```

---

### 📌 Notas importantes sobre o `BIND PLAN`

- É obrigatório quando o programa precisa de um plano de execução explícito (ambientes CICS, batch, TSO).
- `PKLIST` indica quais packages fazem parte do plano.
- A IBM recomenda usar sempre packages em vez de bind direto de DBRM no PLAN (modelo moderno).
- `ACTION(REPLACE)` sobrescreve um plano existente com os novos parâmetros.

---

### ✅ Boas práticas com `BIND`

- Sempre use `EXPLAIN(YES)` para análise de performance.
- Prefira `VALIDATE(BIND)` em produção, para garantir consistência e evitar falhas em runtime.
- Use `COPY PACKAGE` antes de `REBIND` em sistemas críticos.
- Mantenha versionamento e histórico de binds — ajuda na rastreabilidade e rollback.

---

> 🎓 Dominar a sintaxe e os parâmetros do BIND permite automatizar deploys, reduzir riscos e atuar de forma precisa em ambientes DB2 corporativos.




---

## 4. Parâmetros Explicados

### 🎯 Objetivo:
Compreender a finalidade e os impactos dos principais parâmetros utilizados nos comandos `BIND` e `REBIND PACKAGE`. Saber configurar corretamente esses parâmetros é fundamental para garantir segurança, performance e compatibilidade do ambiente de execução.

---

### 📋 Tabela Resumo dos Parâmetros

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

### 🧠 Detalhamento Técnico por Grupo de Parâmetros

---

#### 📌 `QUALIFIER` e `OWNER`

| Parâmetro | Função técnica |
|-----------|----------------|
| `QUALIFIER('MEUESQUEMA')` | Define o schema padrão em tempo de execução para resolução de nomes de objetos SQL. Se omitido, o nome será resolvido com base no CURRENT SQLID do executor. |
| `OWNER('USUARIO')`        | Define o proprietário do package no catálogo (`SYSPACKAGE.OWNER`). Esse usuário precisa ter `BIND` e, depois, `GRANT EXECUTE` pode ser delegado a outros. |

> ⚠️ **Cuidado**: se o OWNER não tiver permissões de acesso aos objetos durante o BIND, o processo falha (exceto se `VALIDATE(RUN)` for usado).

---

#### 🛡️ `VALIDATE`

| Valor | Comportamento |
|-------|----------------|
| `BIND` | Checa todas as permissões e existência dos objetos no momento do BIND. Mais seguro. Recomendado para produção. |
| `RUN`  | Permite o BIND mesmo sem permissões ou objetos existentes. Checagem será feita apenas em tempo de execução. Pode causar erros em runtime. |

> 💡 **Boas práticas:**  
Use `VALIDATE(BIND)` em produção para evitar deploys quebrados. Use `VALIDATE(RUN)` apenas em ambientes de desenvolvimento ou integração contínua.

---

#### 🔐 `ISOLATION` e `RELEASE`

| Parâmetro | Opções | Explicação |
|-----------|--------|------------|
| `ISOLATION` | `CS`, `RR`, `UR`, `RS` | Define o nível de consistência e bloqueio das leituras. |
| `RELEASE`  | `COMMIT`, `DEALLOCATE`  | Define quando os recursos alocados (como locks) serão liberados. |

**Detalhes sobre ISOLATION:**

| Código | Significado | Uso comum |
|--------|-------------|-----------|
| `CS`   | Cursor Stability: bloqueia a linha apenas enquanto o cursor está nela | Mais usado |
| `RR`   | Repeatable Read: mantém lock até o final da transação | Alta consistência |
| `UR`   | Uncommitted Read: não bloqueia nada; pode ler dados sujos | Para consultas sem impacto |
| `RS`   | Read Stability: meio-termo entre CS e RR | Garantia de não ver inserções repetidas |

**Detalhes sobre RELEASE:**

- `COMMIT`: libera os recursos a cada COMMIT — padrão, mais seguro.
- `DEALLOCATE`: mantém recursos até o final da thread — mais performático, usado em long-running tasks ou CICS.

---

#### 🔍 `EXPLAIN`

- `EXPLAIN(YES)` instrui o DB2 a gerar uma linha na `PLAN_TABLE` com detalhes do plano de acesso.
- Permite ao DBA analisar a escolha de índice, tipo de join, ordem de acesso e estatísticas envolvidas.

> ✅ Essencial após RUNSTATS, mudanças de índice ou REBINDs críticos.

---

#### ⚙️ `ACQUIRE`

| Valor | Explicação |
|-------|------------|
| `USE` | Aloca locks apenas conforme a necessidade de execução. É o padrão. |
| `ALLOCATE` | Aloca todos os locks no início da thread — usado com `RELEASE DEALLOCATE` para ganho de performance. |

> 📌 `ACQUIRE(ALLOCATE)` só deve ser usado se **todos os objetos necessários estiverem sempre acessíveis**, pois ele reserva recursos antecipadamente.

---

#### 📅 `APPLCOMPAT`

- Define o nível de compatibilidade do SQL em relação à versão do DB2.
- Afeta funções, tipos de dados, comportamento de CASTs, regras de GROUP BY, entre outros.

| Valor           | Versão base     | Situação comum de uso |
|------------------|------------------|------------------------|
| `V12R1M500`       | Versão inicial do DB2 12 | Manter compatibilidade com sistema antigo |
| `V12R1M510`       | Versão aprimorada com novos recursos | Recomendado após ajustes/testes |
| `V13R1M501`       | Compatível com novas funções DB2 13 | Exige validação rigorosa |

> 🧪 Antes de alterar `APPLCOMPAT`, avalie via `EXPLAIN` se o plano será alterado. Testes são fundamentais para evitar regressão de performance ou sintaxe inválida.

---

### 📌 Exemplo prático completo com todos os parâmetros

```sql
BIND PACKAGE('FATURAMENTO') MEMBER('REL_MENSAL')
    QUALIFIER('CORP')
    OWNER('DBA_USUARIO')
    VALIDATE(BIND)
    ISOLATION(CS)
    RELEASE(COMMIT)
    EXPLAIN(YES)
    ACQUIRE(USE)
    APPLCOMPAT(V12R1M510);
```

---

> 🎓 Dominar os parâmetros do BIND permite ao DBA atuar com segurança e previsibilidade nos ambientes mais críticos. Escolhas bem-feitas aqui evitam falhas em produção e garantem desempenho consistente.


---

## 5. Quando Atualizar o BIND

### 🎯 Objetivo:
Identificar com precisão os cenários que exigem a atualização do BIND (via `BIND` ou `REBIND`), a fim de garantir consistência, integridade e desempenho na execução de SQL no ambiente DB2 for z/OS.

---

### 🔄 O que significa "atualizar o BIND"?

Atualizar o BIND implica em **recompilar o plano de acesso do DB2** com base no contexto mais recente:
- Estrutura de tabelas e índices
- Estatísticas atualizadas
- Versões de SQL compatíveis
- Políticas de bloqueio e acesso
- Permissões vigentes

A atualização pode ser feita com:
- `BIND PACKAGE`: para novos pacotes ou recompilação com novo DBRM
- `REBIND PACKAGE`: para reutilizar o mesmo DBRM com novo plano de acesso

---

### 📋 Situações típicas que exigem atualização do BIND

| Tipo de Mudança                | Ação Necessária      | Explicação Técnica                                                                 |
|-------------------------------|----------------------|-------------------------------------------------------------------------------------|
| `ALTER TABLE`, `ADD COLUMN`, `DROP COLUMN` | `REBIND` obrigatório | O plano de acesso se torna inválido ou obsoleto. SQLCODE -818 pode ocorrer.         |
| `CREATE/DROP/ALTER INDEX`     | `REBIND` recomendado | O otimizador pode adotar um plano de acesso melhor com base no novo índice.         |
| `RUNSTATS` em tabelas/indexes | `REBIND` desejável   | Para o otimizador refletir as estatísticas atualizadas e melhorar a escolha de caminho. |
| `ALTER VIEW`, `ALTER SYNONYM`, `ALTER TRIGGER` | `REBIND` necessário | Dependências do package podem mudar — impacta resolução de nomes e permissões.     |
| Mudança no parâmetro `ISOLATION`, `RELEASE`, etc. | Novo `BIND`         | Os parâmetros influenciam diretamente o controle de concorrência e locks.           |
| Alteração em `APPLCOMPAT` (versão de SQL) | Novo `BIND` ou `REBIND` | Garante que novas funcionalidades SQL sejam habilitadas. Evita comportamento obsoleto. |
| Upgrade de versão do DB2      | `REBIND` recomendado | Revalida os planos e permite uso de novos recursos e otimizações internas.         |
| Revogação ou concessão de `GRANT` em objetos SQL | `REBIND` ou `VALIDATE(BIND)` | Garante que permissões sejam reavaliadas. Sem isso, falhas podem ocorrer no runtime. |
| `COPY PACKAGE`, `FREE PACKAGE`, `DROP/REBIND PLAN` | REBIND direto        | Necessário reconstituir pacotes para execução correta.                              |
| Modificação de lógica no programa e recompilação | Novo `BIND`         | Um novo DBRM exige um novo BIND para ser executado.                                 |

---

### ⚠️ Riscos de não atualizar o BIND

| Risco potencial                     | Consequência prática                                                                 |
|-------------------------------------|---------------------------------------------------------------------------------------|
| Plano desatualizado                 | O otimizador pode usar estratégia ruim → degradação de performance                   |
| DBRM e Package fora de sincronia    | Erro `SQLCODE -818`: timestamp do executável não bate com o package                  |
| Pacote inválido (`VALID = 'N'`)     | Erro `DSNT201I`, `-805` ou falha silenciosa na execução                              |
| Uso de estatísticas defasadas       | Escolhas ruins de join, scan completo, alto custo de GETPAGES                        |
| Incompatibilidade com nova versão   | SQL que antes funcionava pode quebrar com `APPLCOMPAT` desatualizado (`SQLCODE -4743`) |

---

### 🛠️ Boas práticas para manter o BIND atualizado

1. ✅ **Após cada `RUNSTATS`, agende `REBIND` dos pacotes impactados**:
   - Use consultas à `SYSPACKDEP` para descobrir dependências por tabela.

2. ✅ **Mantenha rotina de `REBIND` periódico em produção**:
   - Pode ser mensal, trimestral ou alinhado a ciclos de deploy/upgrade.

3. ✅ **Automatize validação de pacotes com `VALID = 'N'`**:
   ```sql
   SELECT COLLID, NAME, VALID FROM SYSIBM.SYSPACKAGE WHERE VALID = 'N';
   ```

4. ✅ **Após alterações de estrutura (DDL), rebinder antes da execução**:
   - Evita falhas inesperadas em runtime.

5. ✅ **Utilize `COPY PACKAGE` antes de REBIND crítico**:
   - Permite rollback seguro:
     ```sql
     COPY PACKAGE(COLID.PROGRAMA) COPYID('BKP_BEFORE_REBIND');
     ```

6. ✅ **Monitore SQLCODEs relacionados a pacotes inválidos**:
   - Principais: `-805`, `-818`, `-818`, `-922`, `DSNT201I`, `-4743`

---

### 🧪 Exemplo de cenário que exige REBIND

**Situação:**  
Equipe de modelagem executou `RUNSTATS` após reindexação de uma tabela crítica de faturamento. O programa `CALC_FATURAMENTO` passou a ter picos de CPU.

**Ação recomendada:**

```sql
REBIND PACKAGE(FATURAMENTO.CALC_FATURAMENTO) 
    EXPLAIN(YES) 
    APPLCOMPAT(V12R1M510) 
    VALIDATE(BIND);
```

**Resultado esperado:**  
Novo plano baseado em estatísticas atuais → queda de custo de acesso, menor tempo de CPU e I/O.

---

> 🎓 Saber **quando atualizar o BIND** é tão importante quanto saber executá-lo. O DBA proativo evita falhas em produção e garante performance estável com base na evolução do ambiente.

---

## 6. REBIND: Atualizando sem Recompilar

### 🎯 Objetivo:
Permitir a atualização do plano de acesso de um package existente **sem recompilar o programa** e **sem gerar um novo DBRM**.  
O comando `REBIND PACKAGE` força o otimizador do DB2 a regenerar o plano de execução baseado na **versão atual das estatísticas** e **estrutura dos objetos envolvidos (tabelas, índices, views, etc)**.

---

### ✅ Quando usar o REBIND?

Use o `REBIND PACKAGE` nos seguintes cenários:

| Situação                                                                 | Motivo técnico                                                                 |
|--------------------------------------------------------------------------|--------------------------------------------------------------------------------|
| 📊 Após execução de `RUNSTATS`                                          | Para refletir as estatísticas atualizadas no plano de acesso                   |
| 🏗️ Após `CREATE/DROP/ALTER INDEX` ou mudança de colunas                 | Para forçar o otimizador a reavaliar o uso de índices                          |
| ⚙️ Após instalação de `PTFs` ou atualizações de manutenção no DB2       | Algumas PTFs afetam diretamente o otimizador ou o interpretador de SQL        |
| ⏱️ Para resolver desempenho degradado (e.g. aumento de GETPAGES)        | Um novo plano pode reduzir custo de acesso                                     |
| 🔁 Para recuperar pacotes inválidos (`VALID = 'N'` no catálogo)         | Pacotes inválidos **só voltam a ser válidos via REBIND**                       |
| 🧪 Ao mudar a política de compatibilidade via `APPLCOMPAT`              | Para aplicar nova versão de regras SQL, funções e comportamento do otimizador |

---

### 📘 Sintaxe recomendada

```sql
REBIND PACKAGE('COLECAO') 
    MEMBER('PROGRAMA') 
    EXPLAIN(YES) 
    VALIDATE(BIND) 
    APPLCOMPAT(V12R1M510)
```

---

### 🧩 Explicação de cada parâmetro

| Parâmetro            | Finalidade                                                                 |
|----------------------|---------------------------------------------------------------------------|
| `PACKAGE('COLECAO')` | Indica a **collection** onde o pacote foi originalmente bindado           |
| `MEMBER('PROGRAMA')` | Nome do programa/fonte utilizado no bind                                  |
| `EXPLAIN(YES)`       | Gera informações de plano de acesso na `PLAN_TABLE`                       |
| `VALIDATE(BIND)`     | Valida todas as dependências no momento do REBIND (evita surpresa em runtime) |
| `APPLCOMPAT(V12R1M510)` | Define nível de compatibilidade SQL a ser utilizado (ex: novas funções, regras de casting) |

> 💡 Dica: você pode incluir `REOPT(ALWAYS)` no REBIND para otimização dinâmica baseada em parâmetros reais de entrada em tempo de execução.

---

### 🔎 Exemplo prático com foco em performance

```sql
REBIND PACKAGE('FATURAMENTO') 
    MEMBER('CALCULO_MENSAL') 
    EXPLAIN(YES) 
    VALIDATE(BIND) 
    APPLCOMPAT(V12R1M510) 
    REOPT(ALWAYS)
```

Neste exemplo:
- O pacote do programa `CALCULO_MENSAL` será reavaliado com base nos índices e estatísticas mais recentes.
- O plano será gravado na `PLAN_TABLE`.
- A estratégia de acesso poderá mudar de TABLE SCAN para INDEX MATCHING, reduzindo o custo total da query.

---

### ⚠️ Cuidados importantes

- **Não use REBIND cegamente em produção**: avalie o impacto via `EXPLAIN`.
- **Sempre salve uma versão anterior com `COPY PACKAGE`**, antes de rebinder:
  ```sql
  COPY PACKAGE(FATURAMENTO.CALCULO_MENSAL) COPYID('ANTES_REBIND');
  ```
- O REBIND pode gerar plano mais lento se estatísticas estiverem desatualizadas. Garanta que `RUNSTATS` foi executado antes.
- O REBIND é inofensivo para o código do programa — ele **não altera o executável**.

---

### 🔄 Alternativa: `REBIND TRIGGER PACKAGE` (a partir de V12R1M509)

Se você quiser rebinder todos os pacotes que foram invalidados por uma mudança de estrutura, pode usar:

```sql
REBIND TRIGGER PACKAGE;
```

Isso rebinda automaticamente todos os pacotes marcados como `VALID = 'N'`.

---

> ✅ Use o REBIND como uma ferramenta de controle fino de performance e estabilidade. Ele é uma das armas mais poderosas de um DBA experiente.



---

## 7. Boas Práticas em Ambientes Críticos

### 🎯 Objetivo:
Estabelecer diretrizes técnicas para uso seguro, eficiente e sustentável do BIND em ambientes de alta criticidade, como sistemas bancários, governamentais ou operacionais de missão crítica.

---

### ✅ Boas práticas recomendadas

| Prática | Justificativa Técnica |
|--------|------------------------|
| **Padronizar `QUALIFIER` e `OWNER` por sistema, ambiente e aplicação** | Garante uniformidade na resolução de nomes SQL e facilita a administração de permissões e troubleshooting por escopo lógico. |
| **Sempre utilizar `EXPLAIN(YES)`** | Permite auditoria, análise de performance e rastreio de mudanças de plano de execução. Essencial após REBIND ou RUNSTATS. |
| **Usar `RELEASE(DEALLOCATE)` em programas com threads persistentes (ex: CICS)** | Melhora performance ao manter recursos alocados entre transações. Requer cautela com consumo de locks e buffers. |
| **Controlar permissões de `BIND`/`REBIND` via perfis (`ROLES`) e autoridade DB2 (`DBADM`, `BINDADD`)** | Reduz risco de operações críticas indevidas. Use RBAC para delegar responsabilidades com segurança. |
| **Criar políticas de versionamento usando `COLLECTION` por release ou ambiente (`COL_APP01_REL04`)** | Facilita rollback, coexistência de versões e deploy seguro em produção. Também auxilia auditorias e controle de ciclo de vida. |
| **Usar `VALIDATE(BIND)` em ambientes de homologação e produção** | Garante integridade no momento do bind. Evita falhas em runtime por permissões faltantes ou objetos inexistentes. |
| **Monitorar `SYSPACKAGE.LASTUSED` e aplicar política de limpeza (`FREE PACKAGE`)** | Reduz carga no catálogo e melhora organização. Ideal para remover pacotes obsoletos após longos períodos de inatividade. |

---

### 🔐 Reforço de boas práticas de segurança e gestão

- 📌 **Evite `VALIDATE(RUN)` em produção**: pode mascarar erros de permissão ou dependências quebradas, só detectáveis em runtime.
- 📌 **Automatize auditoria de pacotes antigos**:
  ```sql
  SELECT COLLID, NAME, LASTUSED 
  FROM SYSIBM.SYSPACKAGE 
  WHERE LASTUSED < CURRENT DATE - 180 DAYS;
  ```
- 📌 **Mantenha histórico de binds críticos via `COPY PACKAGE`** com `COPYID` descritivo (ex: `RELEASE_2024Q1`).

---

> 🎓 Boas práticas não são apenas recomendações — são políticas que reduzem incidentes, facilitam diagnósticos e melhoram a governança do ciclo de vida dos pacotes em ambientes críticos.


---

## 8. Tabelas do Catálogo Relacionadas

### 🎯 Objetivo:
Listar e explicar as principais tabelas do catálogo (`SYSIBM`, `SYSIBM.SYS*`) relacionadas ao gerenciamento de BIND, REBIND, pacotes (`PACKAGES`) e planos (`PLANS`). Estas tabelas são essenciais para auditoria, troubleshooting, versionamento e automação da administração.

---

### 📋 Tabelas principais relacionadas a BIND e PACKAGES

| Tabela                     | Finalidade Técnica                                                                 |
|----------------------------|------------------------------------------------------------------------------------|
| **SYSIBM.SYSPACKAGE**      | Metadados dos packages: nome, collection, OWNER, BINDTS, ISOLATION, RELEASE, VALID, LASTUSED. É a tabela principal. |
| **SYSIBM.SYSPACKDEP**      | Dependências dos packages (tabelas, views, aliases, functions). Crucial para identificar o impacto de alterações estruturais. |
| **SYSIBM.SYSPACKAUTH**     | Registra quem tem autorização para executar (`EXECUTE`) ou rebinder (`BIND`) os packages. |
| **SYSIBM.SYSPACKSTMT**     | Detalha cada SQL compilado no package. Traz tipo de statement, número da instrução, flags de otimização. Útil para tuning. |
| **SYSIBM.SYSPACKCOPY**     | Armazena cópias (`COPY PACKAGE`) feitas com `COPYID`. Essencial para fallback e rollback de versões de pacote. |
| **SYSIBM.SYSPLAN**         | Armazena planos (`PLANs`) com referências a packages e configurações globais de execução. |
| **SYSIBM.SYSPACKLIST**     | Lista de packages associados a cada plano (`PKLIST`). Permite rastrear a composição de um PLAN. |
| **SYSIBM.SYSPACKAGE_HIST** | *(opcional, se habilitado)* — Histórico de alterações no SYSPACKAGE, em ambientes com trilha de auditoria avançada. |
| **SYSIBM.SYSPACKERR**      | Informações sobre erros de bind ou rebinder que foram capturados durante processos automatizados. |
| **SYSIBM.SYSDBRM**         | *(legado)* — Tabela usada em BINDs baseados em DBRM direto, ainda pode existir em sistemas antigos. |

---

### 🧠 Dicas práticas

- 🧪 Use `SYSPACKDEP` após alterações em tabelas para identificar todos os pacotes impactados que exigem `REBIND`.
- 📅 `LASTUSED` em `SYSPACKAGE` é **crucial** para detectar pacotes não utilizados há meses — úteis para limpeza.
- 🔐 Use `SYSPACKAUTH` para revisar permissões delegadas de REBIND, evitando exposição indevida.
- 🔍 `SYSPACKSTMT` é excelente para mapear SQLs críticos e identificar padrões problemáticos (ex: uso excessivo de tabelas temporárias, subqueries, etc).
- ♻️ `SYSPACKCOPY` viabiliza rollback com `COPYID` nomeado e seguro:
  ```sql
  COPY PACKAGE(MINHA_COL.PROG_X) COPYID('BEFORE_REBIND');
  ```

---

### ⚠️ Observações importantes

- Em ambientes com **controle de mudanças rígido**, monitore alterações no catálogo com triggers ou ferramentas de auditoria (quando suportado).
- Verifique se há **limpeza automatizada de pacotes obsoletos** — se não houver, crie rotinas baseadas em `LASTUSED`.

---

> 🎓 O catálogo do DB2 é a fonte de verdade do ambiente. Conhecê-lo profundamente empodera o DBA para diagnósticos rápidos, automações inteligentes e administração segura dos objetos de execução.

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

