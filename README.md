# Loteca ML Default

Projeto para desenvolver e aprimorar uma estratégia de Loteca baseada no histórico de concursos disponível em `data/concursos_anteriores.csv`, gerando **um único palpite** para o próximo concurso com foco em maximizar a frequência de resultados de **13 pontos ou mais**.

## Objetivo

Construir um pipeline que separe claramente:

1. **Probabilidades do jogo** — `p(1)`, `p(X)` e `p(2)`.
2. **Modelo estatístico/ML** — estima a chance de acerto de `top1`, `top2` e `top3`.
3. **Otimizador do volante** — escolhe globalmente os 9 secos e 5 duplos respeitando as constraints.
4. **Backtest da estratégia completa** — mede o desempenho histórico por concurso.

Fluxo desejado:

```text
ODDS / PROBABILIDADES
        ↓
PRÉ-PROCESSAMENTO
        ↓
MODELO
estima chance real de Top1 / Top2 / Top3 acertar
        ↓
OTIMIZADOR
escolhe 9 secos + 5 duplos
respeitando as constraints
        ↓
PALPITE ÚNICO + DEBUG + BACKTEST
```

---

## Definições

- `p(1)`, `p(X)` e `p(2)` são as probabilidades de vitória do mandante, empate e vitória do visitante.
- `p(top1)`, `p(top2)` e `p(top3)` são `p(1)`, `p(X)` e `p(2)` ordenadas da maior para a menor.
- Em caso de empate entre probabilidades, usar o desempate:

```text
1 > 2 > X
```

- `top1`, `top2` e `top3` representam, via One-Hot Encoding, qual posição ordenada correspondeu ao resultado real.

Exemplo:

```text
p(1) = 0,50
p(X) = 0,25
p(2) = 0,25

p(top1) = 0,50 -> 1
p(top2) = 0,25 -> 2
p(top3) = 0,25 -> X
```

Se o resultado final for vitória do visitante, então:

```text
top1 = 0
top2 = 1
top3 = 0
```

---

## Hard Constraints

Estas regras devem ser respeitadas pelo palpite final:

1. Exatamente **9 top1, 5 top2 e 5 top3** selecionados no conjunto das apostas.
2. Exatamente **9 secos, 5 duplos e 0 triplos**.
3. Sempre apostar a favor do **FLAMENGO/RJ**.

A regra do Flamengo deve considerar sua posição no jogo:

- Flamengo mandante -> o palpite deve conter `1`.
- Flamengo visitante -> o palpite deve conter `2`.

---

## Soft Constraints

Estas regras devem influenciar a otimização, mas não precisam ser satisfeitas a qualquer custo:

1. Explorar métricas relacionadas às **runs de 1** de `top1`, ordenando os jogos por `p(top1)` decrescente dentro de cada concurso.
2. Explorar métricas relacionadas às **runs de 1** de `top2`, ordenando os jogos por `p(top2)` decrescente dentro de cada concurso.
3. Explorar métricas relacionadas às **runs de 1** de `top3`, ordenando os jogos por `p(top3)` decrescente dentro de cada concurso.
4. Se possível, apostar em **empate ou derrota do PALMEIRAS/SP**.

