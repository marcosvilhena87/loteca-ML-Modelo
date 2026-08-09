# Loteca ML Default

Projeto para desenvolver e aprimorar uma estratégia de Loteca baseada no histórico de concursos em `data/concursos_anteriores.csv`, gerando **um único palpite** para o próximo concurso com foco em maximizar a frequência de **13 pontos ou mais**.

## Objetivo

O pipeline deve separar claramente:

1. **Probabilidades do jogo** — `p(1)`, `p(X)` e `p(2)`.
2. **Modelo estatístico/ML** — estima a chance de acerto/falha de Top1, Top2 e Top3.
3. **Ranking de risco do Top1** — ordena os 14 jogos pela chance de falha do Top1.
4. **Otimizador do volante** — escolhe globalmente os 9 secos e 5 duplos respeitando as constraints.
5. **Backtest da estratégia completa** — mede o desempenho histórico por concurso.

```text
ODDS / PROBABILIDADES
        ↓
PRÉ-PROCESSAMENTO
        ↓
MODELO
P(Top1 acerta) / P(Top1 falha)
        ↓
RANKING DE RISCO
escolhe candidatos a excluir Top1
        ↓
OTIMIZADOR
9 secos + 5 duplos
        ↓
PALPITE ÚNICO + DEBUG + BACKTEST
```

---

## Definições

- `p(1)`, `p(X)` e `p(2)` são as probabilidades de vitória do mandante, empate e vitória do visitante.
- `p(top1)`, `p(top2)` e `p(top3)` são essas probabilidades ordenadas da maior para a menor.
- Em caso de empate, usar:

```text
1 > 2 > X
```

- `top1`, `top2` e `top3` representam, via One-Hot Encoding, qual posição ordenada correspondeu ao resultado real.

---

## Hard Constraints

1. Exatamente **9 Top1, 5 Top2 e 5 Top3** selecionados.
2. Exatamente **9 secos, 5 duplos e 0 triplos**.
3. Sempre apostar a favor do **FLAMENGO/RJ**.

Regra do Flamengo:

- Flamengo mandante → o palpite deve conter `1`.
- Flamengo visitante → o palpite deve conter `2`.

### Consequência estrutural

Com 9 secos e 5 duplos existem 19 marcações.

```text
9 Top1 + 5 Top2 + 5 Top3 = 19 marcações
```

Logo, a estrutura natural do volante é:

```text
9 jogos: Top1 seco
5 jogos: Top2 + Top3
```

Portanto, a principal pergunta estratégica é:

> **Quais são os cinco jogos em que o Top1 deve ser descartado?**

O ML, as runs, os ranks, a entropia e as demais features devem ser avaliados principalmente pela capacidade de responder essa pergunta.

---

## Soft Constraints

1. Explorar métricas relacionadas às runs de Top1, ordenando os jogos por `p(top1)` decrescente por concurso.
2. Fazer o mesmo para Top2.
3. Fazer o mesmo para Top3.
4. Se possível, apostar em **empate ou derrota do PALMEIRAS/SP**.

A preferência contra o Palmeiras deve ser bônus/penalidade no score, nunca regra absoluta.

---

## Estrutura do repositório

```text
main.py

data/
├── concursos_anteriores.csv
└── proximo_concurso.csv

scripts/
├── common.py
├── preprocess_data.py
├── train_model.py
└── predict_results.py

models/

output/
├── predictions.csv
└── backtest_results.csv
```

### `scripts/common.py`

- normalização dos clubes;
- reconstrução de Top1/Top2/Top3;
- desempate `1 > 2 > X`;
- validações;
- hard/soft constraints;
- logging;
- configuração UTF-8.

### `scripts/preprocess_data.py`

- leitura dos CSVs;
- conversão das probabilidades;
- validação;
- criação de features;
- ranks;
- runs sem leakage;
- alvos históricos.

### `scripts/train_model.py`

- modelos;
- walk-forward temporal;
- calibração;
- baselines;
- ranking de falha Top1;
- métricas por concurso;
- ablation study;
- validação estatística.

### `scripts/predict_results.py`

- previsão do próximo concurso;
- `P_modelo_falha_top1`;
- `risk_rank_top1`;
- score de exclusão;
- otimização global;
- constraints;
- `predictions.csv`;
- debug detalhado.

---

## Formato dos dados

Os CSVs usam:

- delimitador: `;`
- decimal: `,`
- preferencialmente UTF-8.

