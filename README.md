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

Se o resultado final for vitória do visitante:

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

### Consequência estrutural das constraints

Com 9 secos e 5 duplos existem exatamente 19 marcações no volante.

Como também são exigidos:

```text
9 Top1 + 5 Top2 + 5 Top3 = 19 marcações
```

a estrutura natural do volante passa a ser:

```text
9 jogos: Top1 seco
5 jogos: Top2 + Top3
```

Portanto, a principal decisão estratégica do sistema é:

> **Quais são os cinco jogos em que o Top1 deve ser descartado?**

O ML, as runs, os ranks e as demais features devem ser avaliados principalmente pela capacidade de identificar esses cinco jogos.

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
├── predictions.csv
└── backtest_results.csv
```

### Responsabilidades sugeridas

#### `scripts/common.py`

- normalização dos nomes dos clubes;
- cálculo/reconstrução de `top1`, `top2` e `top3`;
- desempate `1 > 2 > X`;
- validações dos dados;
- aplicação das hard/soft constraints;
- utilitários de logging;
- configuração UTF-8 para terminal quando necessário.

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
- avaliação por faixas de probabilidade e ranks;
- geração das métricas de avaliação.

#### `scripts/predict_results.py`

- aplicação do modelo ao próximo concurso;
- cálculo dos scores por jogo;
- score de exclusão do Top1;
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
- preferencialmente encoding UTF-8.

Exemplo:

```python
pd.read_csv(
    arquivo,
    sep=";",
    decimal=",",
    encoding="utf-8"
)
```

No Windows, para evitar caracteres como `FRAN�A`, `ATL�TICO` ou `S�O PAULO`, o programa deve preferencialmente configurar também a saída do terminal:

```python
import sys

if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8")
    sys.stderr.reconfigure(encoding="utf-8")
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

```text
rank_top1
rank_top2
rank_top3
```

Os ranks devem ser calculados dentro de cada concurso, ordenando pelas respectivas probabilidades de forma decrescente.

### Incerteza do jogo

```text
gap12   = p(top1) - p(top2)
gap23   = p(top2) - p(top3)
ratio12 = p(top1) / p(top2)
```

Também avaliar:

- entropia das três probabilidades;
- concentração das probabilidades;
- distância de `p(top1)` para `1/3`;
- diferença entre favorito e segunda opção;
- percentil de `p(top1)` dentro do concurso.

A entropia pode ser calculada por:

```text
H = -Σ p(i) * log(p(i))
```

Jogos mais equilibrados tendem a apresentar maior entropia.

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
- distância desde o último erro;
- quantidade de observações usada em cada estimativa.

Exemplo:

```text
Concurso N
rank_top1 = 4
run_acertos_top1 = 3
hit_rate_10_top1 = 0,70
```

### Suavização

Taxas calculadas sobre amostras pequenas devem ser suavizadas para evitar excesso de confiança.

Uma possibilidade:

```text
p_ajustada = (acertos + alpha) / (n + alpha + beta)
```

Evitar assumir que uma sequência longa necessariamente implica reversão no próximo concurso. Essa hipótese deve ser comprovada ou rejeitada pelo backtest.

### Regra anti-leakage

Para prever o concurso `N`:

```text
feature do concurso N = histórico disponível somente até N-1
```

Nenhuma run, hit rate ou feature histórica pode usar o resultado do próprio concurso que está sendo previsto.

---

## Alvos do modelo

Além de estudar diretamente `1/X/2`, testar modelos auxiliares para estimar:

```text
P(resultado == top1)
P(resultado == top2)
P(resultado == top3)
```

O alvo mais importante para a estrutura atual é:

```text
P(Top1 acerta)
```

ou, equivalentemente:

```text
P_falha_top1 = 1 - P(Top1 acerta)
```

Essa probabilidade pode alimentar diretamente a decisão de quais cinco Top1 eliminar.

---

## Mercado x Modelo

O sistema deve preservar separadamente:

```text
p_mercado(top1)
P_modelo(top1)
```

E calcular diferenças como:

