# Modelo Preditivo — Copa do Mundo 2026

---

### Visão Geral (English below this section) 

Oiiiieee!! Este projeto constrói um modelo de machine learning para prever os resultados dos jogos da Copa do Mundo 2026 (vitória, empate ou derrota). O modelo é treinado com dados históricos de partidas entre seleções nacionais e é avaliado retroativamente com os jogos reais da Copa que já aconteceram.

---

### Dados Utilizados

Os dados foram obtidos do Kaggle, do dataset público de [Mart Jürisoo](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017), que contém todos os resultados de jogos internacionais masculinos desde 1872. Foram utilizados três arquivos:

- **results.csv** — data, times, placar de cada jogo
- **shootouts.csv** — resultados de disputas de pênaltis (importante para determinar o vencedor real em eliminatórias)
- **former_names.csv** — mapeamento de nomes antigos de países para os nomes atuais (ex: Zaire → República Democrática do Congo)

---

### Limpeza dos Dados

**Normalização de nomes:** Alguns países mudaram de nome ao longo dos anos. Sem esse mapeamento, a República Democrática do Congo teria o histórico partido em dois, os jogos antigos como "Zaire" e os recentes como "DR Congo". O arquivo `former_names.csv` resolve isso.

**Incorporação de pênaltis:** O `results.csv` registra apenas o placar do tempo regulamentar. Em jogos eliminatórios que terminam empatados e vão para os pênaltis, é necessário saber quem realmente avançou. O `shootouts.csv` fornece essa informação, e criamos uma coluna `winner` que reflete o vencedor real de cada partida.

**Filtragem por times:** Somente partidas envolvendo os 48 times classificados para a Copa 2026 foram mantidas. Dados de times que nem participam do torneio não contribuem para as previsões, seria apenas 'noise'. A lista de times foi extraída diretamente do dataset (a partir dos jogos reais da Copa 2026), e não de uma fonte externa, para evitar erros.

**Divisão em três conjuntos:**
- `train` — todas as partidas históricas antes de 2023 (base de treinamento)
- `historical_test` — partidas de 2023 a 2025 (validação em jogos recentes conhecidos)
- `wc_test` — jogos da Copa 2026 (teste final de acurácia)

---

### Considerações Importantes sobre os Dados

**Por que manter todos os dados históricos e não só os últimos 20 anos?**
Uma preocupação inicial foi que dados muito antigos distorceriam o modelo, já que as seleções mudam completamente ao longo das décadas. A solução foi manter todo o histórico disponível, mas aplicar **pesos de recência**: jogos recentes valem mais do que jogos antigos. Cortar os dados arbitrariamente em 2000 descartaria informação útil que poderia ser aproveitada com peso reduzido.

**Pesos de recência:**
- Últimos 3 anos: peso 3.0
- Últimos 3 a 8 anos: peso 2.0
- Mais antigos: peso 1.0

**Pesos por torneio:**
- Copas do Mundo, Eurocopas, Copas América etc.: peso 2.0
- Eliminatórias: peso 1.5
- Amistosos: peso 1.0

---

### Engenharia de Atributos

Para cada partida, construí os seguintes atributos baseados no histórico de treinamento de cada time (sempre calculados **apenas** com dados de `train`, nunca com dados futuros):

| Atributo | Descrição |
|---|---|
| `a_attack` / `b_attack` | Média ponderada de gols marcados por jogo |
| `a_defense` / `b_defense` | Média ponderada de gols sofridos por jogo |
| `a_win_rate` / `b_win_rate` | Taxa de vitória ponderada |
| `a_draw_rate` / `b_draw_rate` | Taxa de empate ponderada |
| `a_goal_diff` / `b_goal_diff` | Saldo de gols médio ponderado por jogo |
| `a_form` / `b_form` | Pontos acumulados nos últimos 5 jogos (V=3, E=1, D=0) |
| `*_diff` | Diferença de cada atributo entre os dois times |
| `h2h_a_win_rate` | Taxa de vitória do time A no confronto direto específico |
| `h2h_draw_rate` | Taxa de empate no confronto direto específico |
| `is_neutral` | Se a partida é em campo neutro (todos os jogos da Copa são) |

---

### Processo e Erros ao Longo do Caminho

#### Tentativa 1: Random Forest
O primeiro modelo foi um **Random Forest** com os atributos básicos. O resultado foi decepcionante: **36% de acurácia** que é pouco melhor do que acertar aleatoriamente (33%).

