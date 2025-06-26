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

# 2. Conceitos Básicos de Modelagem (Visão para DBAs)

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

# 3. Instalação e Configuração Inicial do PowerDesigner

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


