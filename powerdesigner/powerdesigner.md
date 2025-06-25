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

> ⚠️ Observação: Este manual não aborda técnicas de modelagem conceitual ou lógica. O foco é **manutenção, leitura e apoio à modelagem física DB2** em ambientes críticos como o da CEF.
