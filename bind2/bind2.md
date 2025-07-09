# Capítulo: Compreendendo o Processo de BIND no DB2 for z/OS

## Índice

- [Introdução](#introdução)  
- [O que é o BIND?](#o-que-é-o-bind)  
- [Processo de BIND no DB2 for z/OS](#processo-de-bind-no-db2-for-zos)  
  - [3.1 Pré-compilação](#31-pré-compilação)  
  - [3.2 BIND](#32-bind)  
  - [3.3 Execução](#33-execução)  
- [Tipos de BIND](#tipos-de-bind)  
  - [4.1 BIND](#41-bind)  
  - [4.2 REBIND](#42-rebind)  
- [Comandos BIND no DB2 for z/OS](#comandos-bind-no-db2-for-zos)  
- [Considerações Adicionais](#considerações-adicionais)  
- [Referências](#referências)  

---

## Introdução

Este capítulo explora o processo de BIND no DB2 for z/OS, detalhando suas etapas, tipos e comandos associados, com base em fontes oficiais da IBM.

---

## O que é o BIND?

No contexto do DB2 for z/OS, o BIND é o processo que associa um programa de aplicação a um banco de dados específico. Durante o BIND, o DB2 verifica a sintaxe do SQL embutido, valida a existência dos objetos referenciados e otimiza os planos de acesso para as instruções SQL. O resultado desse processo é a criação de pacotes que contêm os planos de acesso otimizados, permitindo que o programa interaja eficientemente com o banco de dados.

---

## Processo de BIND no DB2 for z/OS

### 3.1 Pré-compilação

Antes do BIND, o código-fonte do programa é pré-compilado. Durante essa etapa, o pré-compilador converte o SQL embutido em um formato intermediário, conhecido como DBRM (Database Request Module). O DBRM contém as instruções SQL em uma forma que pode ser processada pelo DB2 durante o BIND.

### 3.2 BIND

Após a pré-compilação, o DBRM é submetido ao processo de BIND. Durante o BIND:

- **Validação de Sintaxe**: O DB2 verifica a correção da sintaxe das instruções SQL presentes no DBRM.  
- **Verificação de Objetos**: O DB2 confirma a existência dos objetos de banco de dados referenciados, como tabelas e colunas.  
- **Otimização de Planos de Acesso**: O DB2 analisa diferentes estratégias para acessar os dados e seleciona a mais eficiente, considerando estatísticas e índices disponíveis.  
- **Criação de Pacotes**: O DB2 gera pacotes que contêm os planos de acesso otimizados, que serão utilizados durante a execução do programa.

### 3.3 Execução

Com os pacotes gerados pelo BIND, o programa pode ser executado. Durante a execução, o DB2 utiliza os planos de acesso contidos nos pacotes para acessar os dados de forma eficiente.

---

## Tipos de BIND

### 4.1 BIND

O comando BIND é utilizado para associar um programa a um banco de dados pela primeira vez ou quando há alterações significativas no programa ou no banco de dados. Durante o BIND, é possível especificar diversas opções que influenciam o comportamento da aplicação, como nível de isolamento, cache de autorização e protocolos de conexão.

O comando **`BIND PACKAGE ... ACTION(ADD|REPLACE)`** é usado para:  
- **ADD**: primeiro vínculo de um pacote.  
- **REPLACE**: substituir um pacote existente por novo DBRM ou pacote atualizado.  
As opções incluem isolamento, cache, CICS, proprietário etc.

### 4.2 REBIND

O comando REBIND é utilizado para reavaliar os planos de acesso de um pacote existente sem alterar o programa ou o banco de dados. Isso é útil quando há mudanças no banco de dados, como a adição de índices ou a execução de estatísticas, que podem afetar a eficiência dos planos de acesso. O REBIND permite que o DB2 atualize os planos de acesso para refletir essas mudanças, mantendo a eficiência da aplicação.

Utiliza-se **`REBIND PACKAGE`** quando apenas estatísticas foram atualizadas (ex.: novos índices, RUNSTATS). O REBIND:

- Regera o plano de acesso sem alterar SQL;
- Pode ser realizado com `PLANMGMT(EXTENDED)` para **phase-in**, permitindo que threads ativas continuem usando o pacote antigo enquanto o novo é carregado 

Rebind é mais eficiente que um novo bind, pois evita recompilar o SQL.

---

## Comandos BIND no DB2 for z/OS

Para realizar o BIND de um pacote no DB2 for z/OS, utiliza-se o seguinte comando:

```sql
BIND PACKAGE (nome_do_pacote)
     ACTION(ADD)
     MEMBER (nome_do_membro)
     QUALIFIER (qualificador)
     ISOLATION (CS)
     OWNER (proprietario)
     CACHESIZE (tamanho_cache)
     ENABLE (CICS)
     CICS (conexao_cics)
```

Onde:

- `ACTION(ADD)`: Indica que o pacote está sendo adicionado.  
- `MEMBER (nome_do_membro)`: Especifica o membro do DBRM a ser utilizado.  
- `QUALIFIER (qualificador)`: Define o qualificador para objetos não qualificados.  
- `ISOLATION (CS)`: Define o nível de isolamento como Cursor Stability.  
- `OWNER (proprietario)`: Define o proprietário do pacote.  
- `CACHESIZE (tamanho_cache)`: Define o tamanho do cache de autorização.  
- `ENABLE (CICS)`: Habilita o pacote para uso no CICS.  
- `CICS (conexao_cics)`: Especifica a conexão CICS a ser utilizada.  

Para mais detalhes sobre as opções do comando BIND, consulte a documentação oficial da IBM. (flylib.com)

### REBIND PACKAGE

```sql
REBIND PACKAGE (colecao.nome_pacote)
     PLANMGMT(EXTENDED)
     APREUSE(WARN | NONE | ERROR)
     ...
```

Use quando o ambiente muda (ex.: novos índices, estatísticas atualizadas)


---

## Considerações Adicionais

- **Procedimentos de Validação**: Se a tabela contiver procedimentos de validação, estes podem ser invalidados se os nomes das colunas excederem 18 bytes EBCDIC. (flylib.com)  
- **Uso de Acionadores (Triggers)**: Considere o uso de acionadores para substituir a funcionalidade em procedimentos de campo, procedimentos de edição definidos como `WITH ROW ATTRIBUTES` e procedimentos de saída de validação em tabelas com nomes de colunas maiores que 18 bytes EBCDIC. (flylib.com)

---

## Referências

1. [BIND PACKAGE subcommand (DSN)](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=commands-bind-package-dsn)  
2. [BIND and REBIND options for packages, plans, services](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=commands-bind-rebind-options-packages-plans-services)  
3. [REBIND PACKAGE subcommand (DSN)](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=commands-rebind-package-dsn)  
4. [Phase-in REBIND details (PLANMGMT)](https://www.ibm.com/docs/en/db2-for-zos/12.0.0?topic=commands-rebind-package-dsn#phase-in)