```text
delta_top1 = P_modelo(top1) - p(top1)
delta_top2 = P_modelo(top2) - p(top2)
delta_top3 = P_modelo(top3) - p(top3)
```

Exemplo:

```text
p(top1) mercado = 0,52
P_modelo(top1)  = 0,39
delta_top1      = -0,13
```

Uma diferença negativa relevante pode indicar um candidato interessante para excluir o Top1.

---

## Score de exclusão do Top1

Criar uma métrica explícita e auditável, por exemplo:

```text
score_exclusao_top1
```

Esse score pode combinar:

- `1 - P_modelo(top1)`;
- entropia;
- `gap12`;
- rank do Top1;
- delta entre modelo e mercado;
- métricas de runs;
- soft constraints.

Exemplo conceitual:

```text
Score =
    w1 * (1 - P_modelo_top1)
  + w2 * entropia
  + w3 * score_run
  + w4 * score_rank
  + w5 * delta_mercado_modelo
```

Os pesos não devem ser escolhidos arbitrariamente: devem ser avaliados por backtest temporal.

---

## Backtest

### Regra fundamental

O backtest deve ser **temporal**.

Para avaliar um concurso `N`:

```text
Treino: concursos < N
Teste : concurso N
```

Nunca usar `train_test_split` aleatório sobre linhas de vários concursos.

### Walk-forward

```text
Treina até N-1 -> testa N
Treina até N   -> testa N+1
Treina até N+1 -> testa N+2
...
```

### Janelas temporais

Comparar também:

```text
histórico inteiro
últimos 50 concursos
últimos 100 concursos
últimos 200 concursos
últimos 300 concursos
```

Outra alternativa é atribuir peso maior aos concursos recentes, por exemplo com decaimento exponencial.

---

## Backtest do volante completo

`top_accuracy` e `log_loss` são métricas auxiliares. O backtest principal deve reconstruir o **volante completo de 9 secos + 5 duplos** para cada concurso histórico.

Registrar:

- concurso;
- pontos obtidos;
- acertos dos secos;
- acertos dos duplos;
- Top1 descartados;
- quantos Top1 descartados realmente falharam;
- cumprimento das constraints;
- score do volante.

Resumo esperado:

```text
Concursos avaliados: N
Média de pontos: ...
Mediana: ...
Desvio-padrão: ...

10+ pontos: ...%
11+ pontos: ...%
12+ pontos: ...%
13+ pontos: ...%
14 pontos: ...%

Melhor: ...
Pior: ...
```

O KPI principal é:

```text
P(pontos >= 13)
```

---

## Métrica de qualidade dos Top1 descartados

Criar uma métrica específica:

```text
Top1_exclusion_accuracy
```

Exemplo:

```text
Top1 descartados: 5
Top1 que realmente falharam: 4
Top1_exclusion_accuracy = 80%
```

Essa métrica mede diretamente a qualidade da principal decisão estratégica do sistema.

Também acompanhar:

```text
Top1 mantidos que acertaram
Top1 mantidos que falharam
Top1 descartados que acertaram
Top1 descartados que falharam
```

---

## Avaliação por faixa de probabilidade

Medir a taxa real de acerto de Top1 por faixas, por exemplo:

```text
33–40%
40–45%
45–50%
50–60%
60%+
```

Exemplo de análise:

```text
Faixa p(top1): 40–45%
Acerto observado: 36%
Quantidade: 430 jogos
```

Isso ajuda a identificar regiões onde as probabilidades podem estar mal calibradas.

---

## Avaliação por rank

Medir separadamente a taxa histórica de acerto de cada posição:

```text
rank_top1 = 1
rank_top1 = 2
...
rank_top1 = 14
```

Exemplo:

```text
Rank 1: 74%
Rank 2: 69%
...
Rank 14: 35%
```

O mesmo pode ser feito para `top2` e `top3`.

---

## Baselines obrigatórios

Toda abordagem de ML deve ser comparada com estratégias simples.

### Baseline A — 5 menores `p(top1)`

Excluir Top1 nos cinco jogos de menor probabilidade Top1.