**Problemas identificados:**

1. **Empates quase nunca eram previstos.** Dos 72 jogos da Copa no conjunto de teste, 38 terminaram empatados (53%). O modelo captava apenas 5% deles. Empates são difíceis de prever porque os atributos médios dos dois times acabam sendo similares, exatamente quando um empate é mais provável.

2. **Confusão entre mando de campo e campo neutro.** O conjunto de treinamento inclui partidas com mandante real, onde o fator casa é significativo. Mas os jogos da Copa são todos em campo neutro. O modelo aprendia um padrão (vantagem do mandante) que simplesmente não existe no torneio.

#### Tentativa 2: XGBoost com Retreinamento Incremental
Troquei o Random Forest pelo **XGBoost** e adicionamos o retreinamento incremental por rodada: após cada rodada da Copa, os resultados são incorporados ao treinamento e o modelo é retreinado antes de prever a próxima rodada.

Isso trouxe uma melhora real. O melhor resultado foi **60% na Rodada 2** que é acima da média de modelos profissionais de previsão de futebol (50–55%).

**Por que a Rodada 3 foi pior?**
A fase de grupos da Copa tem uma dinâmica peculiar na terceira rodada: times já classificados poupam jogadores, times já eliminados jogam sem pressão, e às vezes existe interesse mútuo em um empate. Nenhum modelo consegue prever isso com dados históricos.

#### Tentativa 3: Modelo em Dois Estágios
A solução para o problema dos empates foi dividir a previsão em dois estágios separados:

- **Estágio 1:** Este jogo vai terminar em empate? (binário: empate vs. resultado decisivo)
- **Estágio 2:** Se for resultado decisivo, quem vence? (binário: time A ou time B)

O limiar de empate foi calibrado no conjunto histórico 2023–2025 antes de tocar nos dados da Copa, evitando overfitting ao torneio.

Novos atributos adicionados nessa versão: `draw_rate`, `h2h_draw_rate`, `goal_diff`, e `is_neutral`.

---

### Resultados Finais

| Rodada | Acurácia |
|---|---|
| Rodada 1 | a definir após execução |
| Rodada 2 | ~60% (melhor resultado) |
| Rodada 3 | inferior (efeito tático da 3ª rodada) |

---

### Próximos Passos

O maior ganho potencial de acurácia ainda não implementado é o **Elo Rating**, um sistema de pontuação dinâmica que atualiza a força de cada seleção após cada jogo. Deve adicionar de 3 a 5 pontos percentuais de acurácia.

Outros possíveis aprimoramentos:
- Regressão de Poisson combinada com XGBoost em ensemble
- Dados de elenco e lesões para partidas futuras
- Rankings FIFA históricos como atributo adicional

---

### Como Executar

1. Baixe `results.csv`, `shootouts.csv` e `former_names.csv` do [Kaggle](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017) e coloque-os na pasta do projeto
2. Instale as dependências: `pip install pandas numpy xgboost scikit-learn matplotlib seaborn`
3. Execute os notebooks em ordem:
   - `data_cleaning.ipynb`
   - `feature_engineering.ipynb`
   - `prediction_model.ipynb`

---

---

## English

### Overview

This project builds a machine learning model to predict the outcomes of 2026 FIFA World Cup matches (win, draw, or loss). The model is trained on historical international match data and evaluated retroactively against real World Cup 2026 results.

---

### Data Sources

