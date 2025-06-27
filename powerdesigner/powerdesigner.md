# Manual PowerDesigner para DBAs (Foco DB2 for z/OS)

## Índice

1. [Apresentação do PowerDesigner](#1-apresentação-do-powerdesigner)
2. [Conceitos Básicos de Modelagem](#2-conceitos-básicos-de-modelagem)
3. [Instalação e Configuração Inicial](#3-instalação-e-configuração-inicial)
4. [Abrindo um Modelo Físico Existente (PDM)](#4-abrindo-um-modelo-físico-existente-pdm)
5. [Visão Geral da Interface do PowerDesigner](#5-visão-geral-da-interface-do-powerdesigner)
6. [Analisando Tabelas DB2 no Modelo](#6-analisando-tabelas-db2-no-modelo)
7. [Adicionando Campos a Tabelas Existentes](#7-adicionando-campos-a-tabelas-existentes)
8. [Criando Índices (Indexes)](#8-criando-índices-indexes)
9. [Configurando Particionamento (Partitioning)](#9-configurando-particionamento-partitioning)
10. [Alterando Tipos de Dados](#10-alterando-tipos-de-dados)
11. [Visualizando Relacionamentos entre Tabelas](#11-visualizando-relacionamentos-entre-tabelas)
12. [Sincronização entre Modelo e Banco (Reverse/Forward Engineering)](#12-sincronização-entre-modelo-e-banco-reverseforward-engineering)
13. [Exportando Script SQL para DB2 z/OS](#13-exportando-script-sql-para-db2-zos)
14. [Boas Práticas para DBAs em Modelos PowerDesigner](#14-boas-práticas-para-dbas-em-modelos-powerdesigner)
15. [Glossário de Termos Técnicos do PowerDesigner](#15-glossário-de-termos-técnicos-do-powerdesigner)
16. [Referências Oficiais e Documentações](#16-referências-oficiais-e-documentações)

---

> ⚠️ Observação: Este manual não aborda técnicas de modelagem conceitual ou lógica. O foco é **manutenção, leitura e apoio à modelagem física DB2** em ambientes críticos.
>

---

# 1. Apresentação do PowerDesigner

## 1.1 O que é o PowerDesigner?

O **SAP PowerDesigner** é uma ferramenta corporativa de modelagem de dados e arquitetura de informação. Desenvolvido originalmente pela Sybase (adquirida pela SAP), ele é amplamente utilizado em grandes instituições para documentar, projetar e manter estruturas de dados complexas, como é o caso do **DB2 for z/OS** utilizado em bancos e órgãos governamentais.

> 🎯 Finalidade: Criar, visualizar, alterar e documentar modelos **conceituais, lógicos e físicos** de bancos de dados, além de gerar scripts SQL e metadados compatíveis com diversos SGBDs, incluindo **DB2 for z/OS**.

---

## 1.2 Por que o PowerDesigner é usado em ambientes como o da CEF?

- **Padronização**: Facilita o controle de qualidade e aderência a padrões internos de modelagem e nomenclatura.
- **Segurança**: Permite visualizar impactos antes de alterações em ambientes críticos.
- **Geração de scripts seguros**: O PowerDesigner pode gerar DDL específico para DB2 z/OS com compatibilidade com zParms e exigências da mainframe.
- **Integração com versionamento**: Modelos podem ser versionados, auditados e mantidos em repositórios corporativos.
- **Alinhamento entre áreas**: Facilita a comunicação entre áreas de desenvolvimento, arquitetura e produção, traduzindo conceitos técnicos em visões compreensíveis.

---

## 1.3 O papel do DBA no uso do PowerDesigner

O DBA pode usar o PowerDesigner para:

| Ação                          | Objetivo                                                                 |
|-------------------------------|--------------------------------------------------------------------------|
| Avaliar a estrutura física    | Verificar colunas, índices, constraints, partições, tabelas auxiliares. |
| Ajustar atributos             | Alterar tipos de dados, tamanhos, nomes de colunas, precisões.          |
| Criar índices                 | Analisar necessidade e criar índices adicionais.                        |
| Validar particionamentos      | Checar se as estratégias estão corretas para desempenho e manutenção.   |
| Verificar relacionamentos     | Entender integrações com tabelas externas (chaves estrangeiras).        |
| Gerar scripts DDL             | Exportar scripts SQL compatíveis com DB2 z/OS para aplicação em produção. |

> 🧠 Mesmo sem ser o modelador principal, o DBA tem papel **fundamental** na **validação da performance, consistência e viabilidade técnica** do modelo físico.

---

## 1.4 Tipos de modelos suportados

O PowerDesigner organiza os dados em três níveis de abstração principais:

| Nível             | Descrição                                                                                        | Exemplo prático para DBAs |
|-------------------|--------------------------------------------------------------------------------------------------|----------------------------|
| **CDM** (Conceitual) | Representação de entidades e relacionamentos sem detalhes técnicos.                             | Pouco usado por DBAs       |
| **LDM** (Lógico)     | Define tabelas, colunas, chaves e domínios sem vínculo direto com SGBD específico.              | Referência geral           |
| **PDM** (Físico)     | Define a implementação real no banco de dados, com nomes reais, tipos de dados, índices etc.   | 💡 Foco principal do DBA   |

> ✅ Para o DBA, o interesse está no **PDM (Physical Data Model)**, que representa como os dados estão realmente implementados no **DB2 for z/OS**.

---

## 1.5 Compatibilidade com DB2 for z/OS

O PowerDesigner possui **suporte nativo** para modelagem física de diversos SGBDs, incluindo:

- **IBM DB2 for z/OS**
- **DB2 LUW**
- **Oracle**
- **SQL Server**
- **Sybase ASE e IQ**
- **PostgreSQL**, entre outros

Ao criar ou importar um modelo para o DB2 for z/OS, o PowerDesigner:

- Adota **dialeto específico do DB2 z/OS** (ex: `VARCHAR(n) FOR BIT DATA`, `ORGANIZE BY ROW`, `PARTITION BY RANGE` etc).
- Permite definir **Storage Group**, **Buffer Pool**, **Tablespace** e **segurança de schema**.
- Inclui suporte a **partitioning**, **tablespace**, **índices únicos**, **restrições CHECK**, **naming conventions** etc.

---

## 1.6 O que é possível fazer no PDM (Modelo Físico)?

No PowerDesigner, o modelo físico permite visualizar e atuar sobre:

- **Tabelas (com campos, PK/FK, índices e constraints)**
- **Views e materialized views**
- **Stored Procedures e Triggers**
- **Domains e Defaults**
- **Partitioning (inclusive com range e hash)**
- **Security (schemas, ownership)**
- **Tablespaces, buffer pools e outras opções do DB2 z/OS**

---

## 1.7 Exemplo de Caso Real – CEF

> 🔎 Situação comum no ambiente da Caixa:

Um DBA precisa analisar se uma tabela de transações financeiras pode ter **um índice adicional** para melhorar a performance de uma query crítica. Ele acessa o **PDM no PowerDesigner**, localiza a tabela, identifica a ausência de um índice sobre os campos `DT_LANCTO` e `CD_AGENCIA`, e propõe a criação do índice, gerando o script DDL para análise de impacto.

---

## 1.8 Benefícios para o DBA

✔ Visualização clara da estrutura física sem depender de `SYSIBM.SYSTABLES` ou scripts manuais  
✔ Geração de scripts padronizados  
✔ Comparação entre versões de modelo  
✔ Garante conformidade com padrões internos  
✔ Permite reversão e auditoria

---

## 1.9 Conclusão

Mesmo não sendo uma ferramenta de uso exclusivo do DBA, o PowerDesigner é **indispensável em ambientes corporativos complexos**, como o da **CEF**, e o domínio básico sobre leitura e pequenas alterações no **modelo físico (PDM)** é fundamental para garantir **desempenho, integridade e aderência a padrões** no DB2 for z/OS.

---

## 1.10 Referências

- SAP PowerDesigner Product Documentation: [https://help.sap.com/viewer/product/POWERDESIGNER/](https://help.sap.com/viewer/product/POWERDESIGNER/)
- IBM DB2 for z/OS Documentation: [https://www.ibm.com/docs/en/db2-for-zos/](https://www.ibm.com/docs/en/db2-for-zos/)

---

# 2. Conceitos Básicos de Modelagem

## 2.1 Objetivo do Capítulo

Este capítulo tem por objetivo apresentar os principais **conceitos de modelagem de dados físicos** com foco no uso do **PowerDesigner** por DBAs que atuam em **ambientes corporativos críticos** utilizando **DB2 for z/OS**.

> 🧠 Mesmo sem atuar na modelagem em si, o DBA precisa entender a estrutura física representada no modelo para apoiar alterações, validar performance e aplicar boas práticas.

---

## 2.2 Tabela (Table)

### O que é?
Uma **tabela** representa um conjunto de registros armazenados no banco de dados. Cada linha corresponde a um registro (row), e cada coluna a um campo (column) com um tipo de dado específico.

### O que o DBA deve observar:
- Nome físico da tabela (ex: `TB_CLIENTE_MOV`)
- Tablespace associado
- Tipo de organização (ROW, COLUMN, PARTITIONED)
- Tamanho estimado
- Estatísticas associadas (ex: cardinalidade esperada)

---

## 2.3 Coluna (Column)

### O que é?
As **colunas** definem os atributos armazenados em uma tabela. Cada coluna possui:
- Nome físico (`CD_CLIENTE`)
- Tipo de dado (`CHAR(8)`, `INTEGER`, `DECIMAL(15,2)`)
- Precisão e escala (no caso de `DECIMAL`)
- Regra de obrigatoriedade (`NOT NULL`, `DEFAULT`)
- Domínio, se aplicável

### Importante:
- Colunas com nomes técnicos (sem acento, espaços ou caracteres especiais)
- Uso adequado de tipos compatíveis com o DB2 z/OS
- Evitar campos muito grandes (ex: `VARCHAR(4000)` sem necessidade)
- Observar campos com `DEFAULT`, que podem impactar processos de ETL ou replicação

---

## 2.4 Domínio (Domain)

### O que é?
Um **domínio** representa um tipo de dado reutilizável no modelo. Permite padronizar:
- Tipo de dado
- Tamanho
- Regra de validação
- Descrição semântica

### Exemplo:
Um domínio `DM_CODIGO_USUARIO` pode ser `CHAR(8)` com `UPPER CASE ONLY` aplicado em várias colunas.

> ✅ Domínios facilitam a padronização e a governança de dados.

---

## 2.5 Chave Primária (Primary Key)

### O que é?
A **chave primária** (PK) identifica unicamente cada linha de uma tabela.

### No PowerDesigner:
- A PK é marcada com uma chave dourada 🔑
- É composta por uma ou mais colunas
- Deve ser avaliada para:
  - Evitar uso excessivo de colunas grandes na PK
  - Garantir unicidade e performance nas buscas

### No DB2 z/OS:
- Uma tabela pode ter apenas **uma** chave primária
- A PK gera automaticamente um **índice único clustered**, salvo exceções

---

## 2.6 Chave Estrangeira (Foreign Key)

### O que é?
A **chave estrangeira** (FK) define uma relação entre duas tabelas, mantendo a integridade referencial.

### No PowerDesigner:
- Indicada por uma seta entre tabelas
- É criada a partir da PK da tabela pai
- O DBA deve validar:
  - Se as colunas envolvidas estão corretamente indexadas
  - Se há necessidade real da constraint física (em ambientes críticos, FKs nem sempre são implementadas fisicamente)

---

## 2.7 Índices (Indexes)

### O que é?
Um **índice** é uma estrutura auxiliar que acelera as operações de leitura.

### Tipos comuns:
- **Índice único (UNIQUE)**: garante unicidade
- **Índice não único (NONUNIQUE)**: melhora performance sem restrições
- **Índice composto**: envolve mais de uma coluna
- **Índice clustered** (DB2 z/OS): organiza fisicamente os dados

### O que observar:
- Campos mais acessados por cláusulas `WHERE`, `JOIN` e `ORDER BY`
- Custo de manutenção vs. ganho de leitura
- Overhead em tabelas com grande volume de inserções

---

## 2.8 Relacionamentos

### O que são?
São ligações lógicas entre entidades (tabelas), geralmente por chaves primárias e estrangeiras.

### Representação:
- Visualizadas como linhas entre tabelas no diagrama
- Permitem entender o fluxo de dados, integração e dependência entre objetos

### Para o DBA:
- Essencial para análise de impacto
- Apoia decisões sobre replicação, particionamento, e distribuição de dados
- Importante para modelar joins e estratégias de particionamento

---

## 2.9 Tabela de Referência x Tabela Fato

Embora não seja uma distinção obrigatória, no contexto corporativo:

| Tipo de Tabela     | Características                                          |
|--------------------|----------------------------------------------------------|
| **Tabela de Referência** | Tabela menor, com dados estáticos ou quase estáticos (`UF`, `TP_MOV`) |
| **Tabela Fato**         | Tabela volumosa, com eventos ou transações (`TB_LANCTO`, `TB_MOVTO_CAIXA`) |

> 🧠 Essa distinção ajuda o DBA a priorizar análise de performance e particionamento nas tabelas com maior carga.

---

## 2.10 Conclusão

Compreender esses conceitos é essencial para o DBA navegar com segurança no PowerDesigner. Saber identificar uma tabela, uma coluna, um índice, ou uma FK corretamente permite realizar **alterações pontuais** com confiança e gerar scripts adequados ao ambiente DB2 z/OS.

A partir dos próximos capítulos, veremos **na prática** como abrir um modelo físico (PDM), identificar esses elementos e aplicar ajustes comuns.

---

## 2.11 Referências

- SAP PowerDesigner Help Portal: [https://help.sap.com/viewer/product/POWERDESIGNER/](https://help.sap.com/viewer/product/POWERDESIGNER/)
- IBM Documentation - DB2 for z/OS: [https://www.ibm.com/docs/en/db2-for-zos/](https://www.ibm.com/docs/en/db2-for-zos/)
- IBM Redbooks - DB2 Fundamentals

---

# 3. Instalação e Configuração Inicial

## 3.1 Objetivo do Capítulo

Este capítulo apresenta um passo a passo detalhado para instalação e configuração inicial do **SAP PowerDesigner**, com foco no uso do **modelo físico (PDM)** para **DB2 for z/OS**.

> 📌 Indicado para DBAs que **nunca usaram o PowerDesigner** e precisam começar pela base: instalar, configurar e abrir um projeto com segurança.

---

## 3.2 Pré-requisitos

Antes de iniciar, verifique os seguintes requisitos no seu ambiente:

| Requisito                     | Descrição                                               |
|------------------------------|---------------------------------------------------------|
| Sistema Operacional          | Windows 10 ou superior (64 bits)                        |
| Permissão de instalação      | Acesso de administrador na máquina local                |
| Software necessário          | PowerDesigner (versão 16.6 SP05 ou superior)            |
| Licença                      | Licença local ou por servidor (fornecida pela empresa)  |
| Conexão à internet (opcional)| Para acessar ajuda online e updates                     |

---

## 3.3 Onde baixar o PowerDesigner?

O PowerDesigner é um software proprietário da SAP. O download deve ser feito por meio do portal de software da empresa ou com apoio da área de arquitetura/suporte técnico da organização.

### Caso não tenha acesso:
Solicite à equipe responsável o **instalador da versão corporativa em uso no ambiente**, geralmente no formato:

---


> 🔐 Atenção: Não há versão gratuita ou trial pública. O uso é sempre licenciado.

---

## 3.4 Etapas da Instalação

### 1. Executar o instalador como administrador
Clique com o botão direito sobre o arquivo `.exe` e selecione **"Executar como administrador"**.

### 2. Aceitar termos da SAP
Marque a opção de aceitação da licença de uso.

### 3. Escolher tipo de instalação
Selecione:

- `Typical Installation` (recomendada para iniciantes)

### 4. Definir caminho de instalação
Use o padrão sugerido ou altere conforme políticas da empresa:

---


### 5. Finalizar e reiniciar
Conclua a instalação e reinicie o computador para garantir que todas as dependências sejam ativadas corretamente.

---

## 3.5 Primeira execução

Ao abrir o PowerDesigner pela primeira vez:

- Será solicitado o tipo de licença:
  - **Local License File**: apontar para o arquivo `.lic`
  - **License Server**: apontar para o servidor corporativo de licenças SAP

> 🧠 Caso não saiba qual utilizar, consulte a equipe de suporte de TI ou arquitetura de dados.

---

## 3.6 Ajuste de Idioma para Português (opcional)

Por padrão, o PowerDesigner é instalado em **inglês**. Para facilitar a leitura por novos usuários:

1. Acesse o menu `Tools` → `General Options`
2. Vá até a aba `General`
3. Em **Language**, selecione `Portuguese (Brazil)`
4. Reinicie o PowerDesigner

> 🌐 Observação: Algumas expressões técnicas podem continuar em inglês mesmo após a troca de idioma, pois são termos técnicos universais.

---

## 3.7 Ativando suporte ao DB2 for z/OS

O PowerDesigner suporta múltiplos bancos. Para trabalhar com DB2 for z/OS:

1. Menu: `File` → `New Model`
2. Selecione: **Physical Data Model (PDM)**
3. Na tela seguinte, selecione o **DBMS desejado**:

---


4. Clique em **OK**

> ✅ A partir daqui, todos os objetos criados ou analisados seguirão as regras sintáticas, estruturas e limitações do DB2 for z/OS, incluindo partições, buffer pools e naming conventions.

---

## 3.8 Configurações recomendadas para DBAs

Ajustes simples para tornar o ambiente mais prático para análise técnica:

| Ajuste                                   | Caminho                                          | Motivo                                                                 |
|------------------------------------------|--------------------------------------------------|------------------------------------------------------------------------|
| Exibir nomes físicos nos diagramas       | `Tools` → `Display Preferences` → `Table`        | Permite ver nomes reais das tabelas e colunas usadas no banco         |
| Ativar exibição de constraints e índices | `Display Preferences` → `Constraints/Indexes`    | Facilita visualização de chaves, PK, FK, índices diretamente no diagrama |
| Mostrar tipo de dado                     | `Display Preferences` → `Column` → `Datatype`    | Útil para avaliar tamanhos, tipos e ajustes necessários                |
| Salvar em backup automático              | `Tools` → `General Options` → `Files`            | Garante segurança no trabalho em modelos grandes                      |

---

## 3.9 Criar pasta de trabalho local (boa prática)

Organize seus arquivos em uma pasta padrão como:

---

E salve cada modelo com nome claro e versão:

---

> 📁 Isso facilita controle de versões locais antes de envio ao repositório oficial (se houver).

---

## 3.10 Conclusão

Com essa configuração inicial completa, o DBA está apto a **abrir modelos existentes, visualizar objetos físicos e navegar pela estrutura com confiança**. Não é necessário ser modelador para compreender o conteúdo de um `.pdm` — basta conhecer os conceitos e saber onde olhar.

Nos próximos capítulos, veremos como **abrir modelos físicos prontos**, navegar entre objetos, localizar tabelas, criar índices e fazer pequenas alterações com segurança e domínio técnico.

---

## 3.11 Referências

- SAP PowerDesigner Installation Guide  
  [https://help.sap.com/docs/POWERDESIGNER](https://help.sap.com/docs/POWERDESIGNER)
- IBM DB2 for z/OS Documentation  
  [https://www.ibm.com/docs/en/db2-for-zos/](https://www.ibm.com/docs/en/db2-for-zos/)

---

# 4. Abrindo um Modelo Físico Existente (PDM)

## 4.1 Objetivo do Capítulo

Neste capítulo, você aprenderá como **abrir um modelo físico (.pdm)** no PowerDesigner e **navegar entre os principais objetos** do banco de dados, como tabelas, colunas, índices e relacionamentos — sem precisar ser modelador.

> 🎯 Ideal para DBAs que precisam avaliar tabelas do DB2, localizar objetos específicos, entender a estrutura de dados ou preparar ajustes simples como criação de índices ou campos.

---

## 4.2 O que é um arquivo `.pdm`?

O arquivo com extensão `.pdm` representa um **Physical Data Model** — ou modelo físico de dados. Nele estão armazenadas todas as definições da estrutura real de um banco de dados:

- Tabelas, colunas, domínios
- Índices, constraints, PK/FK
- Partições, views, procedures
- Estrutura de armazenamento (tablespaces, buffer pools)
- Comentários técnicos e semânticos

> 💡 Este é o **tipo de modelo que o DBA mais utiliza** para realizar validações técnicas, ajustes e gerar DDLs para aplicação no DB2 for z/OS.

---

## 4.3 Como abrir um modelo `.pdm`

### Passo a Passo:

1. Abra o PowerDesigner
2. Vá até o menu: `File` → `Open`
3. Navegue até o diretório onde está o modelo e selecione o arquivo `.pdm`

> 📁 Dica: crie uma estrutura de pastas organizada, como `C:\PowerDesigner\Modelos\DB2\`.

---

## 4.4 Tela principal do modelo aberto

Após abrir o arquivo `.pdm`, você verá uma **área central com o diagrama visual**, e uma **barra lateral esquerda chamada Browser**.

### Principais áreas:

| Área                   | Função                                                    |
|------------------------|-----------------------------------------------------------|
| **Diagram (centro)**   | Visualização gráfica das tabelas e relacionamentos        |
| **Browser (esquerda)** | Lista hierárquica de todos os objetos do modelo           |
| **Properties (baixo)** | Mostra os detalhes do objeto selecionado                  |

---

## 4.5 Navegando pelas tabelas

### Opção 1 – Pelo Diagrama
Clique sobre a tabela desejada no diagrama. Os campos e índices aparecem na parte inferior (aba `Properties`).

### Opção 2 – Pelo Browser
No painel esquerdo (Browser):
1. Expanda `Physical Data Model`
2. Expanda `Tables`
3. Clique com o botão direito sobre a tabela desejada e selecione `Properties`

> 📌 Use o campo de busca no topo do Browser para localizar rapidamente uma tabela pelo nome.

---

## 4.6 Visualizando as colunas de uma tabela

Ao abrir a janela de propriedades da tabela (`Properties`):

1. Vá até a aba `Columns`
2. Veja:
   - Nome da coluna
   - Tipo de dado (`INTEGER`, `CHAR(10)`, `DECIMAL(15,2)`, etc.)
   - Regras de obrigatoriedade (`NULL`, `NOT NULL`)
   - Valor padrão (`DEFAULT`)
   - Comentário semântico da coluna

> 🧠 Valide se os tipos de dados estão compatíveis com as boas práticas para DB2 for z/OS.

---

## 4.7 Verificando a chave primária

Na aba `Keys`:

1. Verifique se há uma chave marcada como **Primary**
2. Clique para ver quais colunas compõem a PK
3. Confirme se está indexada corretamente (a PK geralmente gera um índice automaticamente)

---

## 4.8 Avaliando os índices existentes

Aba `Indexes`:

1. Veja todos os índices definidos na tabela
2. Observe:
   - Nome do índice (`IX_TB_CLIENTE_01`)
   - Colunas utilizadas
   - Tipo: `Unique` ou `Non-Unique`
   - Ordem (`ASC`, `DESC`)
   - Clustered (DB2 usa `CLUSTER` em alguns casos)

> 🧩 Verifique se os índices atendem às principais consultas (`WHERE`, `JOIN`, `ORDER BY`).

---

## 4.9 Analisando relacionamentos (PK/FK)

No diagrama visual (ou aba `References` da tabela):

- Relacionamentos aparecem como **linhas entre tabelas**
- O PowerDesigner mostra se a relação é:
  - **Identifying (PK incorporada na FK)**
  - **Non-identifying (FK separada)**
- Clique na linha do relacionamento para ver:
  - Tabela de origem e destino
  - Colunas envolvidas
  - Regra de integridade (cascade, restrict, etc.)

---

## 4.10 Dica: Mostrar nomes físicos no diagrama

Por padrão, o PowerDesigner pode exibir **nomes lógicos** nos diagramas. Para mudar para **nomes reais (físicos)**:

1. Clique com o botão direito no diagrama
2. Selecione: `Display Preferences`
3. Em `Table`, marque a opção: **Show Physical Name**
4. Clique em OK

> 🔎 Isso facilita a vida do DBA, que trabalha com nomes reais no banco (ex: `TB_TRANSACAO_FINANCEIRA`).

---

## 4.11 Conclusão

Você já está apto a abrir modelos físicos no PowerDesigner, localizar e interpretar as tabelas, colunas, índices e relacionamentos — tudo o que um DBA precisa para analisar ou iniciar um ajuste pontual em ambientes críticos que utilizam DB2 for z/OS.

Nos próximos capítulos, você aprenderá como adicionar colunas, criar índices, avaliar particionamentos e gerar scripts DDL.

---

## 4.12 Referências (para este e capítulos anteriores)

```markdown
# Referências Técnicas Oficiais

- SAP PowerDesigner – Documentação Geral  
  https://help.sap.com/viewer/product/POWERDESIGNER/

- IBM Documentation – DB2 for z/OS  
  https://www.ibm.com/docs/en/db2-for-zos/

- IBM Redbooks – Data Modeling Techniques for DB2  
  https://www.redbooks.ibm.com/abstracts/sg247467.html

- SAP Community – PowerDesigner PDM Tips  
  https://community.sap.com/topics/powerdesigner

- SAP Note – Supported DBMS List  
  https://launchpad.support.sap.com/#/notes/1844496

---

# 5. Visão Geral da Interface do PowerDesigner

Neste capítulo, apresentamos uma visão completa da interface do PowerDesigner, com foco na navegação eficiente e nos recursos relevantes para a atuação técnica do DBA. O objetivo é garantir familiaridade com os principais painéis e menus, permitindo que qualquer profissional identifique rapidamente tabelas, colunas, índices, relacionamentos e configurações específicas do DB2 for z/OS.

---

#### Elementos principais da interface

Ao abrir um modelo PDM, a interface do PowerDesigner está dividida em áreas funcionais que interagem entre si:

| Área | Descrição |
|------|-----------|
| **Workspace (Centro)** | Área onde diagramas são exibidos visualmente. Cada aba corresponde a um modelo ou submodelo aberto. |
| **Object Browser (Esquerda)** | Estrutura hierárquica dos objetos do modelo. Permite navegação rápida por tabelas, índices, views, domínios, triggers, etc. |
| **Properties (Inferior ou janela flutuante)** | Exibe as propriedades do objeto selecionado: colunas, chaves, índices, tipo de dado, entre outros. |
| **Toolbox (Direita)** | Usado principalmente por modeladores para criar objetos (não será o foco do DBA). |
| **Output (Inferior, aba opcional)** | Mostra logs, mensagens de validação, geração de scripts, entre outros eventos. |

---

#### Menu principal — itens mais relevantes para DBAs

- `File`: abrir, salvar, exportar e importar modelos.
- `Edit`: localizar objetos, renomear, buscar dependências.
- `View`: habilitar/desabilitar painéis como o Object Browser, Output e Overview.
- `Model`: acessar propriedades do modelo, alterar DBMS, configurar opções.
- `Database`: menu exclusivo para geração de scripts SQL, engenharia reversa, comparação de modelos (veremos nos capítulos futuros).
- `Tools`: opções de layout, validação de modelo, formatação e preferências.

---

#### Navegação por painéis

##### 1. Object Browser (Painel esquerdo)

Permite expandir estruturas e navegar pelos principais componentes físicos:

- **Tables**: lista todas as tabelas do modelo.
- **Indexes**: permite ver todos os índices definidos (inclusive os que não estão em tabelas visíveis no diagrama).
- **References**: mostra os relacionamentos entre tabelas (FKs).
- **Views, Users, Domains, Procedures**: outros objetos que podem ser incluídos no modelo.

Use o campo de busca para localizar rapidamente um objeto pelo nome.

##### 2. Diagram Workspace (Centro)

- Mostra as entidades (tabelas) e os relacionamentos em forma gráfica.
- Permite movimentar objetos para melhor visualização.
- Útil para validar chaves estrangeiras e dependências.

##### 3. Properties (Janela flutuante ou inferior)

Clicando duas vezes em qualquer objeto (tabela, coluna, índice, relacionamento), será aberta a janela de propriedades correspondente.

Principais abas para o DBA:
- `Columns`: definição de colunas, tipos de dados, nullability, default.
- `Keys`: chaves primárias e alternativas.
- `Indexes`: índice físico associado.
- `Triggers` (se houver).
- `Rules and Checks`: constraints e regras aplicadas.

---

#### Personalizações recomendadas

##### Ativando nomes físicos no diagrama

1. Clique com o botão direito sobre uma área em branco do diagrama.
2. Selecione: `Display Preferences`.
3. Vá até a aba `Table`.
4. Marque:
   - “Show Physical Name”
   - “Show Column Data Types”
5. Clique em OK.

##### Ajustando a ordem das colunas exibidas

Por padrão, as colunas podem aparecer organizadas pela ordem lógica de criação. Para visualizar como no banco:

1. Abra a tabela.
2. Vá em `Columns`.
3. Use os botões de ordenação para reorganizar conforme a ordem física esperada.

---

#### Validações que podem ser feitas via interface

- Identificação de colunas sem tipo definido.
- Tabelas sem chave primária.
- Índices não associados a nenhuma constraint.
- Campos com nomes fora do padrão.
- Tabelas sem relacionamento com outras (tabelas órfãs).

Use:
```
Tools > Check Model
```
para gerar uma lista completa de problemas, alertas e sugestões.

---

#### Comandos rápidos para DBAs

| Ação | Caminho |
|------|---------|
| Localizar tabela | Ctrl + F |
| Exibir propriedades da tabela | Duplo clique ou botão direito > Properties |
| Reorganizar visualmente o diagrama | Tools > Layout > Auto Layout |
| Ver miniatura geral do modelo | View > Workspace > Overview |
| Gerar script DDL | Database > Generate Database... (veremos no Capítulo 13) |

---

#### Conclusão

A interface do PowerDesigner é altamente configurável, mas sua navegação pode ser dominada rapidamente por DBAs que se concentrem nos elementos corretos: Object Browser, Properties, Diagram e Model Options. Essa familiaridade tornará o trabalho de revisão, validação e preparação de alterações muito mais ágil e seguro.

No próximo capítulo, abordaremos a **análise detalhada das tabelas DB2 no modelo**, com foco em avaliação de colunas, constraints, nomes e estrutura.

---

#### Referências

- https://help.sap.com/viewer/product/SAP_POWERDESIGNER/16.7/en-US
- https://www.ibm.com/docs/en/db2-for-zos
- https://wiki.scn.sap.com/wiki/display/SYBPDS/PowerDesigner+User+Interface
- https://help.sap.com/docs/SAP_POWERDESIGNER/interactive-diagram-options

---

# 6. Analisando Tabelas DB2 no Modelo
[Voltar ao Índice](#índice)

Este capítulo apresenta as técnicas de inspeção, leitura e análise das tabelas DB2 for z/OS já existentes no modelo físico (PDM), garantindo melhor compreensão da estrutura atual antes de qualquer alteração.

---

#### 🔍 Acessando as Tabelas no Modelo

1. **Via Model Explorer**  
   - Expanda o nó **PhysicalDataModel_1** > **Tables**.  
   - Todas as tabelas estão listadas em ordem alfabética.
   - Clique com o botão direito em uma tabela > **Properties**.

2. **Via Diagrama**  
   - Dê **duplo clique** em qualquer tabela no espaço gráfico para abrir as propriedades.

---

#### 📑 Propriedades das Tabelas

Ao abrir as propriedades de uma tabela, observe as seguintes abas:

| Aba             | Conteúdo                                                                 |
|------------------|--------------------------------------------------------------------------|
| **General**      | Nome lógico e físico, owner, comentários.                                |
| **Columns**      | Lista de colunas, tipos de dados, PK, Not Null, Default, etc.            |
| **Keys**         | Definição de chaves primárias e alternativas.                            |
| **Indexes**      | Índices criados sobre a tabela, incluindo índice de clustering.          |
| **Triggers**     | Triggers associadas (caso utilizadas).                                   |
| **Check**        | Restrições CHECK.                                                        |
| **Rules**        | Regras aplicadas via domínio.                                            |

---

#### 🧠 Interpretação Técnica

- A presença de colunas `LAST_UPD_TS`, `INCLUSAO_USUARIO`, `DATA_ALTERACAO` pode indicar governança temporal.
- Tabelas sem PK definidas devem ser avaliadas — podem causar problemas em replicações e acessos.
- Avalie se a combinação de colunas da PK faz sentido com o negócio (ex: natural keys vs surrogate keys).
- Verifique o uso de `CHAR` fixo vs `VARCHAR` e os impactos de alocação no armazenamento.

---

#### 📎 Observações sobre Nomenclatura e Padrões

- Nomes de tabelas devem seguir convenções previamente definidas (ex: prefixos por domínio).
- Campos de auditoria e controle (usuário, timestamps) devem ter nomes padronizados.
- Tabelas técnicas ou de apoio (ex: parâmetros, conversões) devem estar identificadas.

---

#### ⚠️ Itens Críticos a Avaliar em Ambientes de Alta Confiabilidade

- Ausência de **índices** em tabelas com grande volume de leitura.
- **Check constraints** não utilizados em tabelas com dados sensíveis.
- **Campos obrigatórios** mal definidos (`Nullable = True` onde deveria ser `False`).
- Uso incorreto de **tipos numéricos**, como `DECIMAL` com precisão inadequada.
- Presença de colunas **reservadas para evolução futura**, mas sem uso definido.

---

#### 🛠️ Boas Práticas de Análise

- Utilize o botão **Preview DDL** para visualizar a geração do script da tabela.
- Registre observações e inconsistências diretamente nas propriedades (campo **Comment**).
- Utilize **Model Check** (menu Model > Check Model) para validar estrutura e integridade.

---

#### 📚 Referências e Leitura Complementar

- SAP PowerDesigner Help – Table Properties:  
  https://help.sap.com/viewer/product/SAP_POWERDESIGNER

---

# 7. Adicionando Campos a Tabelas Existentes
[Voltar ao Índice](#índice)

Este capítulo orienta como adicionar colunas (campos) a tabelas já existentes no modelo físico (PDM), com foco na análise técnica do impacto e nas práticas seguras para ambientes críticos que utilizam DB2 for z/OS. Adicionar uma coluna pode parecer simples, mas exige atenção ao tipo de dado, nullability, posicionamento e convenções do ambiente.

---

#### 🎯 Quando adicionar uma nova coluna?

- Inclusão de novos dados exigidos por regras de negócio.
- Evolução natural do modelo (novos processos, integrações).
- Criação de campos auxiliares para rastreio, controle ou auditoria.
- Atendimentos a requisitos de conformidade ou segurança.

---

#### 📌 Como adicionar um campo a uma tabela

1. **Abrir as propriedades da tabela**
   - Duplo clique sobre a tabela no diagrama, ou
   - Clicar com o botão direito no Model Explorer > `Properties`.

2. **Ir até a aba “Columns”**
   - Clique no botão **Add** (ícone de “+”).

3. **Preencher os dados da nova coluna**
   - **Name**: nome físico da coluna (ex: `CD_TIPO_CONTA`)
   - **Data Type**: selecione o tipo compatível com DB2 z/OS.
     - Exemplo: `CHAR(3)`, `DECIMAL(15,2)`, `DATE`, `TIMESTAMP`.
   - **Nullable**: defina se a coluna aceita valores nulos.
   - **Default Value**: (opcional) valor padrão para novos registros.

4. **(Opcional) Ajustar a ordem das colunas**
   - Use os botões de “Move Up / Move Down” para posicionar a nova coluna onde for mais adequado (embora no DB2 a ordem física não tenha impacto funcional, pode facilitar leitura).

5. **Salvar as alterações**
   - Clique em OK para confirmar a edição.

---

#### 🛑 Cuidados importantes

- **Campos NOT NULL exigem DEFAULT**: caso contrário, a criação do campo em uma tabela já populada pode falhar na geração do script SQL.
- **Evite colunas genéricas** como `OBS`, `DADO1`, `VALORX`. Use nomes descritivos e padronizados.
- **Evite campos multiuso** (um campo para mais de um significado). Isso compromete a integridade e legibilidade.

---

#### 🧠 Análise de impacto para DBAs

Antes de incluir um campo, avalie:

| Fator | Avaliação |
|-------|-----------|
| Volume de dados existente | Há impacto de espaço ou performance? |
| Índices existentes | O novo campo será indexado no futuro? |
| Procedimentos armazenados | Algum programa ou processo dependerá desse campo? |
| Views dependentes | Há visualizações que precisam ser ajustadas? |
| Carga inicial | A coluna deve vir preenchida para registros já existentes? |

---

#### 📘 Observações sobre DB2 for z/OS

- O PowerDesigner pode gerar **ALTER TABLE ADD COLUMN** no script final.
- Em versões DB2 mais antigas, a adição de colunas NOT NULL sem default não é permitida.
- Em tabelas particionadas, certifique-se de que o campo novo não interfere na lógica de particionamento.

---

#### 🧩 Convenções de nomenclatura (exemplos)

| Tipo de Dado       | Prefixo recomendado |
|--------------------|---------------------|
| Código numérico    | `CD_`               |
| Descrição textual  | `DS_`               |
| Datas              | `DT_`               |
| Flags e status     | `IN_` ou `ST_`      |
| Identificadores PK | `ID_`               |

---

#### ✅ Boas práticas ao adicionar colunas

- Registre a justificativa da inclusão no campo **Comment** da coluna.
- Marque visualmente a coluna com cor (Display Preferences) para revisão técnica.
- Gere o script SQL com a opção **Alter Statements** para simular a aplicação incremental.

---

#### 📚 Referências

- https://help.sap.com/viewer/product/SAP_POWERDESIGNER
- https://www.ibm.com/docs/en/db2-for-zos
- https://help.sap.com/docs/SAP_POWERDESIGNER/column-properties
- https://www.ibm.com/docs/en/db2-for-zos/12?topic=statements-alter-table

---

# 8. Criando Índices (Indexes)

#### Objetivo
Neste capítulo, abordaremos como criar índices no PowerDesigner, com foco em bancos de dados críticos, como DB2 for z/OS. Índices são estruturas fundamentais para garantir desempenho adequado nas consultas, especialmente em tabelas com grande volume de dados.

---

#### Conceito de Índices
Um **índice** é uma estrutura auxiliar de dados associada a uma ou mais colunas de uma tabela, usada para acelerar a recuperação de linhas. Em ambientes críticos, o uso adequado de índices pode reduzir significativamente o custo de acesso aos dados.

Existem vários tipos de índices, entre os mais comuns:

- **Índice primário (Primary Index)**: geralmente criado automaticamente para a **Primary Key**.
- **Índice único (Unique Index)**: garante que os valores em uma ou mais colunas sejam únicos.
- **Índice composto (Composite Index)**: abrange múltiplas colunas.
- **Índice de clusterização (Clustering Index)**: define a ordenação física dos dados no disco (no DB2 for z/OS, chamado de **CLUSTER**).
- **Índice de partição (Partitioned Index)**: usado em tabelas particionadas, respeitando regras específicas do banco.

---

#### Criando Índices no PowerDesigner

1. **Abrir o Modelo PDM**
   - Com seu modelo físico (PDM) aberto, selecione a tabela desejada.

2. **Criar um Novo Índice**
   - Clique com o botão direito sobre a tabela > `New > Index`.
   - Ou, vá até a aba **Indexes** dentro das propriedades da tabela.

3. **Definir o Nome e Tipo do Índice**
   - No campo `Name`, defina um nome padronizado (ex: `IX_CLIENTE_NOME`).
   - Marque `Unique` se o índice for único.
   - Marque `Cluster` para indicar que é um índice de clusterização.

4. **Selecionar as Colunas**
   - Na aba `Columns`, clique em `Add...` para incluir uma ou mais colunas.
   - Defina a **ordem** e se será em ordem ascendente ou descendente (se suportado).

5. **Configurar Propriedades Avançadas**
   - Na aba `General`, é possível incluir:
     - Comentários descritivos
     - Propriedades específicas do SGBD (usando `DBMS Properties` se necessário)

---

#### Considerações Importantes

- Em ambientes críticos, **evite criar índices desnecessários**, pois eles impactam diretamente nas operações de inserção e atualização.
- Analise os **plano de acesso (Access Path)** periodicamente para decidir sobre a criação, remoção ou ajuste de índices.
- Em tabelas muito acessadas com filtros por múltiplas colunas, **índices compostos** podem ser mais eficazes.
- **Índices exclusivos** devem refletir restrições reais de negócio.

---

#### Exemplo Ilustrativo

Vamos considerar a criação de um índice composto para a tabela `CLIENTES` com as colunas `NOME` e `CIDADE`:

- Nome do índice: `IX_CLIENTES_NOME_CIDADE`
- Tipo: Não exclusivo
- Cluster: Não

```sql
CREATE INDEX IX_CLIENTES_NOME_CIDADE 
ON CLIENTES (NOME ASC, CIDADE ASC);
```

---

#### Validação e Geração do Script

Após criar o índice:

1. **Valide o modelo** clicando em `Tools > Check Model`.
2. **Gere o script SQL** clicando em `Database > Generate Database...`, certificando-se de marcar a opção `Indexes`.

---

#### Recomendações de Padronização

- Prefixos: `IX_` para índices não exclusivos, `UX_` para exclusivos.
- Nomes descritivos, preferencialmente com até 30 caracteres.
- Alinhamento com convenções adotadas na organização.

---

#### Links Úteis

```markdown
- [Modelagem Física no PowerDesigner – IBM DB2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos)
- [SQL Reference – DB2 z/OS Índices](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=reference-sql-statements)
- [Documentação PowerDesigner Oficial – Índices](https://doc.sap.com/documents/sap?current=sap-powerdesigner)
- [DB2 Performance Index Guidelines (IBM)](https://www.ibm.com/docs/en/db2-for-zos/latest?topic=indexes-guidelines-creating)
```


---

# 9. Configurando Particionamento (Partitioning)

A técnica de particionamento é essencial em ambientes críticos e de grande volume de dados, como os encontrados em bancos corporativos. No PowerDesigner, é possível representar a estrutura de particionamento no modelo físico (PDM), permitindo uma visão clara da estratégia de distribuição de dados adotada no banco de dados.

---

### 📌 O que é Particionamento?

Particionamento é o processo de dividir fisicamente uma tabela ou índice em partes menores chamadas de partições, que podem ser armazenadas em diferentes espaços de armazenamento (tablespaces). Isso traz benefícios como:

- Melhor desempenho nas consultas, principalmente quando as partições são acessadas de forma seletiva.
- Redução de contenção de I/O.
- Facilidade de manutenção, como exclusão ou carregamento de dados por partição.

---

### 🎯 Tipos de Particionamento no DB2 for z/OS

Os principais tipos de particionamento suportados pelo DB2 e que podem ser modelados no PowerDesigner são:

- **Particionamento por Intervalo (Range Partitioning):** divide os dados com base em intervalos de valores de uma coluna (ex: datas).
- **Particionamento por Lista (List Partitioning):** divide os dados com base em valores discretos de uma coluna.
- **Particionamento por Hash:** baseado em funções de hashing.
- **Particionamento Composto (Composite Partitioning):** combinação de dois tipos, como intervalo e hash.

---

### 🧩 Representando Particionamento no PowerDesigner

#### Etapas para configurar:

1. **Abra o modelo físico (PDM).**
2. Selecione a tabela desejada.
3. Clique com o botão direito > **Properties**.
4. Acesse a aba `Partition` (ou `Storage` se estiver em modo simplificado).
5. Habilite a opção **Partitioned Table**.
6. Escolha o tipo de particionamento (Range, List etc.).
7. Configure as colunas de particionamento e os valores ou regras.

> 💡 **Dica**: para DB2 z/OS, o particionamento é geralmente feito via `PARTITION BY RANGE(...)` associado a tablespaces particionadas.

---

### 🛠️ Exemplo Prático

Suponha uma tabela de movimentações financeiras (`MOVIMENTACOES`) particionada por ano:

```sql
CREATE TABLE MOVIMENTACOES (
    ID_MOV INT NOT NULL,
    ANO INT NOT NULL,
    VALOR DECIMAL(15,2),
    PRIMARY KEY (ID_MOV)
)
PARTITION BY RANGE (ANO) (
    PARTITION P_2022 VALUES LESS THAN (2023),
    PARTITION P_2023 VALUES LESS THAN (2024),
    PARTITION P_MAX  VALUES LESS THAN (MAXVALUE)
);
```

No PowerDesigner, a estrutura acima pode ser representada criando uma tabela com `Partition Strategy: Range` e definindo `ANO` como a coluna de particionamento, com os respectivos valores limites.

---

### 🎯 Considerações Importantes

- **Chave Primária**: deve conter a coluna de particionamento.
- **Índices**: podem ser locais (por partição) ou globais (cobrindo toda a tabela).
- **Constraints**: verifique a compatibilidade com o particionamento.
- **Limitações**: nem todos os tipos de particionamento são implementáveis em todas as versões do DB2 z/OS — consulte a documentação oficial.

---

### 🧠 Boas Práticas

- Sempre documente no modelo os critérios de particionamento.
- Avalie o volume de dados e os padrões de acesso para escolher o tipo ideal.
- Verifique a possibilidade de manutenção isolada por partição.
- Use **nomenclatura padrão** nas partições (ex: `P_2024`, `P_MAX`) para facilitar manutenção e leitura.

---

### 📚 Referências

```markdown
- IBM Documentation - DB2 for z/OS Partitioning: https://www.ibm.com/docs/en/db2-for-zos/13?topic=databases-table-partitioning
- IBM - Best Practices for Table Design: https://www.ibm.com/docs/en/db2-for-zos/13?topic=design-best-practices-table
- IBM - CREATE TABLE (DB2 13): https://www.ibm.com/docs/en/db2-for-zos/13?topic=statements-create-table
- PowerDesigner Help - Partitioning Tables: https://www.sap.com/documents/2019/11/3f3fdb6e-c77d-0010-87a3-c30de2ffd8ff.html
```

---

# 10. Alterando Tipos de Dados

Alterar tipos de dados em um modelo físico (PDM) no PowerDesigner exige atenção redobrada, pois envolve tanto aspectos estruturais quanto implicações de compatibilidade com o ambiente de destino (como o DB2 for z/OS). Neste capítulo, abordaremos como realizar alterações seguras, mantendo a integridade do modelo e preparando-o corretamente para sincronização com o banco de dados.

---

### 📌 Objetivo do Capítulo

- Ensinar como localizar e alterar tipos de dados de colunas no modelo físico.
- Mostrar os impactos da alteração no PDM e como garantir conformidade com DB2 z/OS.
- Apontar boas práticas e armadilhas comuns.

---

### 🧭 Etapas para Alteração de Tipos de Dados

#### 1. Abrindo o Modelo Físico
Abra o modelo `.pdm` no PowerDesigner (se já não estiver aberto), conforme descrito no capítulo **4 - Abrindo um Modelo Físico Existente (PDM)**.

#### 2. Navegando até a Tabela Desejada
No painel esquerdo (Browser Tree):
- Expanda a seção **Physical Data Model → Tables**.
- Clique duas vezes na tabela desejada.

#### 3. Localizando o Campo a Ser Alterado
- Na aba **Columns**, localize a coluna cujo tipo de dado será alterado.
- Selecione a coluna e clique em **Properties** (botão ou clique com o botão direito > Properties).

#### 4. Alterando o Tipo de Dado
- Na janela de propriedades da coluna, localize o campo **Data Type**.
- Escolha o novo tipo de dado desejado.
- Se o tipo for do DBMS (ex: `CHAR`, `DECIMAL`, `DATE`, etc), selecione diretamente da lista.
- Em ambientes DB2 z/OS, utilize tipos compatíveis com o zSeries, como:
  - `CHAR(n)`
  - `VARCHAR(n)`
  - `DECIMAL(p,s)`
  - `INTEGER`
  - `BIGINT`
  - `DATE`
  - `TIMESTAMP`

#### 5. Salvando e Validando
- Clique em **OK** para aplicar a alteração.
- Salve o modelo (`Ctrl + S`).
- Use o menu **Model → Check Model** para validar o impacto da alteração. Corrija qualquer inconsistência apontada.

---

### ⚠️ Impactos e Cuidados

| Consideração Técnica                          | Descrição |
|----------------------------------------------|-----------|
| **Conversão de Dados no Banco**              | Alterar tipos de dados pode exigir conversão explícita (CAST) ao aplicar em ambiente real. |
| **Perda de Precisão ou Truncamento**         | Alterações para tipos menores ou de precisão reduzida devem ser avaliadas com cautela. |
| **Compatibilidade com Aplicações**           | Sistemas externos que consomem a base podem depender de tipos específicos. |
| **Script de Deploy**                         | O script SQL gerado (Forward Engineering) refletirá essa alteração. |
| **Sincronização (Compare Models)**           | Ao comparar modelos para atualizar o banco, o PowerDesigner detectará a alteração e sugerirá os comandos SQL necessários. |

---

### 🧪 Exemplo Prático

**Cenário:** Alterar a coluna `valor_total` de `INTEGER` para `DECIMAL(15,2)` na tabela `FATURAMENTO`.

**Passos:**
1. Navegue até a tabela `FATURAMENTO`.
2. Vá para a aba `Columns`.
3. Selecione `valor_total`, clique em `Properties`.
4. Altere `Data Type` de `INTEGER` para `DECIMAL(15,2)`.
5. Clique em `OK` e salve o modelo.
6. Valide com `Model → Check Model`.

---

### ✅ Boas Práticas

- Sempre valide o modelo após alterações.
- Documente a alteração (motivo, impacto, dependências).
- Padronize os tipos de dados para manter coerência no modelo.
- Em ambientes críticos, envolva times de desenvolvimento e operação antes de promover a alteração para produção.
- Utilize a funcionalidade **Change History** para rastrear alterações no modelo.

---

### 📚 Referências

```markdown
- [IBM DB2 for z/OS 13 - Data Types](https://www.ibm.com/docs/en/db2-for-zos/13?topic=columns-data-types)
- [PowerDesigner Help - Column Properties](https://doc.ispirer.com/sqlways/sybase-powerdesigner-column-properties)
- [Model Validation - PowerDesigner](https://www.sap.com/documents/2017/04/444402ba-497c-0010-82c7-eda71af511fa.html)
```

---

# 11. Visualizando Relacionamentos entre Tabelas


### Objetivo
Entender como o PowerDesigner representa e permite a análise dos relacionamentos entre tabelas no modelo físico (PDM), possibilitando que o DBA visualize a estrutura e navegue entre tabelas relacionadas com clareza.

### Conceito

No PowerDesigner, os relacionamentos entre tabelas (chaves estrangeiras, cardinalidades, dependências) são representados graficamente como linhas conectando entidades. Essas conexões permitem identificar as interações entre dados e são fundamentais para:
- Validar a integridade referencial.
- Analisar o impacto de alterações.
- Navegar no modelo com eficiência.

A correta visualização dos relacionamentos auxilia DBAs na identificação de gargalos, pontos de acoplamento forte e potencial de particionamento.

### Tipos de Relacionamentos Visualizados

- **Identificadores**: Relacionamentos que fazem parte da chave primária.
- **Não identificadores**: São chaves estrangeiras que não participam da chave primária.
- **Auto-relacionamentos**: Quando uma tabela se relaciona consigo mesma.
- **Relacionamentos com cardinalidade 1:N, N:1, e N:N**.

### Navegando pelos Relacionamentos

1. **Expandir relacionamentos**:
   - No painel de navegação à esquerda (Browser), expanda a seção `References` ou clique com o botão direito em uma tabela e selecione `Go to References`.

2. **Selecionar visualização gráfica**:
   - No menu superior, vá em **Display Preferences** > **Links**.
   - Ative ou ajuste a exibição de nomes de constraints, estilo das setas e cardinalidade.

3. **Usar Highlight**:
   - Clique em uma tabela, pressione `Ctrl` e clique em outra.
   - Use o botão direito e selecione **Highlight Related Tables** para destacar caminhos de dependência.

4. **Zoom e Pan**:
   - Utilize a ferramenta de lupa para ampliar regiões densas do modelo.
   - A ferramenta de `Pan` ajuda na navegação sem perder a referência visual.

### Interpretando os Elementos Visuais

- **Setas sólidas**: Indicam direção do relacionamento (de chave estrangeira para chave primária).
- **Chave sobre o campo**: Indica que aquele campo é parte da chave primária.
- **Losangos**: Indicam relacionamentos fracos, geralmente não identificadores.
- **Cardinalidade 1..1, 0..N, etc.**: Mostram quantos registros podem existir no lado relacionado.

### Sugestões para Ambientes Críticos

- Utilize **cores** para destacar relacionamentos críticos ou sensíveis à performance.
- Salve **views customizadas** do modelo com diferentes perspectivas: por subsistema, por prioridade, por grau de relacionamento.
- Ao revisar alterações, sempre reavalie os relacionamentos visuais impactados (Forward Engineering pode alterar ligações inadvertidamente).

### Dica Avançada: Gerando Lista de Relacionamentos

1. Menu `Tools` > `Reports`.
2. Escolha `List of References`.
3. Configure os campos: nome da tabela de origem, destino, tipo de relacionamento, etc.
4. Exporte em formato CSV ou HTML para uso externo.

---

### Links de Referência

```markdown
- [Documentação oficial do PowerDesigner - Model Relationships](https://www.sap.com/documents/2020/09/9460d77e-8d7b-0010-87a3-c30de2ffd8ff.html)
- [IBM DB2 for z/OS Concepts - Referential Integrity](https://www.ibm.com/docs/en/db2-for-zos/12?topic=concepts-referential-integrity)
- [SAP PowerDesigner Tips - Visualizing Table Relationships](https://community.sap.com/topics/powerdesigner)
- [Relational Modeling Best Practices (IBM Data Management)](https://www.ibm.com/docs/en/datastage/11.5?topic=guide-relational-database-design-best-practices)
```

---

# 12. Sincronização entre Modelo e Banco (Reverse/Forward Engineering)


A sincronização entre o modelo físico no PowerDesigner e o banco de dados real é um recurso essencial para ambientes críticos, garantindo que a documentação esteja alinhada com a realidade da implementação.

O PowerDesigner oferece dois mecanismos principais:

- **Reverse Engineering**: importar a estrutura do banco existente para gerar ou atualizar um modelo.
- **Forward Engineering**: gerar scripts SQL a partir do modelo para implementar alterações no banco de dados.

---

### ✅ Reverse Engineering (RE) – Importando o Banco para o Modelo

O objetivo do RE é obter um modelo físico a partir de um banco de dados já existente. Essa funcionalidade é útil para manter a documentação atualizada ou quando se deseja iniciar o trabalho de modelagem com base no ambiente atual.

**Passo a passo:**

1. **Abra o PowerDesigner.**
2. No menu principal, selecione `File > Reverse Engineer > Database…`
3. Na janela que se abrir, configure:
   - **DBMS**: selecione *IBM DB2 for z/OS*.
   - **Connection Profile**: configure ou selecione uma conexão ODBC válida.
   - **Scope**: selecione quais objetos deseja importar (tabelas, índices, views etc.).
4. Avance, revise as opções e conclua o processo.

> 💡 Ao final, o PowerDesigner criará um novo modelo físico (PDM) com os objetos existentes no banco de dados.

---

### ✅ Forward Engineering (FE) – Exportando Script do Modelo para o Banco

O FE permite gerar um script SQL contendo as alterações realizadas no modelo, para posterior aplicação no banco de dados.

**Passo a passo:**

1. Com o modelo físico aberto, acesse `Database > Generate Database…`
2. Na aba **Generation Options**, configure:
   - **Script Name**: defina o nome e caminho do script.
   - **Target DBMS**: selecione *IBM DB2 for z/OS*.
   - Marque **Generate DROP statements** se desejar incluir comandos para eliminar objetos existentes.
3. Use a aba **Preview** para revisar o script antes da exportação.
4. Clique em **OK** para gerar o arquivo.

> 💡 Scripts gerados podem ser aplicados com ferramentas como SPUFI, DSNTEP2, QMF ou via utilitários automatizados.

---

### ⚖️ Considerações Técnicas e Cuidados

| Situação | Ação Recomendada |
|---------|------------------|
| Alterações em produção | Validar e versionar scripts gerados. |
| Ambientes integrados (Dev/QA/Prod) | Manter controle de versão e checklist de deploy. |
| Campos alterados ou renomeados | Analisar impacto em procedures, views e aplicações. |
| Reverse após alterações manuais | Gerar novo modelo e comparar com versão anterior. |

---

### 📌 Dicas para Ambientes Críticos

- Evite aplicar **Forward Engineering direto em produção**. Sempre exporte o script e submeta à validação.
- Utilize ferramentas de comparação de modelos (PowerDesigner > Tools > Compare Models…) para revisar o que foi alterado.
- Configure logs de auditoria para rastrear mudanças no modelo ao longo do tempo.
- Sempre mantenha backup do modelo original antes de executar um RE ou FE.

---

# 13. Exportando Script SQL para DB2 z/OS

## Objetivo

Neste capítulo, você aprenderá a gerar scripts SQL a partir de um modelo físico no PowerDesigner, com foco específico em ambientes críticos que utilizam o DB2 for z/OS. A geração precisa do script é essencial para garantir consistência entre o modelo e a implementação no banco de dados.

## Contexto

Depois de realizar alterações em um Physical Data Model (PDM), é necessário refletir essas modificações no banco de dados. O PowerDesigner permite a exportação automatizada dos comandos SQL (Data Definition Language — DDL), alinhando o modelo às estruturas físicas do banco.

## Passos para gerar o script SQL

### 1. Abrir o modelo físico (PDM)

Certifique-se de estar com o modelo físico aberto e com todas as alterações devidamente salvas.

### 2. Configurar o DBMS alvo

1. No menu principal, acesse:  
   **Database > Edit Current DBMS…**
2. Confirme que o DBMS configurado é compatível com **DB2 for z/OS** (Ex: `DB2 UDB for z/OS 11.1`).
3. Ajuste as opções de geração SQL, se necessário, como por exemplo o uso de **STOGROUP**, **BUFFERPOOL**, **PARTITIONING**, etc.

> 🔧 Recomenda-se manter uma versão personalizada do DBMS, caso regras corporativas exijam sintaxes específicas ou extensões proprietárias.

### 3. Gerar o script SQL

1. Vá em:  
   **Database > Generate Database…**
2. Na janela que se abre:
   - Marque a opção **Generate Database**.
   - Defina o **Output File**, onde o script será salvo.
   - Certifique-se de que a opção **Generate DROP Statements** esteja configurada corretamente.
   - Marque **Check Model** para validar o modelo antes da geração.
3. Clique em **OK** para gerar o script.

### 4. Avaliar o script gerado

Revise manualmente o script gerado para validar:
- Uso correto de **schemas**.
- Definições de **tabelas, índices e constraints**.
- **Sintaxe compatível com o DB2 z/OS**, considerando o nível de compatibilidade da sua versão do DBMS.

### 5. Executar o script no ambiente de homologação

Nunca execute diretamente em produção. Antes:
- Teste integral do script em ambiente de homologação.
- Validação por pares (code review).
- Verificação de impacto nas estatísticas e objetos existentes.

## Considerações adicionais

- **Segmentação por objetos**: o PowerDesigner permite gerar scripts parciais, por tipo de objeto (somente tabelas, somente índices, etc.).
- **Agendamento e automação**: scripts gerados podem ser incorporados em pipelines de DevOps com controle de versão.
- **Geração incremental**: se você utilizou a funcionalidade de comparação (como vimos no capítulo anterior), é possível gerar apenas os deltas (diferenças).

---

# 14. Boas Práticas para DBAs em Modelos PowerDesigner

Neste capítulo, reunimos recomendações e boas práticas voltadas à atuação de DBAs em ambientes críticos que utilizam o PowerDesigner para modelagem física de banco de dados. As orientações aqui apresentadas visam garantir a consistência, integridade e facilidade de manutenção dos modelos ao longo do ciclo de vida do banco.

---

## 🛠️ Organização do Modelo

- **Use uma convenção de nomes padronizada** para entidades, colunas, índices e domínios. Ex: `TBL_CLIENTES`, `IDX_CLIENTES_CPF`.
- **Separe objetos por áreas funcionais** usando diagramas ou layouts temáticos dentro do PDM.
- Utilize **descrições completas nos objetos** (tabelas, colunas, índices), aproveitando os campos de “comment” disponíveis.
- Configure o **nome lógico** (Logical Name) e o **nome físico** (Code) de forma coerente.

---

## 🔍 Documentação e Anotações

- Utilize **Notes e Extended Notes** para registrar decisões de modelagem, regras de negócio e dependências.
- Inclua **anotações visuais** nos diagramas para destacar áreas críticas, status de validação ou sugestões futuras.

---

## 🧪 Validação e Revisão de Modelos

- Realize **validações automáticas** frequentes (`Tools > Check Model`) para identificar inconsistências ou objetos incompletos.
- Promova **revisões periódicas de modelo entre DBAs e analistas**, documentando os pontos discutidos.

---

## ♻️ Reutilização e Padronização

- Crie **Domain Types** (Domínios) reutilizáveis para padronizar tipos de dados (ex: `CPF_DOM`, `DATA_DOM`).
- Defina **naming standards templates** dentro do PowerDesigner e compartilhe com a equipe.

---

## 🔄 Integração com o Banco de Dados

- Use o recurso de **Reverse Engineering** com responsabilidade, validando o conteúdo trazido do banco.
- Realize o **Forward Engineering** com scripts controlados e versionados — nunca aplique diretamente em produção sem validação.

---

## 🗂️ Controle de Versões

- Armazene os arquivos `.pdm` em **sistemas de versionamento (ex: Git)**.
- Adote nomenclaturas consistentes para os arquivos: `modelo_fisico_v1.0.pdm`, `modelo_fisico_v1.1_rev.pdm`.
- Considere utilizar comentários nos commits com os principais ajustes realizados no modelo.

---

## 🔐 Segurança e Responsabilidades

- Proteja os arquivos de modelo com permissões adequadas.
- Restrinja alterações críticas apenas a DBAs ou modeladores responsáveis.
- Documente quem alterou o quê e por qual motivo.

---

## ✅ Checklist de Qualidade para DBAs

| Item                                                        | Verificado? |
|-------------------------------------------------------------|-------------|
| Convenção de nomes aplicada                                 | ☐           |
| Todos os objetos com comentários preenchidos                | ☐           |
| Domínios aplicados aos campos principais                    | ☐           |
| Validação automática do modelo sem erros                    | ☐           |
| Versão e data registradas na documentação                   | ☐           |
| Relacionamentos visualmente compreensíveis no diagrama      | ☐           |
| Script de Forward Engineering revisado                      | ☐           |

---

## 📘 Referências e Materiais de Apoio

```markdown
- Documentação oficial do PowerDesigner: https://support.sap.com/powerdesigner
- SAP PowerDesigner Best Practices Guide: https://help.sap.com/docs/powerdesigner
- Naming Standards (SAP Community): https://community.sap.com/topics/powerdesigner
- Guia de Engenharia Reversa e Sincronização: https://help.sap.com/viewer/product/POWERDESIGNER