```python
pd.read_csv(
    arquivo,
    sep=";",
    decimal=",",
    encoding="utf-8"
)
```

No Windows:

```python
import sys

if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8", errors="replace")
    sys.stderr.reconfigure(encoding="utf-8", errors="replace")
```

---

## Validações obrigatórias

Antes do treinamento ou previsão:

- exatamente 14 jogos por concurso;
- sem duplicidade `Concurso + Jogo`;
- probabilidades entre 0 e 1;
- `p(1)+p(X)+p(2) ≈ 1`;
- `p(top1) >= p(top2) >= p(top3)`;
- consistência entre Tops e probabilidades originais;
- One-Hot histórico válido;
- concurso futuro sem resultado preenchido;
- clubes normalizados;
- ausência de leakage temporal.

Top1/Top2/Top3 devem ser preferencialmente recalculados a partir de `p(1)/p(X)/p(2)`.

---

## Features

### Probabilidades

```text
p(1)
p(X)
p(2)
p(top1)
p(top2)
p(top3)
```

### Ranks

```text
rank_top1
rank_top2
rank_top3
```

### Incerteza

```text
gap12   = p(top1) - p(top2)
gap23   = p(top2) - p(top3)
ratio12 = p(top1) / p(top2)
```

Também avaliar:

- entropia;
- concentração das probabilidades;
- distância de `p(top1)` para `1/3`;
- percentil de `p(top1)`;
- diferença Top1–Top2.

Entropia:

```text
H = -Σ p(i) * log(p(i))
```

---

## Runs

Runs são **features históricas**, não regra automática de reversão.

Para cada rank de Top1/Top2/Top3, calcular apenas usando concursos anteriores:

- run atual de acertos;
- run atual de erros;
- maior run histórica;
- hit rate últimas 5, 10, 20 e 50 observações;
- distância desde último acerto/erro;
- número de observações.

### Suavização

```text
p_ajustada = (acertos + alpha) / (n + alpha + beta)
```

### Regra anti-leakage

```text
feature do concurso N = histórico disponível somente até N-1
```

---

## Alvos do modelo

Além de `1/X/2`, testar:

```text
P(resultado == Top1)
P(resultado == Top2)
P(resultado == Top3)
```

O alvo principal da estratégia deve ser:

```text
P(Top1 acerta)
```

ou:

```text
P_falha_top1 = 1 - P(Top1 acerta)
```

---

## Modelo binário específico para falha do Top1

Testar explicitamente:

```text
y = 1 -> Top1 falhou
y = 0 -> Top1 acertou
```

Saída:

```text
P_modelo_falha_top1
```

Os 14 jogos são ordenados por essa probabilidade.

Criar:

```text
risk_rank_top1
```

`risk_rank_top1 = 1` significa maior risco estimado de falha do Top1.

O objetivo é concentrar o máximo possível das falhas reais nos cinco primeiros lugares do ranking.

---

## Mercado x Modelo

Preservar:

```text
p_mercado(top1)
P_modelo(top1)
P_modelo_falha_top1
```

Calcular:

```text
delta_top1 = P_modelo(top1) - p(top1)
delta_top2 = P_modelo(top2) - p(top2)
delta_top3 = P_modelo(top3) - p(top3)
```

Uma diferença negativa relevante em `delta_top1` pode indicar candidato a exclusão.

---

## Score de exclusão do Top1

Criar:

```text
score_exclusao_top1
```

Pode combinar:

- `P_modelo_falha_top1`;
- entropia;
- `gap12`;
- rank;
- delta mercado-modelo;
- runs;
- soft constraints;
- penalização por favorito muito forte.

Exemplo conceitual:

```text
Score =
    w1 * P_modelo_falha_top1
  + w2 * entropia
  + w3 * score_run
  + w4 * score_rank
  + w5 * delta_mercado_modelo
  - w6 * penalidade_favorito_forte
```

Os pesos devem ser escolhidos em validação temporal.

---

## Backtest temporal

Para avaliar concurso `N`:

```text
Treino: concursos < N
Teste: concurso N
```

Preferir walk-forward:

```text
Treina até N-1 -> testa N
Treina até N   -> testa N+1
...
```

Nunca usar split aleatório misturando concursos futuros e passados.

### Janelas

Comparar:

```text
histórico inteiro
últimos 50 concursos
últimos 100 concursos
últimos 200 concursos
últimos 300 concursos
```