Data was sourced from Kaggle, specifically [Mart Jürisoo's public dataset](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017) of all men's international football results since 1872. Three files were used:

- **results.csv** — date, teams, and score for every match
- **shootouts.csv** — penalty shootout outcomes (needed to determine the true winner in knockout games)
- **former_names.csv** — mapping of old country names to current ones (e.g. Zaire → DR Congo)

---

### Data Cleaning

**Name normalization:** Some countries have renamed over time. Without this mapping, DR Congo's match history would be split between "Zaire" (old games) and "DR Congo" (recent ones), effectively treating them as two different teams. The `former_names.csv` resolves this.

**Incorporating shootouts:** `results.csv` only records the 90-minute score. In knockout games that go to penalties, I need to know who actually advanced. I merged `shootouts.csv` to create a `winner` column reflecting the true match winner.

**Filtering to WC 2026 teams:** Only matches involving the 48 World Cup 2026 teams were kept. The team list was extracted directly from the dataset rather than hardcoded which turned out to matter since initial search results had the wrong teams (Haiti and Curaçao are in, not Honduras and Costa Rica as initially believed).

**Three-way split:**
- `train` — all historical matches before 2023 (training base)
- `historical_test` — 2023–2025 matches (validation on recent known results)
- `wc_test` — WC 2026 games (final accuracy test)

---

### Key Data Decisions

**Why keep all historical data instead of just the last 20 years?**
An early concern was that data from decades ago would distort the model since squads change completely over time. The solution was to keep all available history but apply **recency weights** (recent games count more than old ones). Cutting data at 2000 would discard usable information that could instead be kept at a lower weight.

**Recency weights:**
- Last 3 years: 3.0x
- Last 3–8 years: 2.0x
- Older: 1.0x

**Tournament importance weights:**
- World Cup, Euros, Copa América, etc.: 2.0x
- Qualifiers: 1.5x
- Friendlies: 1.0x

---

### Feature Engineering

For each match, the following features were built from the training history of each team (always computed using **only** `train` data and never future data):

| Feature | Description |
|---|---|
| `a_attack` / `b_attack` | Weighted avg goals scored per game |
| `a_defense` / `b_defense` | Weighted avg goals conceded per game |
| `a_win_rate` / `b_win_rate` | Weighted win rate |
| `a_draw_rate` / `b_draw_rate` | Weighted draw rate |
| `a_goal_diff` / `b_goal_diff` | Weighted avg goal difference per game |
| `a_form` / `b_form` | Points in last 5 games (W=3, D=1, L=0) |
| `*_diff` | Difference of each stat between the two teams |
| `h2h_a_win_rate` | Team A's weighted win rate in this specific head-to-head |
| `h2h_draw_rate` | Draw rate in this specific head-to-head |
| `is_neutral` | Whether match is on neutral ground (all WC games are) |

---

### Process and Lessons Learned

#### Attempt 1: Random Forest
The first model was a **Random Forest** with basic features. Result: **36% accuracy** which is barely above random chance (33.3%).

**Problems identified:**

1. **Draws were almost never predicted.** Of 72 WC 2026 games in the test set, 38 ended in draws (53%). The model only caught 5% of them. Draws are structurally hard to predict because the average stats of two evenly matched teams look similar, which is precisely when a draw is most likely.

2. **Home advantage confound.** The training set includes matches with a real home team, where home advantage is significant. WC games are all on neutral ground. The model was learning a home advantage signal that simply doesn't exist in the tournament.

#### Attempt 2: XGBoost with Incremental Retraining
I replaced Random Forest with **XGBoost** and added incremental retraining by round: after each completed WC round, those results are added to the training set and the model is retrained before predicting the next round.

Best result: **60% on Round 2** which is above the typical range of professional football prediction models (50–55%).

**Why was Round 3 worse?**
The final group stage matchday has a known dynamic: teams already qualified rest key players, teams already eliminated play without pressure, and sometimes mutual draws serve both teams' interests. No model can predict this from historical data.

#### Attempt 3: Two-Stage Model
The draw problem was addressed by splitting prediction into two separate problems:

- **Stage 1:** Will this match be a draw? (binary: draw vs. decisive result)
- **Stage 2:** If decisive, who wins? (binary: team A or team B)

The draw threshold was calibrated on the 2023–2025 historical test set before touching WC data, avoiding overfitting to the tournament.

New features added: `draw_rate`, `h2h_draw_rate`, `goal_diff`, and `is_neutral`.

---

### Results

| Round | Accuracy |
|---|---|
| Round 1 | TBD after execution |
| Round 2 | ~60% (best result) |
| Round 3 | Below rounds 1 & 2 (Round 3 tactical effect) |

---

### Next Steps

The highest potential accuracy gain not yet implemented is **Elo Ratings**, a dynamic strength rating system that updates after every match. Expected improvement: +3 to +5 percentage points.

Other potential improvements:
- Poisson regression (goals-based model) combined with XGBoost in an ensemble
- Squad and injury data for future match predictions
- Historical FIFA rankings as an additional feature

---

### How to Run

1. Download `results.csv`, `shootouts.csv`, and `former_names.csv` from [Kaggle](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017) and place them in the project folder
2. Install dependencies: `pip install pandas numpy xgboost scikit-learn matplotlib seaborn`
3. Run notebooks in order:
   - `data_cleaning.ipynb`
   - `feature_engineering.ipynb`
   - `prediction_model.ipynb`
