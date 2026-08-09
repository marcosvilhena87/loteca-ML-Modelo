Implementar e/ou Aprimorar:

Minha estratégia para loteca é baseada nos históricos dos concursos de data/concursos_anteriores.csv
1) Um modelo de predição que gere um palpite único com o objetivo de fazer 13 pontos.
2) p(1)/p(x)/p(2) são as probabilidades de 1/x/2.
3) p(top1)/p(top2)/p(top3) são as probabilidades p(1)/p(x)/p(2) ordenadas da maior para a menor, desempate p(1) > p(2) > p(x)
4) One-Hot Encoding top1/top2/top3 relacionados ao resultado.
5) Logging/Debugging no terminal que auxilie na minha estratégia.

Hard Constraints:
1) 9 top1, 5 top2 e 5 top3.
2) 9 secos, 5 duplos e 0 triplos.
3) Sempre apostar a favor do FLAMENGO/RJ.

Soft Constraints:
1) Alvo: Métricas relacionadas às runs de 1, ordenando os jogos por p(top1) decrescente por concurso, 1 se top1 e 0 senão.
2) Alvo: Métricas relacionadas às runs de 1, ordenando os jogos por p(top2) decrescente por concurso, 1 se top2 e 0 senão.
3) Alvo: Métricas relacionadas às runs de 1, ordenando os jogos por p(top3) decrescente por concurso, 1 se top3 e 0 senão.
4) Se possível apostar empate OU derrota para o PALMEIRAS/SP.

Aproveitando a estrutura do repositório:

main.py
data/concursos_anteriores.csv 
data/proximo_concurso.csv

scripts/preprocess_data.py
scripts/train_model.py
scripts/predict_results.py

output/predictions.csv

- Os arquivos data/concursos_anteriores.csv e data/proximo_concurso.csv usam:
  - delimitador de colunas: ';'
  - separador decimal nas odds: ',' (vírgula)

Duplos no Palpite com o formato: 1X, 12, X2
Triplos no Palpite com o formato: 1X2
