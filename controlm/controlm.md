# 📘 Manual CONTROL-M no z/OS para Iniciantes — Comandos e Operações Essenciais

Este manual é voltado para profissionais que estão iniciando com o **CONTROL-M**, uma das ferramentas mais utilizadas para **automação de jobs em ambientes Mainframe z/OS**.

Aqui você entenderá:
- O que é o CONTROL-M
- Como ele funciona no dia a dia
- Quais comandos são utilizados
- Como consultar, liberar, reter e forçar jobs

---

## 📚 Índice

- [1. O que é o CONTROL-M?](#1-o-que-é-o-control-m)
- [2. Como acessar o CONTROL-M no z/OS](#2-como-acessar-o-control-m-no-zos)
- [3. Conceitos principais](#3-conceitos-principais)
- [4. Comandos mais utilizados no CONTROL-M](#4-comandos-mais-utilizados-no-control-m)
  - [4.1. Consultar jobs](#41-consultar-jobs)
  - [4.2. Liberar e reter jobs](#42-liberar-e-reter-jobs)
  - [4.3. Forçar e excluir jobs](#43-forçar-e-excluir-jobs)
  - [4.4. Alterar atributos do job](#44-alterar-atributos-do-job)
- [5. Tabela de comandos CONTROL-M](#5-tabela-de-comandos-control-m)
- [6. Boas práticas e dicas](#6-boas-práticas-e-dicas)
- [7. Referências Oficiais e Links Úteis](#7-referências-oficiais-e-links-úteis)

---

## 1. O que é o CONTROL-M?

**CONTROL-M** é um **agendador de jobs** usado para automatizar e controlar fluxos de trabalho batch em sistemas Mainframe. Ele substitui a necessidade de executar JCLs manualmente, organizando execuções de forma automática e controlada.

Principais benefícios:
- Automatiza rotinas complexas
- Reduz erros manuais
- Controla dependências entre jobs
- Permite visualização e intervenção em tempo real

---

## 2. Como acessar o CONTROL-M no z/OS

Você acessa o CONTROL-M a partir da **interface ISPF**, geralmente via:

```text
ISPF Option → CONTROL-M ou digite CTM na linha de comando
```

Ou diretamente:

```tso
TSO CTM
```

Isso abrirá a tela de controle de jobs, onde você poderá:
- Ver o status dos jobs
- Executar comandos
- Acompanhar a fila de jobs
- Realizar operações manuais

---

## 3. Conceitos principais

| Termo         | Significado                                                                 |
|---------------|------------------------------------------------------------------------------|
| **Job**       | Um processo batch que será executado                                        |
| **Tabela**    | Conjunto de jobs organizados por área, sistema ou aplicação                 |
| **Status**    | Estado atual do job (WAITING, HELD, EXECUTING, ENDED OK, ENDED NOTOK, etc.) |
| **RBA**       | Refere-se ao identificador do job em execução (Registo de Batch Ativo)      |
| **Order Date**| Data de agendamento ou entrada do job                                       |

---

## 4. Comandos mais utilizados no CONTROL-M

Os comandos podem ser digitados no painel de controle da interface ISPF do CONTROL-M, na linha ao lado do job ou no menu de comando.

---

### 4.1. Consultar jobs

#### ➤ Listar jobs por nome:

```ctm
FIND JOB=MEUJOB
```

#### ➤ Filtrar por status ou data:

```ctm
FIND JOB=MEUJOB STAT=WAITING
FIND JOB=MEUJOB ODATE=HOJE
```

---

### 4.2. Liberar e reter jobs

#### ➤ Liberar job retido (HELD):

```ctm
FREE JOB=MEUJOB
```

#### ➤ Reter um job manualmente:

```ctm
HOLD JOB=MEUJOB
```

---

### 4.3. Forçar e excluir jobs

#### ➤ Forçar execução de job (independente de condições):

```ctm
FORCE JOB=MEUJOB ODATE=HOJE
```

> ⚠️ Use com cautela. Forçar um job ignora dependências e pode causar falhas.

#### ➤ Excluir um job da fila:

```ctm
DELETE JOB=MEUJOB
```

---

### 4.4. Alterar atributos do job

#### ➤ Alterar data de execução (Order Date):

```ctm
MODIFY JOB=MEUJOB ODATE=HOJE
```

#### ➤ Alterar prioridade de execução:

```ctm
MODIFY JOB=MEUJOB PRIORITY=5
```

---

## 5. Tabela de comandos CONTROL-M

| Comando          | O que faz                                      |
|------------------|-------------------------------------------------|
| `FIND`           | Localiza jobs por nome, status ou data          |
| `HOLD`           | Reter job (pausar execução)                     |
| `FREE`           | Liberar job retido                              |
| `FORCE`          | Força execução imediata do job                  |
| `DELETE`         | Remove job da fila de execução                  |
| `MODIFY`         | Altera atributos do job                         |
| `SHOW`           | Exibe detalhes do job                           |
| `ORDER`          | Ordena (ativa) manualmente um job da tabela     |
| `REFRESH`        | Atualiza a tela com o status mais recente       |

---

## 6. Boas práticas e dicas

- ✅ Sempre verifique **dependências** antes de forçar a execução de um job.
- 📌 Use `SHOW JOB=xxx` para analisar o histórico e detalhes antes de qualquer ação.
- 🧠 Aprenda a identificar *pré-condições* e *pós-condições* nos fluxos.
- ⚠️ Evite `FORCE` e `DELETE` sem confirmação de analistas responsáveis.
- 🔄 Use `REFRESH` frequentemente para evitar trabalhar com dados desatualizados.

---

## 7. Referências Oficiais e Links Úteis

- [📘 BMC Documentation - CONTROL-M for z/OS](https://docs.bmc.com/docs/controlmzos)
- [📘 BMC Communities – Artigos e dúvidas frequentes](https://community.bmc.com/s/)
- [📘 CONTROL-M Automation API (moderna, mas relevante)](https://docs.bmc.com/docs/automation-api)
- [IBM z/OS Mainframe Basics](https://www.ibm.com/docs/en/basicskills)

---

> ✅ Este guia foi desenvolvido para quem está começando com o CONTROL-M e precisa interagir com jobs em ambientes Mainframe. Ele serve como uma base sólida para operadores, analistas e DBAs atuando com **automações batch no z/OS**.
