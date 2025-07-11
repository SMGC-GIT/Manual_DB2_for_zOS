# 🖥️ Manual TSO (Time Sharing Option) no z/OS - Comandos Básicos e Essenciais

Este manual é voltado para quem está iniciando no ambiente **Mainframe IBM z/OS**, especialmente profissionais que vão atuar com **TSO (Time Sharing Option)**, **ISPF**, **JCL** e **DB2 for z/OS**. Aqui você encontrará explicações claras, comandos básicos e exemplos práticos.

---

## 📚 Índice

- [1. O que é TSO?](#1-o-que-é-tso)
- [2. Acessando o TSO no z/OS](#2-acessando-o-tso-no-zos)
- [3. Introdução ao ISPF](#3-introdução-ao-ispf)
- [4. Comandos TSO Específicos](#4-comandos-tso-específicos)
  - [4.1. Comandos para datasets (arquivos)](#41-comandos-para-datasets-arquivos)
  - [4.2. Comandos para execução de jobs (JCL)](#42-comandos-para-execução-de-jobs-jcl)
  - [4.3. Comandos úteis para DB2 for z/OS](#43-comandos-úteis-para-db2-for-zos)
  - [4.4. Comandos para navegação e diagnóstico](#44-comandos-para-navegação-e-diagnóstico)
- [5. Navegando com o SDSF (Saída de Jobs)](#5-navegando-com-o-sdsf-saída-de-jobs)
- [6. Dicas e Boas Práticas](#6-dicas-e-boas-práticas)
- [7. Referências Oficiais](#7-referências-oficiais)

---

## 1. O que é TSO?

**TSO (Time Sharing Option)** é o ambiente de linha de comando dos sistemas IBM z/OS. Ele permite que cada usuário:

- Digite comandos
- Acesse arquivos (chamados *datasets*)
- Submeta jobs para processamento em lote (JCL)
- Execute comandos SQL e programas DB2
- Navegue no sistema com segurança

Você pode usar TSO de duas formas:
- **Interativamente**, via terminal 3270 (ISPF)
- **Via scripts** com comandos CLIST ou REXX

---

## 2. Acessando o TSO no z/OS

Quando você acessa o sistema (via TN3270 ou similar), será solicitado:

1. **Login (usuário)**
2. **Senha**
3. Ao logar, verá o menu do **ISPF**. Para acessar o TSO diretamente:

- Digite `Option 6` (Comandos TSO)
- Ou digite `TSO` se estiver em tela inicial

---

## 3. Introdução ao ISPF

O **ISPF (Interactive System Productivity Facility)** é uma interface baseada em menus usada sobre o TSO.

Principais opções do ISPF:

| Opção | Função                            |
|-------|-----------------------------------|
| 1     | Visualizar e editar arquivos      |
| 2     | Trabalhar com comandos UTIL       |
| 3.4   | Procurar arquivos (*datasets*)    |
| 4     | Utilitários do sistema            |
| 5     | Acessar o SDSF (ver jobs)         |
| 6     | Linha de comando TSO              |

Você pode navegar pelas opções digitando o número desejado ou um comando precedido de `=` (ex: `=6` para comandos TSO de qualquer tela).

---

## 4. Comandos TSO Específicos

Estes comandos são executados na opção `=6` do ISPF ou direto no terminal.

---

### 4.1. Comandos para datasets (arquivos)

**Datasets** são como arquivos ou pastas. Podem conter código fonte, JCLs, dados de entrada ou saída, etc.

#### `ALLOCATE` – Criar (alocar) um dataset

```tso
ALLOCATE DS('SEU.USUARIO.ARQ1') NEW SPACE(5,5) TRACKS RECFM(F B) LRECL(80) DSORG(PS)
```

Explicação:
- `DS`: Nome do dataset
- `NEW`: Criação de novo arquivo
- `SPACE(5,5)`: Espaço primário/secundário
- `LRECL`: Tamanho de cada linha
- `DSORG`: Organização sequencial (PS = Physical Sequential)

---

#### `FREE` – Liberar dataset alocado

```tso
FREE DDNAME(SYSUT1)
```

Libera o dataset associado a um *DDNAME* (usado em ALLOCATE).

---

#### `DELETE` – Apagar dataset

```tso
DELETE 'SEU.USUARIO.TEMPORARIO'
```

Remove permanentemente um dataset.

---

#### `RENAME` – Renomear dataset

```tso
RENAME 'SEU.USUARIO.ARQ1' 'SEU.USUARIO.ARQ1.BACKUP'
```

---

#### `LISTC` – Listar datasets no catálogo

```tso
LISTC LEVEL(SEU.USUARIO)
```

Mostra todos os arquivos que começam com `SEU.USUARIO`.

---

### 4.2. Comandos para execução de jobs (JCL)

**JCL (Job Control Language)** é o que permite submeter tarefas em lote (batch).

#### `SUBMIT` – Submeter um job para execução

```tso
SUBMIT 'SEU.USUARIO.JCL(JOB1)'
```

Submete o membro `JOB1` do dataset JCL.

---

#### `STATUS` – Verificar se o job está executando

```tso
STATUS
```

---

#### `OUTPUT` – Ver resultado do job executado

```tso
OUTPUT JOBNAME(JOB1)
```

Mostra os detalhes da execução (mensagens, logs, retorno).

---

### 4.3. Comandos úteis para DB2 for z/OS

#### `DB2I` – Acesso ao ambiente DB2

```tso
DB2I
```

Abre o menu de opções DB2 (inclui SPUFI, DCLGEN, BIND, etc.)

---

#### `SPUFI` – Executar comandos SQL manualmente

SPUFI = SQL Processor Using File Input. Permite digitar SQL e ver o resultado.

No menu DB2I:
1. Vá até SPUFI
2. Informe dataset de entrada e saída
3. Digite e execute comandos SQL

---

#### `DSN` – Iniciar sessão DB2 em modo linha

```tso
DSN SYSTEM(DSN1)
RUN PROGRAM(DSNTIAUL) PLAN(DSNTIAUL) PARMS('SQL')
END
```

Executa programas via CLI DB2. Exige um plano previamente bindado.

---

### 4.4. Comandos para navegação e diagnóstico

#### `ISRDDN` – Ver arquivos (DDNAMEs) em uso

```tso
ISRDDN
```

Mostra os arquivos alocados na sua sessão. Excelente para diagnóstico.

---

#### `DSLIST` – Acesso rápido ao menu 3.4

```tso
DSLIST
```

Atalho para localizar, abrir ou editar datasets.

---

#### `ISRFIND` – Procurar texto dentro de datasets

```tso
ISRFIND
```

Permite buscar palavras-chave em vários arquivos ao mesmo tempo.

---

## 5. Navegando com o SDSF (Saída de Jobs)

O **SDSF** é usado para visualizar jobs que foram executados no sistema.

Para acessá-lo:
- No ISPF: Opção `5`

Comandos principais no SDSF:

| Comando     | O que faz                                |
|-------------|-------------------------------------------|
| `ST`        | Mostra jobs ativos ou finalizados         |
| `?` ou `S`  | Visualiza a saída do job                  |
| `P`         | Envia para impressão                      |
| `C`         | Cancela job                               |
| `OWNER *`   | Mostra apenas jobs do seu usuário         |
| `PRE xxx`   | Filtra jobs por nome                      |

---

## 6. Dicas e Boas Práticas

✅ **Organize seus datasets**: Ex: `USUARIO.JCL`, `USUARIO.PROC`, `USUARIO.SOURCE`

✅ **Use prefixo padrão** para facilitar busca: ex: `LISTC LEVEL(SEU.USUARIO)`

✅ **Evite deletar sem certeza**: use `FREE` antes de `DELETE`

✅ **No ISPF, use `=X` para sair e `=6` para comandos**

✅ **Faça uso frequente do SDSF** para entender o que está acontecendo no sistema

---

## 7. Referências Oficiais

- 📘 [IBM TSO/E Command Reference](https://www.ibm.com/docs/en/zos/latest?topic=commands-tsoe-command-reference)
- 📘 [IBM z/OS Basic Skills InfoCenter](https://www.ibm.com/docs/en/basicskills)
- 📘 [IBM z/OS ISPF User's Guide](https://www.ibm.com/docs/en/zos/latest?topic=zos-ispf)
- 📘 [IBM DB2 for z/OS Documentation](https://www.ibm.com/docs/en/db2-for-zos)

---

> ✅ Este manual é um ponto de partida essencial para qualquer pessoa iniciando no universo **Mainframe z/OS** e especialmente no papel de **DBA ou analista DB2 for z/OS**.
>
> Com ele, você já poderá criar arquivos, navegar no sistema, executar jobs e consultar dados no DB2 com segurança e eficiência.

