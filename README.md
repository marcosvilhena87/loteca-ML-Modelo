# Loteca ML — Baseline e Laboratório de Estratégias

Projeto para experimentar modelos de predição e estratégias de montagem de um **palpite único da Loteca**, usando o histórico de concursos e as probabilidades estimadas para os resultados `1`, `X` e `2`.

O objetivo do repositório é servir como uma base limpa para comparar diferentes:

- modelos de predição;
- representações das probabilidades;
- formas de ordenação dos jogos;
- métricas de runs de acertos;
- funções-objetivo;
- estratégias de escolha de secos e duplos.

---

## 1. Estrutura do repositório

```text
loteca-ML-Default/
├── main.py
├── data/
│   ├── concursos_anteriores.csv
│   └── proximo_concurso.csv
├── models/
├── output/
│   └── predictions.csv
└── scripts/
    ├── common.py
    ├── preprocess_data.py
    ├── train_model.py
    └── predict_results.py
```

### Fluxo sugerido

```text
main.py
   ↓
preprocess_data.py
   ↓
train_model.py
   ↓
predict_results.py
   ↓
output/predictions.csv
```

---

## 2. Dados

Os arquivos:

```text
data/concursos_anteriores.csv
data/proximo_concurso.csv
```

utilizam:

- delimitador de colunas: `;`
- separador decimal nas odds: `,`

---

## 3. Probabilidades básicas

O modelo deve estimar:

```text
p(1)  = probabilidade de vitória do mandante
p(x)  = probabilidade de empate
p(2)  = probabilidade de vitória do visitante
```

---

## 4. Ranking das probabilidades

Para cada jogo:

```text
p(top1) = maior entre p(1), p(x), p(2)
p(top2) = segunda maior
p(top3) = menor
```

Em caso de empate entre probabilidades, usar o desempate:

```text
1 > 2 > X
```

Também registrar o resultado correspondente a cada posição:

```text
top1_result
top2_result
top3_result
```

Exemplo:

```text
p(1) = 0,52
p(x) = 0,28
p(2) = 0,20

top1_result = 1
top2_result = X
top3_result = 2
```

---

## 5. One-Hot Encoding do resultado observado

Criar indicadores relacionados ao resultado real:

```text
hit_top1 = 1 se o resultado real foi top1, senão 0
hit_top2 = 1 se o resultado real foi top2, senão 0
hit_top3 = 1 se o resultado real foi top3, senão 0
```

Esses indicadores serão usados principalmente nas análises históricas de ordenação e runs.

---

# 6. Restrições da estratégia

## Hard Constraints

A solução final deve respeitar:

```text
9 top1
5 top2
5 top3
```

e:

```text
9 secos
5 duplos
0 triplos
```

Formato dos duplos:

```text
1X
12
X2
```

### Flamengo

Sempre apostar a favor do:

```text
FLAMENGO/RJ
```

---

## Soft Constraints

### Palmeiras

Se possível:

```text
apostar empate OU derrota para o PALMEIRAS/SP
```

Essa preferência não deve violar as Hard Constraints.

---

# 7. Distribuições de três colunas

Além da representação tradicional:

```text
p(1) | p(x) | p(2)
```

testar outras representações sempre com três colunas.

## 7.1 Ranking

```text
p(top1) | p(top2) | p(top3)
```

## 7.2 Margens

```text
p(top1)-p(top2)
p(top2)-p(top3)
p(top1)-p(top3)
```

## 7.3 Diferenças entre 1 / X / 2

```text
p(1)-p(x)
p(1)-p(2)
p(x)-p(2)
```

## 7.4 Razões

```text
p(1)/p(x)
p(1)/p(2)
p(x)/p(2)
```

ou, pelo ranking:

```text
p(top1)/p(top2)
p(top2)/p(top3)
p(top1)/p(top3)
```

É necessário proteger as divisões contra denominadores iguais ou muito próximos de zero.

## 7.5 Força do favorito e alternativas

```text
p(top1)
p(top2)+p(top3)
p(top1)-(p(top2)+p(top3))
```

## 7.6 Cobertura do duplo

```text
p(top1)
p(top1)+p(top2)
p(top3)
```

Essa distribuição é especialmente importante para a estratégia de:

```text
9 secos + 5 duplos
```

pois:

```text
p(top1) + p(top2)
```

representa a probabilidade estimada coberta pelo duplo formado pelas duas alternativas mais prováveis.

## 7.7 Duplo contra terceiro resultado

```text
p(top1)+p(top2)
p(top3)
p(top2)-p(top3)
```

---

# 8. Formas de ordenação dos jogos

Para cada concurso, ordenar os 14 jogos por diferentes critérios e estudar a sequência de acertos resultante.

## Baseline

### Ordenação A

```text
p(top1) decrescente
```

Sequência:

```text
1 se resultado real = top1
0 caso contrário
```

### Ordenação B

```text
p(top2) decrescente
```

Sequência:

```text
1 se resultado real = top2
0 caso contrário
```

### Ordenação C

```text
p(top3) decrescente
```