Também testar ponderação por recência.

---

## Backtest do volante completo

O backtest principal deve reconstruir o volante 9 secos + 5 duplos em cada concurso.

Registrar:

- pontos;
- acertos dos secos;
- acertos dos duplos;
- Top1 descartados;
- descartes corretos;
- constraints;
- score;
- oráculo;
- gap para o oráculo.

Resumo:

```text
Concursos avaliados: N
Média de pontos: ...
Mediana: ...
Desvio-padrão: ...
10+: ...%
11+: ...%
12+: ...%
13+: ...%
14:  ...%
Melhor: ...
Pior: ...
```

KPI principal:

```text
P(pontos >= 13)
```

---

## Métricas da decisão Top1

### Top1_exclusion_accuracy / Precision@5

```text
Precision@5 =
falhas reais de Top1 entre os 5 selecionados / 5
```

Na estrutura atual é equivalente conceitualmente à `Top1_exclusion_accuracy`.

### Top1_keep_accuracy

```text
Top1_keep_accuracy =
Top1 mantidos que realmente acertaram / 9
```

### Recall@5

```text
Recall@5 =
falhas reais de Top1 capturadas nos 5 descartes
/
total de falhas reais de Top1 no concurso
```

### Métricas de ranking

Quando útil:

- `NDCG@5`;
- Average Precision / MAP;
- posição média das falhas reais;
- concentração das falhas reais no Top5 de risco.

---

## Matriz de decisão Top1

```text
                         RESULTADO REAL
                    Top1 acerta | Top1 falha
------------------------------------------------
Top1 mantido            A       |     B
Top1 descartado         C       |     D
```

- `A`: manter corretamente;
- `B`: manter Top1 que falhou;
- `C`: descartar Top1 correto;
- `D`: descartar corretamente.

Objetivos:

```text
maximizar A + D
minimizar C
```

`C` é especialmente caro porque o Top1 correto foi removido do volante.

---

## Distribuição dos 5 descartes

Para cada concurso registrar:

```text
0/5 corretos
1/5
2/5
3/5
4/5
5/5
```

Relacionar com pontuação:

```text
Descartes corretos | Concursos | Média | 12+ | 13+
0/5                | ...       | ...   | ... | ...
1/5                | ...       | ...   | ... | ...
...
5/5                | ...       | ...   | ... | ...
```

Essa análise mede quanto cada melhoria na seleção dos cinco descartes vale em pontos.

---

## Avaliação por faixa e rank

### Faixas de `p(top1)`

Exemplo:

```text
33–40%
40–45%
45–50%
50–60%
60%+
```

Medir taxa real de acerto/falha em cada faixa.

### Rank Top1

```text
rank 1
rank 2
...
rank 14
```

Medir taxa histórica de falha por rank.

---

## Baselines obrigatórios

Comparar sempre no mesmo conjunto de concursos:

### A — menores `p(top1)`
Excluir os cinco menores Top1.

### B — menores `gap12`
Excluir os cinco menores `p(top1)-p(top2)`.

### C — maior `p(top2)+p(top3)`
Priorizar maior massa alternativa ao Top1.

### D — Runs
Escolha apenas por runs.

### E — ML
Escolha apenas pelo modelo.

### F — ML + Runs

### G — ML + Runs + Soft Constraints

Comparar principalmente:

```text
P(13+)
P(12+)
Precision@5
Recall@5
Top1_keep_accuracy
mean_points
```

---

## Oráculo histórico

Para cada concurso encerrado, calcular o melhor resultado matematicamente possível respeitando as mesmas hard constraints.

```text
Modelo:  11 pontos
Oráculo: 13 pontos
Gap:      2 pontos
```

O oráculo serve apenas como teto diagnóstico, nunca como informação de previsão.

---

## Calibração

Acompanhar:

- Log Loss;
- Brier Score;
- curvas de calibração.

Calibrar especialmente:

```text
P_modelo_falha_top1
```

Testar, quando aplicável:

- isotonic regression;
- Platt scaling.

---

## Ensemble Mercado + ML

Testar:

```text
P_final = w * P_mercado + (1-w) * P_ML
```

`w` deve ser escolhido em validação temporal.

Modelos possíveis:

- Logistic Regression;
- Random Forest;
- ExtraTrees;
- HistGradientBoosting;
- outros justificados pelo backtest.

---

## Ensemble de rankings

