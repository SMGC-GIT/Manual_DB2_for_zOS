# 🧠 IBM OMEGAMON for DB2 for z/OS – Guia Técnico e Didático

## 📚 Índice

- [1. O que é o IBM OMEGAMON?](#1-o-que-é-o-ibm-omegamon)
- [2. Principais funcionalidades](#2-principais-funcionalidades)
- [3. Arquitetura e componentes](#3-arquitetura-e-componentes)
- [4. Monitoramento do DB2 for z/OS](#4-monitoramento-do-db2-for-zos)
- [5. Telas e modos de visualização](#5-telas-e-modos-de-visualização)
- [6. Alertas e automação](#6-alertas-e-automação)
- [7. Melhores práticas de uso](#7-melhores-práticas-de-uso)
- [8. Comandos úteis e navegação](#8-comandos-úteis-e-navegação)
- [9. Integração com outras ferramentas](#9-integração-com-outras-ferramentas)
- [10. Links úteis e documentação oficial](#10-links-úteis-e-documentação-oficial)

---

## 1. O que é o IBM OMEGAMON?

**IBM OMEGAMON** é uma suíte de ferramentas de monitoramento para ambientes **z/OS**, incluindo CICS, IMS, Storage, Network, z/VM e especialmente **DB2 for z/OS**.

O módulo **OMEGAMON for DB2 Performance Expert** fornece visibilidade detalhada em tempo real e histórico sobre:

- Sessões ativas de DB2
- SQLs executando
- Locks e waits
- Utilização de buffer pools
- Acessos indevidos ou gargalos

> 🎯 Foco principal: **detecção, diagnóstico e resolução de problemas de desempenho** no ambiente DB2.

---

## 2. Principais funcionalidades

| Funcionalidade                        | Descrição                                                                 |
|--------------------------------------|---------------------------------------------------------------------------|
| 🔎 Monitoramento em tempo real        | Sessões de threads, conexões e SQLs em execução                          |
| 📊 Relatórios históricos              | Dados de desempenho e contadores SMF                                     |
| 🚨 Alertas automáticos                | Notificações sobre thresholds excedidos                                  |
| 🧠 Análise de buffer pools            | Eficiência de caches e estratégias de I/O                                |
| 🪪 Sessões de usuários                | Análise de threads conectadas e consumo por usuário                      |
| 🧮 Estatísticas de SQL                | Tempo médio, número de execuções, CPU, I/O, locking                      |
| 🔒 Diagnóstico de locks e deadlocks   | Identificação de threads bloqueadas e conflitos                         |
| 🔄 Acompanhamento de utilities        | Monitoramento de REORG, RUNSTATS, LOAD e UNLOAD                         |

---

## 3. Arquitetura e componentes

- **Monitoring Agent**  
  Captura dados de desempenho diretamente do subsistema DB2.

- **OMEGAMON Enhanced 3270 UI**  
  Interface moderna baseada em TSO/ISPF com painel intuitivo.

- **TEMS (Tivoli Enterprise Monitoring Server)**  
  Camada intermediária de controle e armazenamento de métricas.

- **TEPS (Portal Server)**  
  Interface gráfica web (opcional).

- **PARMGEN / ICAT**  
  Métodos de customização e configuração dos agentes.

---

## 4. Monitoramento do DB2 for z/OS

### Principais métricas monitoradas:

- **Thread Activity**
  - Threads ativas, suspensas, concluídas
  - Status (wait, ready, commit)

- **SQL Activity**
  - Top SQLs por CPU ou tempo total
  - SQLs mal otimizadas
  - Planos de acesso e tempo médio

- **Buffer Pools**
  - Hit ratio
  - Pages read/written
  - Page residency

- **Locking and Latching**
  - Locks em espera
  - Deadlocks detectados
  - IRLM status

- **Utilities**
  - Tempo de execução
  - Impacto em performance
  - Tabelas envolvidas

---

## 5. Telas e modos de visualização

### Modos disponíveis:

- **Classic 3270 Interface** (ISPF-like)
- **Enhanced 3270 UI** (menu gráfico, responsivo, com atalhos)
- **TEPS Web UI** (via navegador, para dashboards e históricos)

### Exemplos de telas:

- `KDPTHR` – Threads do DB2 em tempo real  
- `KDPLOCK` – Locks em espera ou conflito  
- `KDPBP` – Utilização de buffer pools  
- `KDPSTMT` – SQLs com maior consumo de recursos

---

## 6. Alertas e automação

### Alertas predefinidos:

- Long Running SQL
- Buffer Pool Hit Ratio baixo
- Thread em estado de WAIT por tempo excessivo
- Lock Wait > 10 segundos

### Automação possível:

- Geração de alertas via console ou e-mail
- Ação automática (ex: cancelamento de thread)
- Integração com NetView ou Tivoli System Automation

---

## 7. Melhores práticas de uso

| Prática                                    | Justificativa                                                              |
|--------------------------------------------|----------------------------------------------------------------------------|
| 👁️‍🗨️ Monitoramento contínuo                 | Detectar padrões de degradação com antecedência                           |
| 📌 Criação de dashboards customizados       | Visão rápida para diferentes equipes (DBA, Infra, Desenvolvimento)        |
| 🔄 Coleta de históricos diários             | Comparações entre períodos de carga                                       |
| ⚠️ Definição de thresholds personalizados   | Alertas adequados ao ambiente de negócio                                  |
| 🔍 Análise cruzada com SMF ou STROBE        | Diagnóstico mais completo e preciso                                       |

---

## 8. Comandos úteis e navegação

### Acesso à Enhanced 3270 UI:

```text
TSO KOBSTART
