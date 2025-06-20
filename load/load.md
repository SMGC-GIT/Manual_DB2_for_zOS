# 🚀 LOAD – Boas Práticas, Decisões Técnicas e JCL Avançado

## 📚 Índice

- [🔍 Quando usar o LOAD](#🔍-quando-usar-o-load)
- [💡 Dicas Técnicas de DBA](#💡-dicas-técnicas-de-dba)
- [❓ Questionário Técnico: Escolha do Tipo de LOAD](#❓-questionário-técnico-escolha-do-tipo-de-load)
- [📈 Volumes Ideais por Modo](#📈-volumes-ideais-por-modo)
- [⚙️ Tipos de LOAD e Cenários](#⚙️-tipos-de-load-e-cenários)
- [🛠️ Boas práticas de performance](#🛠️-boas-práticas-de-performance)
- [🚨 Erros comuns e soluções](#🚨-erros-comuns-e-soluções)
- [📝 Exemplos de JCL por tipo](#📝-exemplos-de-jcl-por-tipo)
- [📚 Referências IBM](#📚-referências-ibm)

---

# 🚀 Análise de Performance Pós-LOAD no DB2 for z/OS

## 📚 Índice

- [🔧 1. Métricas Cruciais para Avaliar o LOAD](#🔧-1-métricas-cruciais-para-avaliar-o-load)
- [🛠️ 2. Ferramentas Recomendadas pela IBM](#🛠️-2-ferramentas-recomendadas-pela-ibm)
- [📈 3. Leitura e Interpretação de Bufferpool Metrics](#📈-3-leitura-e-interpretação-de-bufferpool-metrics)
- [⚠️ 4. Indicadores de Gargalo](#⚠️-4-indicadores-de-gargalo)
- [🔍 5. Ações de Tuning Recomendadas](#🔍-5-ações-de-tuning-recomendadas)
- [📚 6. Referências IBM](#📚-6-referências-ibm)


---

### 🔍 Quando usar o LOAD

O **LOAD** é indicado para:
- Montantes altos de dados (milhares a milhões de linhas)
- Necessidade de velocidade e eficiência
- Recriação total ou parcial da tabela com gathers inline
- Uso de opções como `PRESORTED`, `PARALLEL` e `INLINE STATISTICS`

Não use para updates marginais — prefira `UPDATE`, `INSERT` ou `MERGE`.

---

### 💡 Dicas Técnicas de DBA

1. **ALTER TABELA → REFRESH LOGIC**  
   Após alterar colunas (ex: adicionar coluna, mudar tipo), rodar `DSN1COPY` inline ou `INLINE STATISTICS YES` evita inconsistências no LOAD. :contentReference[oaicite:1]{index=1}  
2. **PRESORTED** apenas se dados estiverem pré-ordenados por clustering index, evitando REORG pós-load. :contentReference[oaicite:2]{index=2}  
3. **COPYDDN/RECOVERYDDN + LOG NO** cria image copy inline e acelera o processo sem gerar logs redundantes. :contentReference[oaicite:3]{index=3}  
4. **FORMAT INTERNAL** recomendado se executar UNLOAD previamente, eliminando overhead de conversão. :contentReference[oaicite:4]{index=4}  
5. **PARALLEL** melhora uso de CPU/I/O — mas use em tabelas distribuídas para evitar contenções. :contentReference[oaicite:5]{index=5}  
6. Utilize **INLINE STATISTICS YES** para evitar post-RUNSTATS e garantir consistência de plano. :contentReference[oaicite:6]{index=6}  
7. Em tabelas com LOB/XML, indique `LOBFILE ddname` para evitar overhead de memória. :contentReference[oaicite:7]{index=7}  

---

### ❓ Questionário Técnico: Escolha do Tipo de LOAD

1. Os dados vão substituir completamente a tabela?  
   - Sim → siga para **REPLACE**  
   - Não → use **INSERT** ou **RESUME**

2. Os dados estão pré-ordenados por clustering index?  
   - Sim → use `PRESORTED YES`  
   - Não → deixe sem PRESORTED

3. Deseja evitar logging e criar image copy?  
   - Sim → inclua `COPYDDN` + `LOG NO`

4. É crítico atualizar estatísticas inline?  
   - Sim → use `INLINE STATISTICS YES`

5. Volume alto (> 5M linhas) e hardware disponível?  
   - Sim → adicione `PARALLEL(0)` ou `PARALLEL(n)`

---

### 📈 Volumes Ideais por Modo

- **Batch simples** (`INSERT`, `REPLACE`): >= 1M linhas  
- **Online SHRLEVEL CHANGE**: 100k–500k linhas, acesso concorrente  
- **PRESORTED + PARALLEL**: ideal para >5M linhas  

---

### ⚙️ Tipos de LOAD e Cenarios

| Tipo         | Cenário de uso                                             |
|--------------|------------------------------------------------------------|
| INSERT       | Inclusão incremental de dados                             |
| REPLACE      | Substituição completa de dados                            |
| RESUME       | Continuação após falha, mantendo checkpoints              |
| TERMINATE    | Finaliza carga em andamento quando preciso abortar        |
| COPYDDN      | Cria image copy inline da tabela após carga               |
| PRESORTED    | Quando dados já vêm ordenados por index de cluster        |
| PARALLEL     | Uso de múltiplas threads para acelerar a carga            |

---

### 🛠️ Boas práticas de performance

- Evite conversões CCSID e tipos durante o LOAD :contentReference[oaicite:8]{index=8}  
- Impeça REORG pós-load com `PRESORTED` quando possível :contentReference[oaicite:9]{index=9}  
- Evite fragmentação com `COPYDDN + LOG NO`  
- Acelere carga com `PRESORTED + PARALLEL` :contentReference[oaicite:10]{index=10}  
- Sempre use `INLINE STATISTICS YES` :contentReference[oaicite:11]{index=11}  
- Ao editar colo XML/LOB, use `LOBFILE` para armazenamento externo

---

### 🚨 Erros comuns e soluções

| Código/Mensagem          | Justificativa                       | Solução                                       |
|--------------------------|-------------------------------------|------------------------------------------------|
| SQL1103N                 | Tipo de dado incompatível           | Ajustar formato ou usar `FILETYPE MODIFIED`   |
| SQL3116W / SQL3185W      | Campo esperado vazio                | Rever `NULL INDICATOR`, incluir colunas       |
| DSNT408I                 | Viola integrity constraints         | Usar `ENFORCE NO`, depois `SET INTEGRITY ALL` |
| IEFC020I                 | Dataset não encontrável             | Revisar DD e altações de acesso               |
| IEC090I                  | Espaço insuficiente                 | Ajustar espaço DSNALLOC                       |

---

### 📝 Exemplos de JCL por tipo

#### INSERT com estatísticas
```jcl
//LOADINS JOB …
…
 LOAD DATA
  FROM INDD(INFILE)
  INSERT INTO TABLESPACE DB2LIB.TS1
  INLINE STATISTICS YES
  PARALLEL
/*
//INFILE DD DSN=MY.INPUT.FILE,DISP=SHR
```

#### REPLACE + COPYDDN
```jcl
//LOADREP JOB …
…
 LOAD DATA
  FROM INDD(INFILE)
  REPLACE
  COPYDDN(LAST_IC)
  INLINE STATISTICS YES
  FORMAT INTERNAL
/*
//INFILE DD DSN=MY.INPUT.INT,DISP=SHR
```

#### RESUME + PRESORTED
```jcl
//LOADRES JOB …
…
 LOAD DATA
  RESUME YES
  INSERT INTO DB2LIB.TS1
  PRESORTED YES
  SHRLEVEL CHANGE
  PARALLEL(0)
/*
```

---

### 📚 Referências IBM

- [Improving LOAD Performance](https://www.ibm.com/docs/en/db2-for-zos/12?topic=load-improving-performance) :contentReference[oaicite:12]{index=12}  
- [LOAD Utility Syntax](https://www.ibm.com/docs/en/db2-for-zos/12?topic=utilities-load) :contentReference[oaicite:13]{index=13}  
- [IBMid’s DB2 Utilities in Practice (Redbooks)](https://www.redbooks.ibm.com/redpapers/pdfs/redp5503.pdf) :contentReference[oaicite:14]{index=14}  
- [INDEXING MODE and PARALLEL details](https://www.ibm.com/docs/en/db2/11.1?topic=commands-load-using-admin-cmd) :contentReference[oaicite:15]{index=15}  


---

### 🔧 1. Métricas Cruciais para Avaliar o LOAD

Após a execução de um LOAD, é essencial monitorar esses indicadores para garantir eficiência e detectar possíveis problemas:

- **Buffer Pool Hit Ratio**  
  Fórmula:  
  ```
  (GETPAGES – PAGES-READ) / GETPAGES
  ```  
  Ideal: ≥ 85% para dados, ≥ 95% para índices :contentReference[oaicite:1]{index=1}

- **ReReads / Distinct GetPage**  
  Alta proporção indica necessidade de aumentar tamanhos do bufferpool :contentReference[oaicite:2]{index=2}

- **GETPAGE Requests vs I/O Reads**  
  Avalia se o buffer pool está atendendo eficientemente o workload :contentReference[oaicite:3]{index=3}

- **Sort Time / Commodities per Transaction**  
  LOAD pesado pode gerar overflow de sort; monitore via estatísticas DB2 :contentReference[oaicite:4]{index=4}

- **Log Write Time / LOCK Wait Time**  
  Indicadores de contenção/transação lenta após LOAD :contentReference[oaicite:5]{index=5}

---

### 🛠️ 2. Ferramentas Recomendadas pela IBM

Ferramentas eficazes para análise pós-LOAD:

- **IBM Tivoli OMEGAMON XE for DB2 (Performance Expert)**: monitora bufferpools, I/O, locks e estatísticas de LOAD/UNLOAD :contentReference[oaicite:6]{index=6}  
- **OMPE (Omegamon Performance Expert)**: fornece alertas proativos e recomendações com base em thresholds e padrões :contentReference[oaicite:7]{index=7}  
- **DB2 PM, Visual Explain, Data Studio**: EXPLAIN, performance tuning e análise de planos de acesso :contentReference[oaicite:8]{index=8}

---

### 📈 3. Leitura e Interpretação de Bufferpool Metrics

Foque nesses parâmetros ao analisar o comportamento do sistema:

| Métrica                          | O que indica                                     | Ação recomendada                    |
|----------------------------------|--------------------------------------------------|-------------------------------------|
| Bufpool Hit Ratio < ideal       | Necessidade de aumentar ou redistribuir pages   | ALTER BUFFERPOOL                    |
| Elevated ReRead % (>10%)       | Re-leitura desnecessária de páginas             | Revisar alocação e thresholds       |
| High physical reads per txn     | I/O excessivo                                    | Ajustar pré-fetch e buffers         |
| High sort overflow              | Falta de área de sort                            | Aumentar SORTWORK ou usar PARALLEL  |
| LOCK Wait > normal              | Contenção por locks pós-LOAD                     | Avaliar índices ou ISOLATION LEVEL |

---

### ⚠️ 4. Indicadores de Gargalo

- **Buffer Pool**: hit ratio caindo durante ou pós-LOAD  
- **Sort Overflow**: contagens excessivas  
- **Log Writes demorados**: em situações de IOC ON ou LOG NO  
- **Lock Waits/DCL Locks**: após cargas com SHRLEVEL CHANGE  
- **CPU Spike**: em operações PARALLEL ou REORG subsequentes

---

### 🔍 5. Ações de Tuning Recomendadas

✅ Aumentar `NPAGES` via `ALTER BUFFERPOOL` com `AUTOSIZE` e thresholds ideais :contentReference[oaicite:9]{index=9}  
✅ Ajustar `VPSEQT`, `VDWQT`, `PGSTEAL`, `PGFIX` segundo o workload :contentReference[oaicite:10]{index=10}  
✅ Isolar tablespaces de índices e dados em bufferpools distintos com diferentes padrões (random vs seq) :contentReference[oaicite:11]{index=11}  
✅ Executar `RUNSTATS` pós-LOAD se `INLINE STATISTICS` não foi usado  
✅ Monitorar locks com OMEGAMON e ajustar índices ou isolamento  
✅ Refazer `EXPLAIN` e analisar planos sob carga semelhante

---

### 📚 6. Referências IBM

- [Buffer Pool Hit Ratio e Tuning](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tuning-buffer-pools) :contentReference[oaicite:12]{index=12}  
- [IBM OMEGAMON XE for DB2 Performance](https://en.wikipedia.org/wiki/IBM_OMEGAMON) :contentReference[oaicite:13]{index=13}  
- [DB2 Performance Metrics e Monitoramento](https://www.ibm.com/docs/en/db2-for-zos/13?topic=monitoring-performance) :contentReference[oaicite:14]{index=14}  
- [Buffer Pool Memory Guidelines](https://www.ibm.com/docs/en/db2-for-zos/13?topic=bufferpool-memory-guidelines) :contentReference[oaicite:15]{index=15}

---