A preferência relacionada ao Palmeiras deve funcionar como bônus/penalidade no score do otimizador, e não como regra absoluta.

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
└── predictions.csv
```

### Responsabilidades sugeridas

#### `scripts/common.py`

Funções compartilhadas:

- normalização dos nomes dos clubes;
- cálculo/reconstrução de `top1`, `top2` e `top3`;
- desempate `1 > 2 > X`;
- validações dos dados;
- aplicação das hard/soft constraints;
- utilitários de logging.

#### `scripts/preprocess_data.py`

- leitura dos CSVs;
- conversão das probabilidades;
- validação dos concursos;
- criação de features;
- geração dos alvos históricos;
- construção de ranks e métricas de runs sem vazamento temporal.

#### `scripts/train_model.py`

- treinamento dos modelos;
- backtest temporal por concurso;
- calibração das probabilidades;
- comparação com baselines;
- geração das métricas de avaliação.

#### `scripts/predict_results.py`

- aplicação do modelo ao próximo concurso;
- cálculo dos scores por jogo;
- otimização global dos 9 secos e 5 duplos;
- aplicação das hard/soft constraints;
- geração de `output/predictions.csv`;
- logging detalhado das decisões.

#### `main.py`

Executa o pipeline completo:

```text
preprocessar -> treinar/backtest -> prever -> otimizar -> salvar resultado
```

---

## Formato dos dados

Os arquivos:

```text
data/concursos_anteriores.csv
data/proximo_concurso.csv
```

usam:

- delimitador de colunas: `;`
- separador decimal: `,`

Exemplo de leitura com pandas:

```python
pd.read_csv(arquivo, sep=";", decimal=",")
```

---

## Validações obrigatórias dos CSVs

Antes do treinamento ou da previsão, validar:

- exatamente 14 jogos por concurso;
- ausência de duplicidade em `Concurso + Jogo`;
- probabilidades entre 0 e 1;
- `p(1) + p(X) + p(2)` aproximadamente igual a 1;
- `p(top1) >= p(top2) >= p(top3)`;
- consistência entre os tops e `p(1)/p(X)/p(2)`;
- resultado histórico com One-Hot válido;
- concurso futuro sem resultado real preenchido;
- nomes de clubes normalizados para aplicação das constraints.

Sempre que possível, `top1/top2/top3` devem ser **recalculados a partir de `p(1)/p(X)/p(2)`**, em vez de serem aceitos cegamente do arquivo.

---

## Features sugeridas

### Probabilidades básicas

```text
p(1)
p(X)
p(2)
p(top1)
p(top2)
p(top3)
```

### Ranks dentro do concurso

Para cada jogo:

```text
rank_top1
rank_top2
rank_top3
```

Os ranks devem ser calculados dentro de cada concurso, ordenando pelas respectivas probabilidades de forma decrescente.

### Incerteza do jogo

Criar, entre outras:

```text
gap12 = p(top1) - p(top2)
gap23 = p(top2) - p(top3)
ratio12 = p(top1) / p(top2)
```

Também avaliar:

- entropia das três probabilidades;
- concentração das probabilidades;
- distância de `p(top1)` para `1/3`;
- diferença entre favorito e segunda opção.

Jogos com menor `gap12`, por exemplo, tendem a ser candidatos naturais a receber duplo.

---

## Runs

As runs devem ser tratadas como **features históricas**, e não como regra automática de reversão.

Para cada rank de `top1`, `top2` e `top3`, calcular apenas com informações disponíveis até o concurso anterior:

- run atual de acertos;
- run atual de erros;
- maior run histórica de acertos;
- maior run histórica de erros;
- taxa de acerto nas últimas 5 observações;
- taxa de acerto nas últimas 10 observações;
- taxa de acerto nas últimas 20 observações;
- taxa de acerto nas últimas 50 observações;
- distância desde o último acerto;
- distância desde o último erro.

Exemplo conceitual:

```text
Concurso N
rank_top1 = 4
run_acertos_top1 = 3
hit_rate_10_top1 = 0,70
```

Evitar assumir que uma sequência longa necessariamente implica reversão no próximo concurso. Essa hipótese deve ser comprovada ou rejeitada pelo backtest.

---

## Alvos do modelo

Além de estudar diretamente `1/X/2`, testar modelos auxiliares para estimar:

```text
P(resultado == top1)
P(resultado == top2)
P(resultado == top3)
```

Esses alvos são especialmente úteis porque o volante final trabalha diretamente com Top1, Top2 e Top3.

---

## Backtest

### Regra fundamental

O backtest deve ser **temporal**.

Para avaliar um concurso `N`:

```text
Treino: concursos < N
Teste : concurso N
```

Nunca usar `train_test_split` aleatório sobre linhas de vários concursos, pois isso pode provocar vazamento temporal.

### Walk-forward

Preferencialmente usar validação walk-forward:

```text
Treina até N-1 -> testa N
Treina até N   -> testa N+1
Treina até N+1 -> testa N+2
...
```

---

## Métricas principais

A métrica central deve ser o desempenho **por volante/concurso**, e não apenas accuracy por partida.

Registrar:

- média de pontos;
- mediana de pontos;
- desvio-padrão;
- pior resultado;
- melhor resultado;
- frequência de 10+ pontos;
- frequência de 11+ pontos;
- frequência de 12+ pontos;
- frequência de 13+ pontos;
- frequência de 14 pontos.

Especial atenção para:

```text
P(pontos >= 13)
```

Além disso, para os modelos probabilísticos, acompanhar:

- Log Loss;
- Brier Score;
- curvas de calibração.

---

## Baselines obrigatórios

O modelo de ML deve ser comparado com estratégias simples.

### Baseline 1 — Sempre Top1

Usar o resultado de maior probabilidade em todos os jogos.

### Baseline 2 — Menor confiança

Usar Top1 como seco e transformar em duplo os cinco jogos com menor `p(top1)`.

### Baseline 3 — Menor gap

Usar duplo nos cinco jogos com menor:

```text
p(top1) - p(top2)
```

### Baseline 4 — Runs

Estratégia baseada apenas nas métricas históricas de runs.

Uma nova abordagem só deve ser considerada melhoria se superar consistentemente essas referências no backtest temporal.

---

## Calibração das probabilidades

A qualidade da probabilidade importa tanto quanto o ranking.

Se o sistema informa probabilidade de aproximadamente 70% para um grupo de eventos, o acerto observado nesse grupo deveria ficar próximo de 70%.

Testar, quando aplicável:

- calibração isotônica;
- Platt scaling;
- outras técnicas de calibração adequadas ao modelo escolhido.

---

## Otimização global do volante

A decisão de seco/duplo não deve ser feita isoladamente para cada jogo.

O otimizador deve avaliar o conjunto dos 14 jogos e buscar a melhor combinação sujeita às constraints.

Conceitualmente:

```text
maximizar Score_total
```

sujeito a:

```text
secos   = 9
duplos  = 5
triplos = 0