Além das probabilidades, testar combinação dos ranks:

```text
rank_ml
rank_gap12
rank_p_top1
rank_runs
rank_entropia
```

Exemplo:

```text
rank_final =
    w1 * rank_ml
  + w2 * rank_gap12
  + w3 * rank_runs
  + w4 * rank_entropia
```

---

## Penalização de favorito forte e Regret

Um Top1 muito forte deve exigir evidência adicional para ser excluído.

Testar penalização crescente conforme:

```text
p(top1)
P_modelo(top1)
```

Para descarte errado, registrar:

```text
regret_top1
```

Por exemplo:

```text
regret_top1 = p(top1)
```

Assim um erro em Top1 de 70% pesa mais que um erro em Top1 de 36%.

---

## Otimização global

Maximizar `Score_total` sujeito a:

```text
secos   = 9
duplos  = 5
triplos = 0
Top1 = 9
Top2 = 5
Top3 = 5
Flamengo sempre coberto
```

Na configuração atual, isso equivale principalmente a escolher os cinco Top1 descartados.

---

## Probabilidade coberta

Seco:

```text
coverage = P(resultado selecionado)
```

Duplo:

```text
coverage = P(A) + P(B)
```

Registrar cobertura por jogo, média e mínima do volante.

---

## Estabilidade e distância para o corte

A fronteira entre o 5º e o 6º candidato é crítica.

```text
score_rank_5
score_rank_6
cutoff_gap = score_rank_5 - score_rank_6
```

`cutoff_gap` pequeno indica decisão frágil.

Testar estabilidade sob:

- seeds diferentes;
- janelas diferentes;
- folds temporais diferentes;
- pequenas alterações de pesos;
- modelos diferentes.

### Consensus duplo

Entre soluções próximas do ótimo:

```text
Jogo 05 -> duplo em 96%
Jogo 07 -> duplo em 84%
Jogo 13 -> duplo em 51%
```

Valores próximos de 50% indicam decisão frágil.

---

## Monte Carlo

Simular resultados a partir das probabilidades estimadas para calcular:

```text
P(14)
P(13+)
P(12+)
E[pontos]
```

O Monte Carlo é complementar. O backtest histórico continua sendo a principal referência.

---

## Ablation Study

Testar remoção de grupos de features:

```text
modelo completo
sem runs
sem ranks
sem gap12
sem entropia
sem delta mercado-modelo
sem features de clube
```

Medir impacto em:

```text
P(13+)
P(12+)
Precision@5
Recall@5
Top1_keep_accuracy
mean_points
```

---

## Validação estatística

Eventos de 13+ são raros; diferenças pequenas podem ser ruído.

### Intervalos de confiança

Reportar IC para:

```text
P(13+)
P(12+)
Precision@5
mean_points
```

### Bootstrap por concurso

Reamostrar **concursos inteiros**, preservando os 14 jogos juntos.

### Comparação pareada

Como as estratégias são avaliadas nos mesmos concursos, comparar:

```text
pontos_A - pontos_B
```

concurso a concurso.

---

## Treino, validação e teste temporal

Separar:

```text
Treino temporal
      ↓
Validação temporal
      ↓
Teste final temporal intocado
```

O teste final não deve participar de:

- seleção de features;
- tuning;
- escolha de pesos;
- escolha de janelas;
- escolha de soft constraints.

### Holdout final

Reservar os concursos mais recentes e só consultá-los depois que a estratégia estiver definida.

---

## Reprodutibilidade

Registrar em cada experimento:

```text
experiment_id
timestamp
model_name
features
window_size
decay
seed
optimizer_version
constraint_version
git_commit
```

Quando houver aleatoriedade:

```python
random_state = 42
```

---

## Diagnóstico dos melhores e piores concursos

Analisar separadamente:

- 13+;
- 14 pontos;
- pior decil;
- outliers de pontuação muito baixa.

Nos 13+, avaliar:

- descartes corretos;
- ranks;
- gap12;
- entropia;
- `P_modelo_falha_top1`;
- runs;
- consenso dos duplos.

Nos piores:

- probabilidades anômalas;
- mudança de regime;
- constraint prejudicial;
- dado quebrado;
- encoding;
- comportamento fora da distribuição.

---

## Constraints por clube

Normalizar variações como:

```text
FLAMENGO/RJ
FLAMENGO-RJ
FLAMENGO RJ
```

para:

```text
FLAMENGO_RJ
```

Aplicar o mesmo princípio aos demais clubes.

---

## Teste do impacto das constraints

Comparar:

```text
sem constraint Flamengo
com constraint Flamengo

sem preferência Palmeiras
com preferência Palmeiras

9/5/5 fixo
composição alternativa
```

---

## Logging / Debugging

Evitar:

```text
JOGO 07: X2 (otimização global)
```

Preferir:

```text
[JOGO 07] VILA NOVA-GO x FORTALEZA-CE
Mercado:
  Top1 = 1  35,8%
  Top2 = 2  35,1%
  Top3 = X  29,0%

gap12:                 0,7 p.p.
rank_top1:             13/14
entropia:              ...
P_modelo_falha_top1:   ...
risk_rank_top1:        ...
delta_top1:            ...
run_top1:              ...
hit_rate_10:           ...
score_exclusao_top1:   ...
consensus_duplo:       ...

Decisão: X2
Ação: excluir Top1
```

Resumo:

```text
[RESUMO]
Secos: 9
Duplos: 5
Triplos: 0
Top1: 9
Top2: 5
Top3: 5
Constraint Flamengo: OK

TOP1 MANTIDOS:
...

TOP1 DESCARTADOS:
...
```

---

## `output/predictions.csv`

Colunas sugeridas:

```text
Concurso
Jogo
Mandante
Visitante
p(1)
p(X)
p(2)
p(top1)
p(top2)
p(top3)
rank_top1
rank_top2
rank_top3
gap12
gap23
entropia
P_modelo_top1
P_modelo_top2
P_modelo_top3
P_modelo_falha_top1
risk_rank_top1
delta_top1
delta_top2
delta_top3
score_exclusao_top1
cutoff_gap
consensus_duplo
coverage
tipo_aposta
palpite
motivo
```

Secos:

```text
1
X
2
```

Duplos:

```text
1X
12
X2
```

Triplos:

```text
1X2
```

A estratégia atual usa 0 triplos.

---

## `output/backtest_results.csv`

Uma linha por concurso:

```text
Concurso
Pontos
Acertos_secos
Acertos_duplos
Top1_descartados
Top1_descartados_corretamente
Top1_exclusion_accuracy
Top1_keep_accuracy
Precision_at_5
Recall_at_5
NDCG_at_5
Descartes_corretos_0a5
Score
risk_cutoff_gap
Atingiu_10
Atingiu_11
Atingiu_12
Atingiu_13
Atingiu_14
Oracle_points
Gap_oracle
experiment_id
model_name
seed
```

---

## Critério de sucesso

O objetivo não é maximizar accuracy jogo a jogo.

Indicador principal:

```text
P(pontos >= 13)
```

Indicadores estratégicos:

```text
Top1_exclusion_accuracy / Precision@5
Recall@5
Top1_keep_accuracy
NDCG@5
Gap_oracle
```

Também acompanhar:

- estabilidade temporal;
- calibração;
- desempenho contra baselines;
- transparência;
- ausência de leakage;
- hard constraints;
- significância das melhorias.

---

## Prioridade de implementação

1. **Baselines lado a lado no mesmo backtest.**
2. **`Top1_keep_accuracy`, Precision@5 e Recall@5.**
3. **Oráculo histórico e `Gap_oracle`.**
4. Matriz de decisão Top1 e distribuição de 0/5 a 5/5 descartes corretos.
5. Modelo binário específico para `P(Top1 falha)`.
6. `risk_rank_top1` e métricas de ranking.
7. Avaliação por rank Top1 e faixa de `p(top1)`.
8. Mercado puro x ML x ensemble mercado+ML.
9. Validação anti-leakage das runs.
10. Calibração de `P(Top1 falha)`.
11. Janelas temporais e ponderação por recência.
12. Score explícito de exclusão + penalização de favorito forte.
13. Ensemble de rankings.
14. Ablation study.
15. Otimizador e soluções quase ótimas.
16. `cutoff_gap`, estabilidade e `consensus_duplo`.
17. Monte Carlo orientado a `P(13+)`.
18. Intervalos de confiança, bootstrap e comparação pareada.
19. Treino/validação/teste temporal com holdout final.
20. Reprodutibilidade e seed fixa.
21. Diagnóstico dos concursos 13+ e piores outliers.
22. Logging detalhado.
23. Correção UTF-8.
24. Comparação final entre estratégias.
