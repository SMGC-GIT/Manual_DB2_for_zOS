# 🖥️ Manual TSO (Time Sharing Option) no z/OS - Comandos Básicos e Essenciais

Este manual foi criado para facilitar o aprendizado e uso dos comandos mais comuns do **TSO (Time Sharing Option)** em ambientes IBM **Mainframe z/OS**, com foco em usuários iniciantes e em **DBAs DB2 for z/OS**.

---

## 📚 Índice

- [1. Introdução ao TSO](#1-introdução-ao-tso)
- [2. Acessando o TSO](#2-acessando-o-tso)
- [3. Comandos Básicos do TSO](#3-comandos-básicos-do-tso)
  - [3.1. Comandos para manipulação de datasets](#31-comandos-para-manipulação-de-datasets)
  - [3.2. Comandos para listagem e navegação](#32-comandos-para-listagem-e-navegação)
  - [3.3. Comandos para execução de jobs](#33-comandos-para-execução-de-jobs)
  - [3.4. Comandos úteis para DBAs DB2](#34-comandos-úteis-para-dbas-db2)
- [4. ISPF e SDSF - Navegação Complementar](#4-ispf-e-sdsf---navegação-complementar)
- [5. Dicas Importantes](#5-dicas-importantes)
- [6. Referências Oficiais](#6-referências-oficiais)

---

## 1. Introdução ao TSO

O **TSO (Time Sharing Option)** permite que múltiplos usuários interajam com o sistema z/OS simultaneamente. Ele fornece uma interface de linha de comando ou menus ISPF para executar tarefas como edição de arquivos (datasets), submissão de jobs, execução de utilitários e comandos DB2.

Você pode usar o TSO:
- Via linha de comando TSO
- Através da interface ISPF (Interactive System Productivity Facility)

---

## 2. Acessando o TSO

Ao acessar o terminal 3270 (por exemplo, via **IBM Personal Communications**, **x3270**, **tn3270** etc.), após login com **usuário e senha**, você verá o menu ISPF. Pressione:

- `Option 6` → Para acessar o prompt de comandos TSO diretamente
- Ou digite `TSO` na tela inicial, se permitido

---

## 3. Comandos Básicos do TSO

A maioria dos comandos TSO pode ser usada diretamente via `Option 6` (linha de comando) ou embutida em CLISTs/REXX/scripts.

---

### 3.1. Comandos para manipulação de datasets

| Comando            | Função                                             |
|--------------------|----------------------------------------------------|
| `ALLOCATE`         | Aloca um dataset                                   |
| `FREE`             | Libera datasets previamente alocados               |
| `DELETE`           | Exclui dataset                                     |
| `RENAME`           | Renomeia dataset                                   |
| `LISTC`            | Lista catálogo de datasets                         |
| `COPY`             | Copia conteúdo de um dataset para outro            |

#### Exemplos:

```tso
ALLOCATE DS('USUARIO.TESTE.DATA') NEW SPACE(5,5) TRACKS RECFM(F B) LRECL(80) BLKSIZE(800) DSORG(PS)
FREE DDNAME(SYSUT1)
DELETE 'USUARIO.TESTE.DATA'
RENAME 'USUARIO.VELHO' 'USUARIO.NOVO'
LISTC LEVEL(USUARIO.TESTE)
COPY 'USUARIO.ARQ1' 'USUARIO.ARQ2'
```

---

### 3.2. Comandos para listagem e navegação

| Comando            | Função                                             |
|--------------------|----------------------------------------------------|
| `LISTC LEVEL(xxx)` | Lista datasets com prefixo                         |
| `DSLIST`           | Atalho para listagem de datasets via ISPF          |
| `ISRDDN`           | Lista os DDNAMEs ativos da sessão                  |
| `ISRFIND`          | Busca strings em datasets                          |

#### Exemplos:

```tso
LISTC LEVEL(DB2.SDSNEXIT)
DSLIST
ISRDDN
ISRFIND
```

---

### 3.3. Comandos para execução de jobs

| Comando            | Função                                             |
|--------------------|----------------------------------------------------|
| `SUBMIT`           | Submete um job JCL                                 |
| `STATUS`           | Mostra status de jobs submetidos                   |
| `OUTPUT`           | Visualiza saída do job                             |

#### Exemplos:

```tso
SUBMIT 'USUARIO.JCL(JOBTESTE)'
STATUS
OUTPUT JOBNAME(JOBTESTE)
```

---

### 3.4. Comandos úteis para DBAs DB2

| Comando TSO             | Função                                            |
|--------------------------|--------------------------------------------------|
| `SPUFI`                  | Interface para execução de comandos SQL          |
| `DB2I`                   | Inicia o DB2 Interactive                         |
| `DSN`                    | Chamada da CLI do DB2 para comandos dinâmicos    |
| `RUN PROGRAM(DSNTIAUL)`  | Executa programa utilitário DSNTIAUL             |

#### Exemplos:

```tso
TSO DB2I
```

Ou direto no ISPF → opção **DB2I/SPUFI**:

```tso
DSN SYSTEM(DSN1)
RUN PROGRAM(DSNTIAUL) PLAN(DSNTIAUL) PARMS('SQL')
END
```

---

## 4. ISPF e SDSF - Navegação Complementar

### ISPF (Option 3.4)

Permite navegar em datasets com opções:

- **B**: Browse
- **E**: Edit
- **V**: View

### SDSF (Option 5)

Permite visualizar **jobs no spool**, incluindo filtros por prefixo:

- **ST**: Status
- **DA**: Dados
- **SP**: Spool
- **H**: Held output

Comandos úteis no SDSF:

- `PRE jobname` → Filtra pelo nome do job
- `?` ou `S` → Visualiza saída
- `P` → Print
- `C` → Cancelar job

---

## 5. Dicas Importantes

- ⚠️ Sempre prefixe seus datasets com seu ID (ex: `SILVIA.PROC.JCL`)
- Utilize `=X` para sair rapidamente de qualquer tela ISPF
- Use `=6` para voltar ao prompt de comando
- Grave JCLs e SQLs em datasets organizados por tipo
- No SDSF, use `OWNER *` para listar todos os seus jobs

---

## 6. Referências Oficiais

- [IBM Documentation - z/OS TSO/E](https://www.ibm.com/docs/en/zos/latest?topic=zos-tso-e)
- [IBM z/OS Basic Skills Information Center](https://www.ibm.com/docs/en/basicskills)
- [IBM TSO/E Command Reference](https://www.ibm.com/docs/en/zos/latest?topic=commands-tso-command-reference)
- [IBM DB2 for z/OS Documentation](https://www.ibm.com/docs/en/db2-for-zos)

---

> 📌 Este manual foi desenvolvido para facilitar o dia a dia de quem está começando a atuar com **Mainframe z/OS e DB2**, especialmente em atividades como DBA, analista ou operador. Pode ser usado como material de consulta e referência prática.

