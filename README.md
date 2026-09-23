# Projeto 1 - Machine Learning: Aprendizagem Supervisionada

## 1. Dataset
- Cidade: Hong Kong
- Fonte: https://insideairbnb.com/pt/get-the-data/
- Ficheiro: listings.csv.gz 
- Data do ficheiro: (ver a coluna last_scraped)
- Moeda do preço: HKD
- Nº de linhas e colunas: 6734 x 90

## 2. Estrutura do projeto
(a árvore de pastas, com uma linha a explicar cada ficheiro)

## 3. Como reproduzir
1. Versão do Python e bibliotecas (pandas, numpy, matplotlib, ...)
2. Onde colocar o ficheiro de dados
3. Ordem para correr os scripts/notebook
4. Onde ficam os resultados (figures/, report/)
5. Seed aleatória usada

## 4. Decisões de pré-processamento
A coluna 'price' é o preço por noite

O alvo é a coluna price (convertida de texto para número). Confirmámos que é o preço por noite.
As linhas sem preço (cerca de 10,2%) foram removidas, porque o alvo não se imputa.
As colunas 100% vazias foram ignoradas, incluindo host_response_rate, que o enunciado pede. Explica que está vazia neste ficheiro.
As colunas derivadas do preço (price_quote_total_price, price_quote_price_per_night, estimated_revenue_l365d) ficam fora das features para evitar data leakage.
O preço tem valores extremos (máximo de 171 234) e assimetria forte, o que justifica testar o log-preço.
Os missing dos review scores correspondem a alojamentos sem reviews (ausência com significado).



Removidas 688 linhas sem preço (10,2% do dataset), ficando com 6 046 alojamentos.
O preço tem cauda direita muito longa: mediana 424 HKD, percentil 95 em 2 188 HKD, percentil 99 em 4 619 HKD e máximo em 171 234 HKD. Justifica testar log-preço e tratamento de outliers.

EScolha de intervalos de noites:
Foram vistos os valores para 7 noites pois às vezes têm desconto pequeno,
28 a 30 noites pois em muitas cidades o preço cai bastante, porque é considerado long stay,
em Hong Kong, alugar por menos de 28 noites exige licença, por isso é que muitos alojamentos só apareçam disponíveis para reservas de 28 dias ou mais.


Para cada grupo de noites calculei quantos alojamentos há, o preço mediano e o preço médio

preço por noite (price), mantendo todas as durações da estadia simulada. A duração afeta o preço fortemente (mediana de 545 HKD em 1 noite vs 221 HKD em 28-31 noites), por efeito do desconto de long stay e da regulação de Hong Kong. Optou-se por manter todas as linhas e introduzir noites e um indicador de long stay como features, para o modelo separar o efeito da duração do valor do alojamento em si.


Usei o log-preço para um melhor estudo dos preços em escalas mais pequenas.
A distribuição do preço é fortemente assimétrica à direita, o que justifica usar log-preço como alvo dos modelos lineares.
O log-preço mostra sinais de mistura de duas populações (estadias curtas e long stay), o que reforça a decisão de manter a duração como feature.


Os grupos de duração têm distribuições de log-preço claramente diferentes, e a mistura das cinco distribuições explica o padrão bimodal que aparecia no histograma agregado.
O grupo 28-31 tem um pico muito estreito em torno de log = 5 (cerca de 150 HKD), sinal de que os preços estão a ser derivados de um valor mensal.
Há alojamentos com preços em log negativo (abaixo de 1 HKD), que serão tratados na análise de outliers.

A mediana do preço cresce com accommodates de forma não linear, o que justifica termos polinomiais na regressão.
Existe efeito espacial: os anúncios concentram-se em Kowloon e ilha central de Hong Kong, com padrões diferentes por bairro (já visto na tabela).
Há valores extremos em ambos os lados (0,1 HKD a 170 000 HKD), que exigem tratamento explícito.

Os outliers de preço baixo estão associados a estadias longas (240-365 noites) e sugerem erros de listagem ou valores parciais.
Os outliers de preço alto estão associados a estadias curtas e concentram-se em Entire home, alguns com alta capacidade.
Ambos os critérios(MAD e IQR) concordam bem nos preços altos, mas divergem nos baixos, onde o IQR é mais sensível. A união dos dois foi escolhida como estratégia de filtragem robusta.

Outliers de preço concentram-se em Entire home/apt (29 de 40, taxa 1,31%). Bairros com poucos anúncios têm taxas de outlier elevadas, mas apoiadas em amostras insuficientes.
minimum_nights tem distribuição bimodal (1 noite ou 28-31 noites), reflexo do mercado turístico e da regulação de estadias curtas.
accommodates está bem comportada, sem valores implausíveis.

minimum_nights não foi tratada como outlier. O valor máximo (365 noites) é elevado mas plausível, sem sinais de valores sentinela típicos de erros de listagem (por exemplo, 9 999). A distribuição bimodal (concentrada em 1 e em 28-31) reflete o mercado turístico e a regulação de estadias curtas em Hong Kong, e é informação útil para o modelo.

Estratégia de missing values:

Colunas 100% vazias eliminadas (14 colunas, incluindo host_response_rate, host_response_time, host_acceptance_rate, neighbourhood, license, instant_bookable). São limitações do ficheiro do Inside Airbnb desta versão de Hong Kong; o enunciado esperava algumas destas, mas não estão disponíveis.
Colunas de reviews (45%, ausência com significado): criada feature binária tem_reviews; os scores nulos foram preenchidos com a mediana dos alojamentos que têm reviews.
bedrooms (44,8%, valor desconhecido): comparadas duas políticas de imputação — mediana global (simples) vs mediana por grupo room_type × accommodates (agrupada).
bathrooms (11%): primeiro extraído o valor numérico da coluna bathrooms_text (quase completa) para reduzir o missing; o restante segue as mesmas duas políticas de imputação.
beds (4,9%): imputação, seguindo as duas políticas.
host_about, host_location, description, picture_url: eliminadas por não terem valor preditivo direto para o preço do alojamento.


A coluna amenities foi convertida em duas famílias de features: 11 contagens por categoria (10 categorias temáticas mais uma "outras") e 6 binárias específicas para amenities de luxo.
As categorias foram atribuídas de forma exclusiva, com prioridade para luxo (para não confundir uma "Rooftop pool" com uma amenity básica).
Cerca de 3 amenities por alojamento não se encaixam nas 10 categorias, o que é aceitável para o objetivo do projeto.
As binárias de luxo aparecem em 1-5% dos alojamentos, o suficiente para o modelo captar o efeito.
## 5. Resultados principais
(tabelas e conclusões finais, remetendo para o relatório)

## 6. Limitações