Sequência:

```text
1 se resultado real = top3
0 caso contrário
```

---

## Novas ordenações sugeridas

### 8.1 Margem top1-top2

```text
p(top1) - p(top2)
```

Testar em ordem:

```text
decrescente
crescente
```

Interpretação:

- margem alta → favorito mais destacado;
- margem baixa → maior equilíbrio entre as duas primeiras alternativas.

---

### 8.2 Margem top2-top3

```text
p(top2) - p(top3)
```

Pode ajudar a identificar quando um duplo top1+top2 está bem separado do resultado menos provável.

---

### 8.3 Margem top1-top3

```text
p(top1) - p(top3)
```

Mede o grau geral de desigualdade da distribuição.

---

### 8.4 Cobertura do duplo

```text
p(top1) + p(top2)
```

Ordenar de forma decrescente.

Quanto maior o valor, menor tende a ser o risco residual:

```text
p(top3)
```

---

### 8.5 Risco residual do duplo

```text
p(top3)
```

Testar especialmente em ordem crescente.

---

### 8.6 Incerteza do favorito

```text
p(top1)
```

Além da ordem decrescente, testar:

```text
p(top1) crescente
```

Isso coloca primeiro os jogos mais equilibrados.

---

### 8.7 Razão top1/top2

```text
p(top1) / p(top2)
```

Quanto maior a razão, maior a dominância relativa de top1 sobre top2.

---

### 8.8 Razão top2/top3

```text
p(top2) / p(top3)
```

Pode ajudar a identificar jogos em que top2 é uma alternativa significativamente mais forte que top3.

---

### 8.9 Entropia

Calcular:

```text
H = - Σ p(i) * log(p(i))
```

para:

```text
i ∈ {1, X, 2}
```

Testar:

```text
entropia crescente
entropia decrescente
```

Interpretação:

- entropia baixa → distribuição concentrada;
- entropia alta → jogo mais equilibrado.

---

### 8.10 Desvio padrão das probabilidades

```text
std(p(1), p(x), p(2))
```

Testar em ordem decrescente e crescente.

---

### 8.11 Distância da distribuição uniforme

Comparar cada jogo com:

```text
(1/3, 1/3, 1/3)
```

Quanto maior a distância, mais desigual é a distribuição estimada.

---

### 8.12 Score favorito × margem

```text
p(top1) * (p(top1) - p(top2))
```

Combina:

- confiança absoluta no favorito;
- distância em relação à segunda alternativa.

---

### 8.13 Score de duplo

Uma possibilidade:

```text
(p(top1) + p(top2)) - p(top3)
```

O objetivo é destacar jogos em que dois resultados dominam o terceiro.

---

# 9. Análise de runs

Para cada critério de ordenação, produzir uma sequência binária de 14 posições.

Exemplo:

```text
1 1 0 1 1 1 0 0 1 1 0 1 1 0
```

## Métricas sugeridas

Calcular:

- quantidade total de `1`;
- quantidade total de `0`;
- maior run de `1`;
- maior run de `0`;
- quantidade de runs de `1`;
- tamanho médio das runs de `1`;
- posição da primeira falha;
- posição da última falha;
- posição média dos erros;
- quantidade de acertos no início da ordenação;
- quantidade de acertos no fim da ordenação;
- acurácia acumulada por posição;
- `top-k accuracy`.

### Exemplos de top-k

```text
Top 3
Top 5
Top 7
Top 9
Top 10
Top 14
```

O objetivo não é apenas descobrir qual ordenação possui maior média de acertos, mas também se alguma delas **concentra os acertos em posições historicamente previsíveis**.

---

# 10. Objetivos de otimização

O projeto não deve ficar preso a uma única função-objetivo.

## Objetivo original

```text
Gerar um palpite único com o objetivo de fazer 13 pontos.
```

---

## 10.1 Maximizar 14 pontos

```text
max P(acertos = 14)
```

Objetivo agressivo.

---

## 10.2 Maximizar pelo menos 13 pontos

```text
max P(acertos >= 13)
```

Objetivo diretamente alinhado à meta de 13 ou 14 pontos.

---

## 10.3 Maximizar número esperado de acertos

```text
max E[acertos]
```

Funciona como baseline de precisão global.

---

## 10.4 Maximizar pelo menos 12 pontos

```text
max P(acertos >= 12)
```

Pode favorecer estratégias mais robustas.

---

## 10.5 Minimizar concursos ruins

Exemplo:

```text
min P(acertos < 11)
```

Objetivo defensivo.

---

## 10.6 Maximizar acertos dos secos

Com:

```text
9 secos
```

dar atenção específica à taxa de acerto dos resultados não protegidos por duplo.

---

## 10.7 Minimizar erros nos secos

Uma função de perda pode penalizar mais fortemente:

```text
erro em seco
```

do que:

```text
resultado não coberto por determinado duplo
```

---

## 10.8 Maximizar eficiência dos duplos

O duplo deve ser colocado onde realmente aumenta a chance de sobrevivência do cartão.

Avaliar:

```text
ganho_do_duplo = P(top1 ou top2) - P(top1)
```

que, no caso simples, corresponde à contribuição adicional de:

```text
p(top2)
```

---

## 10.9 Maximizar cobertura estimada

Para cada jogo:

### Seco

```text
P(coberto) = probabilidade do resultado escolhido
```

### Duplo

```text
P(coberto) = soma das probabilidades dos dois resultados escolhidos
```

Otimizar a cobertura global respeitando:

```text
9 secos
5 duplos
0 triplos
```

---

## 10.10 Minimizar top3 não coberto

Como não existem triplos:

```text
0 triplos
```

o resultado `top3` normalmente representa o risco residual mais difícil de proteger.

Uma função-objetivo pode tentar minimizar:

```text
Σ p(top3)
```

nos jogos selecionados para determinados tipos de cobertura.

---

## 10.11 Função de utilidade por faixa de pontos

Exemplo:

```text
score =
10 * P(14)
+ 3 * P(13)
+ 1 * P(12)
```

Os pesos devem ser tratados como hiperparâmetros e avaliados via backtest.

---

## 10.12 Maximizar estabilidade histórica

Além da média geral, avaliar:

- mediana;
- desvio padrão;
- percentis;
- pior faixa histórica;
- consistência ao longo do tempo.

A melhor estratégia não precisa ser aquela que possui somente o maior pico de desempenho.

---

# 11. Problema específico: escolher os 5 duplos

A seleção dos duplos deve ser tratada como um problema próprio.

Para cada jogo, podem ser consideradas features como:

```text
p(top1)
p(top2)
p(top3)
p(top1)-p(top2)
p(top2)-p(top3)
p(top1)+p(top2)
entropia
desvio padrão
```

Possíveis alvos:

```text
1 = jogo deveria receber duplo
0 = jogo deveria permanecer seco
```

ou um score contínuo de benefício esperado do duplo.

---

# 12. Backtest

O backtest deve respeitar a ordem temporal.

Evitar usar informação futura para prever concursos anteriores.

## Comparações mínimas

Comparar:

1. baseline por `p(top1)`;
2. estratégia 9 secos + 5 duplos;
3. diferentes ordenações;
4. diferentes funções-objetivo;
5. diferentes regras para seleção dos duplos.

## Métricas

Registrar, no mínimo:

```text
acertos
P(14)
P(13+)
P(12+)
média de acertos
mediana de acertos
desvio padrão
acerto dos secos
acerto dos duplos
top1_accuracy
top2_capture
top3_frequency
```

---

# 13. Experimentos

Cada execução deve possuir identificação própria.

Exemplo:

```text
run_id
model_name
feature_set
ordering
objective
seed
training_window
```

Isso permite comparar estratégias sem perder rastreabilidade.

---

# 14. Logging / Debugging

O terminal deve ajudar a entender por que o algoritmo tomou determinada decisão.

Exemplo:

```text
[INFO] Concurso: 1263
[INFO] Objetivo: max_p13_plus
[INFO] Estratégia: 9 secos / 5 duplos / 0 triplos
[INFO] Ordenação: p_top1_minus_p_top2_desc

[JOGO 01]
Mandante: TIME A
Visitante: TIME B
p(1): 0.52
p(X): 0.27
p(2): 0.21
top1: 1
top2: X
top3: 2
margem top1-top2: 0.25
cobertura top1+top2: 0.79
decisão: SECO 1

[JOGO 02]
...
decisão: DUPLO X2
```

---

# 15. Output

O arquivo:

```text
output/predictions.csv
```

deve permitir auditar a decisão final.

Colunas sugeridas:

```text
concurso
jogo
mandante
visitante

p_1
p_x
p_2

top1_result
top2_result
top3_result

p_top1
p_top2
p_top3

margin_top1_top2
margin_top2_top3
margin_top1_top3

double_coverage
entropy

tipo_aposta
palpite
objective_score
decision_reason
```

---

# 16. Princípio dos experimentos

As sugestões deste README devem ser tratadas como **hipóteses a testar**, não como regras garantidamente superiores.

Cada nova feature, ordenação ou função-objetivo deve provar seu valor em backtest temporal.

A pergunta principal do projeto é:

> Qual combinação de probabilidades, ordenação, objetivo e alocação de 9 secos + 5 duplos produz a melhor distribuição histórica de resultados para atingir 13 ou mais pontos?

---

# 17. Prioridade inicial sugerida

Antes de adicionar modelos excessivamente complexos, implementar e comparar:

```text
1. p(top1) decrescente
2. p(top2) decrescente
3. p(top3) decrescente
4. p(top1)-p(top2) decrescente
5. p(top2)-p(top3) decrescente
6. p(top1)+p(top2) decrescente
7. entropia crescente
8. p(top1)/p(top2) decrescente
```

e as funções-objetivo:

```text
A. max E[acertos]
B. max P(acertos >= 13)
C. max cobertura do cartão 9s-5d
```

Esses experimentos formam uma primeira matriz de comparação antes da introdução de regras mais sofisticadas.