### Baseline B — 5 menores `gap12`

Excluir Top1 nos cinco jogos com menor:

```text
p(top1) - p(top2)
```

### Baseline C — maior massa Top2 + Top3

Priorizar os cinco jogos com maior:

```text
p(top2) + p(top3)
```

### Baseline D — Runs

Escolher os cinco descartes somente por métricas históricas de runs.

### Baseline E — ML

Escolher pelos scores previstos pelo modelo.

### Baseline F — ML + Runs

Combinar modelo e histórico de runs.

### Baseline G — ML + Runs + Soft Constraints

Adicionar as preferências estratégicas, incluindo Palmeiras.

A comparação principal deve ser feita por frequência de **13+ pontos** e pela `Top1_exclusion_accuracy`.

---

## Calibração das probabilidades

A qualidade da probabilidade importa tanto quanto o ranking.

Acompanhar:

- Log Loss;
- Brier Score;
- curvas de calibração.

Testar, quando aplicável:

- calibração isotônica;
- Platt scaling;
- outras técnicas adequadas ao modelo.

Top1, Top2 e Top3 podem ser calibrados separadamente.

---

## Ensemble Mercado + ML

Não assumir automaticamente que o ML deve substituir as probabilidades de mercado.

Testar combinações como:

```text
P_final = w * P_mercado + (1 - w) * P_ML
```

O peso `w` deve ser escolhido exclusivamente por desempenho em backtest temporal.

Também podem ser comparados modelos como:

- Logistic Regression;
- Random Forest;
- ExtraTrees;
- HistGradientBoosting;
- outros modelos, desde que justificados pelo backtest.

A escolha final não deve ser feita apenas por accuracy individual.

---

## Otimização global do volante

A decisão de seco/duplo não deve ser feita isoladamente para cada jogo.

O otimizador deve avaliar o conjunto dos 14 jogos e buscar a melhor combinação sujeita às constraints.

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

Na estrutura atual, isso equivale principalmente a escolher os **5 Top1 descartados**.

---

## Probabilidade coberta

Para cada jogo calcular a massa probabilística coberta pelo palpite.

Seco:

```text
coverage = P(resultado selecionado)
```

Duplo:

```text
coverage = P(resultado A) + P(resultado B)
```

Registrar:

- cobertura de cada jogo;
- cobertura média do volante;
- menor cobertura do volante.

Atenção: como os duplos atuais são Top2+Top3, um favorito muito forte deve receber penalização elevada antes de ter seu Top1 excluído.

---

## Monte Carlo

Para o próximo concurso e para concursos históricos, pode-se simular milhares de resultados usando as probabilidades estimadas.

Para cada volante, estimar:

```text
P(14)
P(13+)
P(12+)
E[pontos]
```

Exemplo:

```text
P(14)     = 0,08%
P(13+)    = 2,91%
P(12+)    = 13,80%
E[pontos] = 10,27
```

A simulação é uma ferramenta complementar. O backtest histórico real continua sendo a principal referência.

---

## Soluções próximas do ótimo e sensibilidade

Além do melhor volante, calcular internamente algumas soluções quase ótimas.

Exemplo:

```text
Melhor: score 8,371
2º:     score 8,369
3º:     score 8,362
```

Para cada jogo, medir em quantas soluções próximas do ótimo ele recebe duplo.

Exemplo:

```text
Jogo 05: duplo em 94% das soluções
Jogo 07: duplo em 81%
Jogo 13: duplo em 53%
```

Quanto mais próximo de 50%, mais frágil é a decisão daquele jogo.

---

## Oráculo histórico

Para cada concurso já encerrado, calcular o melhor resultado que seria **matematicamente possível** respeitando as mesmas hard constraints.

Exemplo:

```text
Modelo:  11 pontos
Oráculo: 13 pontos
Gap:      2 pontos
```

Isso permite separar:

- limitação estrutural das constraints;
- erro de escolha do algoritmo.

O oráculo nunca deve ser usado para previsão; serve apenas como teto diagnóstico no histórico.

---

## Constraints por clube