top1 selecionados = 9
top2 selecionados = 5
top3 selecionados = 5

FLAMENGO sempre coberto
```

A preferência contra o Palmeiras pode ser incorporada como um bônus no score quando o palpite incluir empate ou derrota do clube.

---

## Escolha dos duplos

Para cada jogo, calcular o ganho marginal estimado de transformar um seco em duplo.

Exemplo:

```text
p(top1) = 0,44
p(top2) = 0,34

Cobertura com seco  = 0,44
Cobertura com duplo = 0,78
Ganho marginal      = +0,34
```

Os cinco duplos devem ser alocados considerando esse ganho, o modelo, as runs e as constraints globais.

---

## Constraints por clube

Os nomes dos clubes devem ser normalizados para evitar diferenças de grafia, acentos ou separadores.

Exemplos possíveis:

```text
FLAMENGO/RJ
FLAMENGO-RJ
FLAMENGO RJ
```

podem ser convertidos internamente para:

```text
FLAMENGO_RJ
```

O mesmo princípio deve ser aplicado ao Palmeiras e aos demais clubes quando necessário.

---

## Teste do impacto das constraints

O backtest deve medir o efeito individual das regras.

Comparar, por exemplo:

```text
modelo sem constraint do Flamengo
modelo com constraint do Flamengo

modelo sem preferência contra Palmeiras
modelo com preferência contra Palmeiras

9/5/5 fixo
composição livre/otimizada
```

Assim será possível medir quanto cada regra melhora ou reduz o desempenho histórico.

---

## Logging / Debugging

O terminal deve explicar como o palpite foi formado.

Exemplo:

```text
[INFO] Concurso: 1263
[INFO] Jogos carregados: 14
[INFO] Hard constraints: OK

[JOGO 09] CORITIBA-PR x PALMEIRAS-SP
p1=0.272 px=0.298 p2=0.430
top1=2 top2=X top3=1
gap12=0.132
rank_top1=...
score_duplo=...
palpite=X2
motivo=incerteza + otimização global + soft constraint Palmeiras

[RESUMO]
Secos: 9
Duplos: 5
Triplos: 0
Top1: 9
Top2: 5
Top3: 5
Constraint Flamengo: OK
```

---

## `output/predictions.csv`

O arquivo final deve ser legível e auditável.

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
score_top1
score_top2
score_top3
tipo_aposta
palpite
motivo
```

Formato dos palpites:

### Secos

```text
1
X
2
```

### Duplos

```text
1X
12
X2
```

### Triplos

```text
1X2
```

Embora o formato de triplo seja suportado, a estratégia atual determina **0 triplos**.

---

## Critério de sucesso

O objetivo não é simplesmente maximizar a taxa de acerto jogo a jogo.

O projeto deve procurar melhorar, em backtest temporal, principalmente:

```text
frequência de concursos com 13+ pontos
```

sem perder de vista:

- estabilidade ao longo do tempo;
- calibração das probabilidades;
- desempenho contra baselines simples;
- transparência das decisões;
- ausência de vazamento temporal;
- cumprimento das hard constraints.

---

## Prioridade de implementação

1. Validação robusta dos CSVs.
2. Reconstrução confiável de Top1/Top2/Top3.
3. Baselines e backtest temporal.
4. Métricas por concurso.
5. Features de incerteza e ranks.
6. Features históricas de runs.
7. Modelos para Top1/Top2/Top3.
8. Calibração probabilística.
9. Otimizador global dos 9 secos + 5 duplos.
10. Constraints Flamengo/Palmeiras.
11. Logging e explicação por jogo.
12. Comparação final entre todas as estratégias.
