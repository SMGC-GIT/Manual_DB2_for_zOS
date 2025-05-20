# 📁 Catálogo DB2 for z/OS

## 🧠 O que é o Catálogo DB2?

O catálogo é um conjunto de tabelas do sistema (DSNDB06) mantidas pelo DB2 for z/OS contendo metadados fundamentais: definições de objetos (tabelas, índices, planos), estatísticas de acesso, autorizações, entre outros.

---

## 🗃️ Principais Tabelas do Catálogo

| Tabela                      | Descrição                                                   |
|----------------------------|-------------------------------------------------------------|
| SYSIBM.SYSTABLES           | Contém todas as tabelas definidas no sistema                |
| SYSIBM.SYSCOLUMNS          | Informações sobre colunas de cada tabela                    |
| SYSIBM.SYSINDEXES          | Detalhes sobre índices existentes                           |
| SYSIBM.SYSKEYS             | Define chaves primárias e estrangeiras                      |
| SYSIBM.SYSCOPY             | Controle de cópias e backups realizados                     |
| SYSIBM.SYSROUTINES         | Lista procedures, functions e suas propriedades             |
| SYSIBM.SYSPACKAGE          | Informações sobre pacotes de programas compilados           |
| SYSIBM.SYSPLAN             | Planos de execução de aplicações                            |
| SYSIBM.SYSDATABASE         | Informações sobre os databases criados                      |

---

## 📌 Consultas Didáticas e Comentadas

```sql
-- Lista todas as tabelas de um schema
SELECT NAME, CREATOR, TYPE
FROM SYSIBM.SYSTABLES
WHERE CREATOR = 'MEUESQUEMA';
```
> 🎯 Mostra todas as tabelas do schema indicado.

---

```sql
-- Retorna colunas da tabela CLIENTES
SELECT NAME, COLTYPE, LENGTH, SCALE
FROM SYSIBM.SYSCOLUMNS
WHERE TBNAME = 'CLIENTES' AND TBCREATOR = 'MEUESQUEMA';
```
> 🔍 Exibe a estrutura da tabela CLIENTES (tipo, tamanho e escala).

---

```sql
-- Lista os índices definidos para a tabela CLIENTES
SELECT NAME, UNIQUERULE, COLCOUNT
FROM SYSIBM.SYSINDEXES
WHERE TBNAME = 'CLIENTES';
```
> 📌 Mostra os índices existentes e se são únicos.

---

```sql
-- Verifica quais colunas são chave primária ou estrangeira
SELECT TBNAME, NAME, COLNAME, KEYSEQ
FROM SYSIBM.SYSKEYS
WHERE TBNAME = 'CLIENTES';
```
> 🧷 Identifica as chaves definidas para a tabela.

---

```sql
-- Lista rotinas (procedures/functions) criadas por um schema
SELECT NAME, ROUTINETYPE, LANGUAGE, PARAMETER_STYLE
FROM SYSIBM.SYSROUTINES
WHERE SCHEMA = 'MEUESQUEMA';
```
> ⚙️ Permite explorar rotinas cadastradas.

---

```sql
-- Lista os pacotes de aplicação disponíveis
SELECT NAME, VALID, OPERATIVE, LASTUSED
FROM SYSIBM.SYSPACKAGE
WHERE COLLID = 'MEUCOLETOR';
```
> 📦 Pacotes compilados e quando foram usados pela última vez.

---

```sql
-- Lista planos de execução
SELECT NAME, CREATOR, VALID, OPERATIVE
FROM SYSIBM.SYSPLAN
WHERE CREATOR = 'MEUESQUEMA';
```
> 🗺️ Mostra os planos definidos para uso de aplicações batch, CICS etc.

---

```sql
-- Verifica cópias realizadas de tabelas
SELECT DBNAME, TSNAME, DSNAME, COPYTYPE, TIMESTAMP
FROM SYSIBM.SYSCOPY
WHERE DBNAME = 'MEUBANCO';
```
> 💾 Histórico de cópias de segurança (COPY, LOAD etc.).

---

## 🧠 Dica Avançada: Quantidade de linhas por tabela

```sql
SELECT CREATOR, NAME, CARDF
FROM SYSIBM.SYSTABLES
WHERE TYPE = 'T' AND CREATOR = 'MEUESQUEMA';
```
> 🔢 Coluna `CARDF` mostra a cardinalidade (nº estimado de linhas) com base nas estatísticas (RUNSTATS).

---

## ✅ Conclusão

Este módulo introduz o catálogo como fonte essencial para o DBA obter informações sobre o ambiente DB2 for z/OS. Recomenda-se o uso frequente destas queries como parte do arsenal de administração e diagnóstico.

---

## 📚 Para mais informações técnicas

Consulte os seguintes tópicos na documentação oficial da IBM:

- 🔹 [Visão geral do catálogo do DB2](https://www.ibm.com/docs/en/db2-for-zos/13?topic=system-db2-catalog)
  > Explica a função do catálogo e descreve sua estrutura geral.

- 🔹 [Tabelas do catálogo do DB2 for z/OS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=catalog-catalog-tables)
  > Lista completa das tabelas do catálogo com links individuais para cada uma.

- 🔹 [Tabela SYSTABLES](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-systables)
  > Detalhes sobre todas as colunas, chaves e restrições da tabela de metadados das tabelas.

- 🔹 [Tabela SYSCOLUMNS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-syscolumns)
  > Fornece informações de definição das colunas de tabelas.

- 🔹 [Tabela SYSINDEXES](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-sysindexes)
  > Informações sobre todos os índices definidos no sistema.

- 🔹 [Tabela SYSVIEWS](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-sysviews)
  > Armazena o SQL das views definidas no banco.

- 🔹 [Tabela SYSPACKAGE](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-syspackage)
  > Contém dados sobre os pacotes DBRM compilados no sistema.

- 🔹 [Tabela SYSPLAN](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tables-sysplan)
  > Informações sobre os planos de execução gerados e armazenados.

---

*Última atualização: 20/05/2025*