Os nomes dos clubes devem ser normalizados para evitar diferenças de grafia, acentos ou separadores.

Exemplos:

```text
FLAMENGO/RJ
FLAMENGO-RJ
FLAMENGO RJ
```

podem ser convertidos internamente para:

```text
FLAMENGO_RJ
```

O mesmo princípio deve ser aplicado ao Palmeiras e demais clubes.

---

## Teste do impacto das constraints

Comparar historicamente:

```text
modelo sem constraint Flamengo
modelo com constraint Flamengo

modelo sem preferência contra Palmeiras
modelo com preferência contra Palmeiras

9/5/5 fixo
composição livre/otimizada
```

Assim é possível medir o custo ou benefício de cada regra.

---

## Logging / Debugging

O terminal deve explicar **por que** cada decisão foi tomada.

Evitar logs genéricos como:

```text
JOGO 07: X2 (otimização global)
```

Preferir algo semelhante a:

```text
[JOGO 07] VILA NOVA-GO x FORTALEZA-CE
Mercado:
  Top1 = 1  35,8%
  Top2 = 2  35,1%
  Top3 = X  29,0%

gap12:          0,7 p.p.
rank_top1:      13/14
entropia:       ...

Modelo:
  P(Top1) = 32,1%
  P(Top2) = 36,7%
  P(Top3) = 31,2%

delta_top1:    -3,7 p.p.
run_top1:       ...
hit_rate_10:    ...

score_exclusao_top1: 0,782
Decisão: X2
Ação: excluir Top1
```

### Resumo do volante

```text
[RESUMO]
Secos: 9
Duplos: 5
Triplos: 0
Top1: 9
Top2: 5
Top3: 5
Constraint Flamengo: OK
```

Também mostrar explicitamente:

```text
TOP1 MANTIDOS:
01 02 03 04 06 10 11 12 14

TOP1 DESCARTADOS:
05 07 08 09 13
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
entropia
P_modelo_top1
P_modelo_top2
P_modelo_top3
delta_top1
delta_top2
delta_top3
score_exclusao_top1
coverage
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

## `output/backtest_results.csv`

Salvar uma linha por concurso histórico, contendo pelo menos:

```text
Concurso
Pontos
Acertos_secos
Acertos_duplos
Top1_descartados
Top1_descartados_corretamente
Top1_exclusion_accuracy
Score
Atingiu_10
Atingiu_11
Atingiu_12
Atingiu_13
Atingiu_14
Oracle_points
Gap_oracle
```

---

## Critério de sucesso

O objetivo não é simplesmente maximizar accuracy jogo a jogo.

O projeto deve procurar melhorar, em backtest temporal, principalmente:

```text
frequência de concursos com 13+ pontos
```

Os dois indicadores estratégicos centrais são:

```text
P(pontos >= 13)
Top1_exclusion_accuracy
```

Sem perder de vista:

- estabilidade ao longo do tempo;
- calibração das probabilidades;
- desempenho contra baselines simples;
- transparência das decisões;
- ausência de vazamento temporal;
- cumprimento das hard constraints;
- distância para o teto histórico do oráculo.

---

## Prioridade de implementação

1. **Backtest completo do volante 9 secos + 5 duplos.**
2. **Métrica `Top1_exclusion_accuracy`.**
3. Baseline dos 5 menores `gap12`.
4. Baseline dos 5 menores `p(top1)`.
5. Avaliação histórica por rank Top1.
6. Validação anti-leakage das runs.
7. Modelo específico para `P(Top1 acerta)`.
8. Comparação mercado x modelo e `delta_top1`.
9. Calibração probabilística.
10. Ensemble mercado + ML.
11. Janelas temporais e ponderação por recência.
12. Score explícito de exclusão Top1.
13. Otimizador global e análise de soluções quase ótimas.
14. Monte Carlo para estimativa de `P(13+)`.
15. Oráculo histórico e `Gap_oracle`.
16. Logging detalhado e explicação por jogo.
17. Correção automática de UTF-8 no terminal.
18. Comparação final entre todas as estratégias.
