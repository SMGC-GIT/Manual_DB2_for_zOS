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

# Análise de Performance Pós-LOAD no DB2 for z/OS

## Índice

- [1. Métricas Cruciais para Avaliar o LOAD](#1-métricas-cruciais-para-avaliar-o-load)
- [2. Ferramentas Recomendadas pela IBM](#2-ferramentas-recomendadas-pela-ibm)
- [3. Leitura e Interpretação de Bufferpool Metrics](#3-leitura-e-interpretação-de-bufferpool-metrics)
- [4. Indicadores de Gargalo](#4-indicadores-de-gargalo)
- [5. Ações de Tuning Recomendadas](#5-ações-de-tuning-recomendadas)
- [6. Referências IBM](#6-referências-ibm)


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


## 1. Métricas Cruciais para Avaliar o LOAD

Após um LOAD de dados, recomenda-se avaliar o ambiente para garantir performance e consistência. As principais métricas incluem:

### Buffer Pool Hit Ratio

- **Fórmula**:  
  `(GETPAGES - PAGE-READS) / GETPAGES`
- **Valor ideal**:  
  - **Dados**: ≥ 85%  
  - **Índices**: ≥ 95%
- **Significado**: Alta taxa de acerto indica que as páginas são encontradas no buffer pool, minimizando I/O físico.
- **Ação de tuning**: Aumentar o tamanho do buffer pool (`NPAGES`) ou ajustar parâmetros de pré-leitura (`VPSEQT`, `VDWQT`).

---

### ReRead Ratio

- **Fórmula**:  
  `Re-reads / Distinct GetPages`
- **Valor ideal**: < 5%
- **Significado**: Indica se páginas estão sendo relidas frequentemente sem necessidade, consumindo recursos.
- **Ação de tuning**: Redimensionar buffer pool ou revisar padrões de acesso.

---

### GETPAGE Requests vs Physical I/O Reads

- **Observação**: Se o número de `GETPAGE` está alto, mas a taxa de acertos é baixa, o buffer pool pode estar mal configurado.
- **Ação**: Aumentar `NPAGES`, analisar padrão de acesso (random/sequencial), ajustar `PGSTEAL`.

---

### Sort Overflow

- **Problema comum após LOAD**: Se a carga gera necessidade de ordenação (ex: LOAD com SORTKEYS), overflow pode ocorrer.
- **Ação**: Aumentar SORTPOOL, utilizar espaço de trabalho em disco (`SORTWKnn`), usar `PARALLEL LOAD`.

---

### Log Write Time

- **Importante** se `LOG YES` foi usado.
- **Ação**: Verificar contention no log, avaliar tamanho de buffers de log (`OUTBUFF`), considerar usar `LOG NO` se possível.

---

### LOCK Wait Time

- Comuns quando se usa `SHRLEVEL CHANGE` em tabelas ativamente acessadas.
- **Ação**: Executar em janelas de menor carga, usar isolamento `UR` em leituras concorrentes.

---

## 2. Ferramentas Recomendadas pela IBM

Ferramentas oficiais para análise detalhada:

| Ferramenta                      | Finalidade |
|----------------------------------|------------|
| IBM OMEGAMON for DB2             | Monitoramento em tempo real de buffer pools, locks, I/O |
| OMPE (Performance Expert)        | Análise histórica e geração de relatórios de performance |
| DB2 Data Studio / Visual Explain | Diagnóstico de planos de acesso e tuning de queries |
| DB2 Statistics Trace             | Coleta de estatísticas detalhadas para análise técnica |
| IFCID (Instrumentation Facility)| Eventos específicos pós-LOAD, usados com traces |

---

## 3. Leitura e Interpretação de Bufferpool Metrics

| Métrica                          | Interpretação                                 | Ação recomendada |
|----------------------------------|-----------------------------------------------|------------------|
| Bufferpool Hit Ratio < 85%      | I/O físico elevado                            | Aumentar `NPAGES`, rever `VPSEQT` |
| ReRead Ratio > 5%               | Reutilização ineficiente                      | Otimizar acesso ou ajustar buffer |
| Physical Reads por transação > 10 | I/O excessivo                                | Ajustar pré-leitura, PGSTEAL |
| Sort Overflow alto              | Área de sort insuficiente                    | Aumentar SORTPOOL, usar SORTWKnn |
| Log Write Time alto             | Contenção de log                              | Aumentar `OUTBUFF`, avaliar `LOG NO` |
| Lock Wait Time elevado          | Conflito de concorrência                     | Rodar fora do horário crítico, usar `SHRLEVEL REFERENCE` |

---

## 4. Indicadores de Gargalo

- **Buffer Pool Hit Ratio** baixo logo após o LOAD.
- **Sort overflow frequente** durante cargas com ordenação.
- **Aumento súbito de Log Write Time** em operações LOG YES.
- **Deadlocks ou timeouts** após LOAD com SHRLEVEL CHANGE.
- **CPU spikes** em cargas com paralelismo (PARALLEL).

---

## 5. Ações de Tuning Recomendadas

### Bufferpools

- Aumentar `NPAGES` usando:
  ```sql
  -ALTER BUFFERPOOL BP0 VPSIZE 5000
  ```
- Ativar `PGFIX(YES)` se adequado para fixar páginas em memória.

### Estatísticas

- Execute `RUNSTATS` após LOAD se `INLINE STATISTICS` não foi especificado:
  ```sql
  RUNSTATS TABLESPACE DB1.TS1 TABLE(ALL) INDEX(ALL)
  ```

### Índices

- Considere reconstrução de índices (com REBUILD INDEX) se a performance de acesso piorar após carga.

### Concorrência

- Evitar `SHRLEVEL CHANGE` em tabelas muito acessadas.
- Usar `SHRLEVEL REFERENCE` quando possível para evitar locks exclusivos.

### Logging

- Use `LOG NO` para cargas temporárias ou ambientes não críticos.
- Verifique uso de espaço de log após operações massivas.

---

## 6. Referências IBM

- [DB2 for z/OS Performance Monitoring](https://www.ibm.com/docs/en/db2-for-zos/13?topic=monitoring-performance)
- [Tuning Buffer Pools](https://www.ibm.com/docs/en/db2-for-zos/13?topic=tuning-buffer-pools)
- [Using IBM OMEGAMON XE for DB2 Performance](https://www.ibm.com/docs/en/om-db2-pe/5.5.0)
- [LOAD Utility Documentation](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-load-utility)
- [Runstats Utility](https://www.ibm.com/docs/en/db2-for-zos/13?topic=utilities-runstats-utility)
