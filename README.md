# Modelo Preditivo de Cancelamento de Serviços

Projeto de ciência de dados para predição de cancelamento de serviços em uma empresa de saneamento, utilizando dados de serviços operacionais.

## Objetivo

Prever, **no momento em que uma ordem de serviço é aberta**, se aquele serviço será cancelado. Com base na probabilidade de cancelamento e nas regras de prioridade do contrato (reincidência de reclamação, registros em ouvidoria/PROCON), a empresa aciona um fiscal para vistoriar a execução — garantindo que o serviço seja realizado corretamente e que o cliente seja devidamente atendido.

## Estrutura do Projeto

```
├── data/
│   ├── raw/            # Dados brutos originais (não versionados)
│   └── processed/      # Dados processados (não versionados)
├── notebooks/
│   ├── 01_eda.ipynb             # Análise Exploratória de Dados
│   ├── 02_feature_engineering.ipynb  # Engenharia de Features
│   └── 03_modeling.ipynb        # Treinamento e Avaliação dos Modelos
├── src/
│   ├── data/
│   │   └── preprocessing.py     # Pipeline de limpeza e preparação
│   ├── features/
│   │   └── build_features.py    # Construção de features agregadas
│   └── models/
│       └── train.py             # Treinamento, tuning e avaliação
└── reports/
    └── figures/                 # Gráficos gerados (não versionados)
```

## Regras de Negócio

### Filtro de Polos
Apenas os polos `07`, `12` e `13` são considerados na análise.

### Serviços Analisados
Somente os serviços que fazem parte dos indicadores de desempenho do contrato e têm impacto direto na remuneração são incluídos (`COD_SERVICO`):

```
1.010, 1.020, 1.030, 1.040, 1.044, 1.050, 1.060, 1.070, 1.180, 1.190,
1.200, 1.230, 1.231, 1.240, 1.250, 1.270, 1.280, 1.450, 1.470, 1.500,
1.501, 1.510, 1.511, 1.515, 1.520, 1.530, 1.540, 1.550, 1.555, 1.560,
1.570, 1.580, 1.590, 1.601, 1.602, 1.603, 1.604, 1.605, 1.608, 1.620,
1.691, 1.710, 1.720, 1.740, 1.770, 2.010, 2.040, 2.080, 2.090, 2.200,
2.210, 2.220, 2.230, 2.240, 2.250, 2.260, 2.270, 2.300, 2.310, 2.320,
2.330, 2.430, 2.460, 2.462, 2.470
```

### Prazos por Serviço
Cada serviço possui um prazo de execução definido contratualmente:

| Prazo | Serviços |
|---|---|
| 24h | 1.010, 1.020, 1.030, 1.050, 1.060, 1.070, 1.180, 1.190, 1.250, 1.450, 1.470, 1.500, 1.501, 1.510, 1.511, 1.520, 1.530, 1.540, 1.550, 1.560, 1.570, 1.580, 1.590, 1.605, 1.620, 1.770, 2.010, 2.200, 2.210, 2.220, 2.230, 2.240, 2.250, 2.260, 2.270, 2.460, 2.462 |
| 48h | 1.601, 1.602, 1.603, 1.604, 1.608, 1.691 |
| 72h | 2.080 |
| 96h | 1.515, 1.555, 2.430, 2.470 |
| 120h | 1.040, 1.270 |
| 168h | 1.200, 1.230, 1.231, 1.240, 1.280, 1.710, 1.720, 1.740, 2.040, 2.090, 2.300, 2.310, 2.320, 2.330 |
| 240h | 1.044 |

### Definição do Target
- **`cancelado = 1`** → `COD_SERVICO_EXECUTADO` começa com `'3'` (serviço cancelado)
- **`cancelado = 0`** → serviço executado normalmente

### Tratamento de Nulos
Linhas com nulos nas colunas `COD_SERVICO_EXECUTADO`, `COD_BACIA_ESGOT`, `COD_SETOR_ABAST` e `COD_AREA_SERVICO` são removidas (< 0,5% do total — impacto desprezível).

## Momento da Predição e Features Válidas

A predição ocorre **no momento da abertura da ordem** (`DATA_REG`). Apenas informações disponíveis nesse instante podem ser usadas como features:

| Feature | Descrição |
|---|---|
| `COD_SERVICO_ETAPA` | Tipo de serviço solicitado |
| `prazo_horas` | Prazo contratual (derivado do serviço) |
| `PRIORI` | Prioridade definida na abertura |
| `COD_POLO` / `COD_AREA_SERVICO` | Localização geográfica |
| `COD_SETOR_ABAST` / `COD_BACIA_ESGOT` | Setor de abastecimento / bacia de esgoto |
| `GRUPO_SERVICO_OPERACIONAL` | Agrupamento operacional do serviço |
| Dia da semana / mês / hora | Derivados de `DATA_REG` |
| Histórico da ligação | Taxa de cancelamento passada, reincidências — apenas eventos **anteriores** à ordem |

> **Data leakage — variáveis proibidas:** `duracao_servico_min`, `DATA_FIM_SERVICO` e `COD_SERVICO_EXECUTADO` (este último é o próprio target) só existem após o encerramento do serviço e **não devem ser usadas como features**.

## Etapas do Projeto

1. **EDA** — distribuição das variáveis, análise temporal, taxa de cancelamento por serviço/polo/prazo
2. **Feature Engineering** — features temporais (dia/mês/hora), históricas por ligação (reincidência), características do serviço
3. **Modelagem** — Regressão Logística (baseline), Random Forest, XGBoost/LightGBM
4. **Avaliação** — AUC-ROC, Precision-Recall, análise de importância de features (SHAP)

## Como Executar

```bash
pip install -r requirements.txt
jupyter lab
```

Abra os notebooks na ordem numérica.
