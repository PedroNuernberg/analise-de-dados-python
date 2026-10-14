# Pipeline ETL — Viagens a Serviço (Portal da Transparência)

Projeto final do curso **Análise de Dados com Python** (SENAI / LAB365). Pipeline em **Python + PostgreSQL** que baixa os dados abertos de *Viagens a Serviço* do Governo Federal (4 CSVs, ~1,9 milhão de linhas), guarda o dado bruto, limpa e tipa, e responde 7 perguntas de negócio com tabelas e gráficos.

**Tecnologias:** `pandas`, `psycopg2`, `requests`, `matplotlib`, PostgreSQL, arquitetura Medallion (Raw → Silver → Gold), leitura em blocos (`chunksize`), carga idempotente (`TRUNCATE`) e resiliente (`try/except` + `rollback()`), credenciais no `.env`.

```text
 CSV ──1_extrair.py──▶ RAW (4 tabelas, tudo texto)
     ──2_transformar.py──▶ SILVER (tipos, colunas calculadas, PK/FK/constraints)
     ──3_analise.ipynb──▶ GOLD (gold_* + vw_gold_*, JOIN + GROUP BY)
```

## Arquivos

| Arquivo | O que faz |
|---|---|
| `0_criar_banco.sql` | Cria o banco `transparencia` e as 8 tabelas (4 Raw + 4 Silver) |
| `1_extrair.py` | Baixa o `.zip`, lê os CSVs em blocos e carrega a Raw |
| `2_transformar.py` | Converte tipos, calcula colunas e carrega a Silver |
| `3_analise.ipynb` | Cria a Gold, responde as 7 perguntas e desenha os gráficos |
| `config.py` / `banco.py` | Parâmetros (lê o `.env`) e conexão com o PostgreSQL |
| `.env.example` / `requirements.txt` | Modelo de credenciais e dependências |

## Como executar

Pré-requisitos: Python 3, PostgreSQL ligado e internet (~70 MB). Não roda no Google Colab.

1. `pip install -r requirements.txt`
2. Copie `.env.example` para `.env` e preencha `POSTGRES_PASSWORD`.
3. Execute `0_criar_banco.sql` em duas partes: o `CREATE DATABASE` conectado ao banco `postgres`; o restante conectado ao `transparencia`.
4. Rode `python 1_extrair.py` e depois `python 2_transformar.py`.
5. Abra `3_analise.ipynb` e execute todas as células.

Os scripts podem ser reexecutados sem duplicar dados. O pipeline leva cerca de 5 minutos.

## Decisões de modelagem

- **Raw:** cópia fiel do CSV, tudo `VARCHAR`, carga posicional.
- **Silver:** PK, FK e 2 constraints por tabela; `silver_viagem` (mãe) é carregada primeiro.
- **Conversões:** `texto_para_decimal` (`"1272,97"` → `1272.97`) e `texto_para_data` (`"17/09/2024"` → data); vazio vira `NULL`.
- **`valor_total`** = diárias + passagens + outros gastos − devolução.
- **`duracao_dias`** = `(data_fim − data_inicio) + 1`.
- Só viagens `Realizada` entram nas perguntas; destinos exigem **≥ 100 viagens**.

## Conferência da carga

Todas as linhas chegaram à Raw e à Silver: viagem 341.860 · pagamento 606.916 · passagem 167.260 · trecho 763.349. A soma dos pagamentos é idêntica nas duas camadas (R$ 1.194.365.457,37).

## Resultados das perguntas

| # | Pergunta | Resposta |
|---|---|---|
| 1 | 5 órgãos com maior custo total | Justiça e Segurança Pública (R$ 485,7 mi), Defesa (154,6), Educação (109,8), Meio Ambiente (49,3), Previdência Social (40,2) |
| 2 | 3 destinos com maior custo médio | Brasília/DF (R$ 27.401,80), Genebra/Suíça (25.469,22), Nova York/EUA (21.214,19) |
| 3 | Viagem mais longa | `0000000000020699856` (Previdência Social): 384 dias, custo R$ 0,00 |
| 4 | Pagamento com maior valor médio | Diárias: R$ 2.078,79 |
| 5 | Transporte mais usado | Veículo Oficial: 385.734 trechos (51,0%) |
| 6 | UF de destino mais frequente | São Paulo: 81.727 trechos |
| 7 | Órgão que mais pagou | Fundo Nacional de Segurança Pública: R$ 278,3 mi |

## Pontos de atenção na base

- 18.545 viagens realizadas têm custo R$ 0,00 (sem gastos registrados no arquivo).
- 13.774 viagens terminam após 30/06/2025 (o arquivo traz viagens *iniciadas* no semestre).
- 3 viagens têm custo negativo (devolução maior que os gastos).

## Melhorias possíveis

Separar cidades da coluna `destinos`; usar `COPY` na carga; adicionar testes das funções de conversão; agendar a execução.
