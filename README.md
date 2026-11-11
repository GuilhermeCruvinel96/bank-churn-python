# Bank Customer Churn — Análise Exploratória em Python

Análise exploratória de uma base de 10 mil clientes de um banco europeu. O objetivo foi entender se existe um padrão por trás de quem decide sair do banco, ou se o churn é essencialmente aleatório.

A análise não partiu de um tema de negócio pré-definido. O processo foi investigativo desde o início. Cada resultado apontou a próxima pista a testar, e algumas foram descartadas pelo caminho. Esse percurso está documentado abaixo na ordem em que aconteceu.

**Base:** [Bank Customer Churn](https://www.kaggle.com/datasets/radheshyamkollipara/bank-customer-churn) (Kaggle). São 10.000 clientes de França, Alemanha e Espanha, distribuídos em 18 colunas. Os dados são fictícios.

## Dados e preparação

A base é uma tabela única, sem valores nulos em nenhuma coluna.

- **Colunas descartadas:** `RowNumber`, `CustomerId` e `Surname`, identificadores sem valor analítico.
- **Variável alvo:** `Exited` (1 = cliente saiu do banco). A taxa geral de churn é de 20,4%, usada como referência para comparar qualquer segmento.
- **Colunas auxiliares criadas durante a análise:**

  `Faixa_Etaria`, faixas de 10 em 10 anos. Substituiu uma primeira segmentação mais larga (jovem, meia-idade e sênior), descartada por concentrar quantidades muito desiguais de clientes em cada faixa.

  ```python
  df['Faixa_Etaria'] = pd.cut(df['Age'], 
                              bins=[17, 25, 35, 45, 55, 65, 92], 
                              labels=['18-25', '26-35', '36-45', '46-55', '56-65', '66-92'])
  ```

  `Saldo_Zerado`, booleana usada em vários cruzamentos ao longo da investigação.

  ```python
  df['Saldo_Zerado'] = df['Balance'] == 0
  ```

- **Peculiaridades da base:** cerca de 25% das contas têm saldo zero, concentradas em França e Espanha (0% na Alemanha). Não é erro de dado. É uma característica estrutural que precisou ser investigada para não distorcer comparações de saldo.

## Percurso da investigação

O notebook completo está em `notebooks/`. Abaixo estão os passos principais, na ordem em que aconteceram, com os trechos de código que sustentam cada achado.

### Uma pista forte demais

No primeiro `.describe()`, a taxa de churn (20,4%) e a taxa de reclamação (20,4%) apareceram muito próximas. O passo seguinte foi checar a taxa de churn entre quem reclamou e quem não reclamou.

```python
df.groupby('Complain')['Exited'].mean()
```

Entre os clientes que reclamaram, 99,5% saíram do banco. Entre os que não reclamaram, só 0,05%. O achado foi tratado como sintoma de insatisfação, não como causa. Ele mostra que o cliente chegou ao limite, mas não explica por quê. A investigação seguiu em busca do que leva o cliente até esse ponto.

### Varredura geral

Com essa ressalva, o próximo passo foi comparar as médias das variáveis entre quem saiu e quem ficou.

```python
df.groupby('Exited')[['Age', 'Balance', 'NumOfProducts', 'IsActiveMember', 'Tenure',
                      'CreditScore', 'EstimatedSalary', 'HasCrCard',
                      'Satisfaction Score']].mean().T
```

`IsActiveMember` foi o maior destaque, com média de 0,36 entre quem saiu contra 0,55 entre quem ficou. `Balance` também chamou atenção, com saldo médio de 91 mil entre quem saiu contra 73 mil entre quem ficou. As demais variáveis pareciam neutras nessa granularidade, o que mais tarde se mostrou enganoso no caso de `NumOfProducts`.

### Idade: um padrão em formato de sino

Como quem ficou no banco tinha idade média menor, a hipótese foi que clientes mais novos teriam saldo menor e, por consequência, churn menor. Um primeiro corte em três faixas largas (jovem, meia-idade e sênior) mostrou que o saldo médio era estável entre as idades. A diferença de saldo vinha do churn, não da idade. Mas as três faixas tinham quantidades de clientes muito desiguais, o que levou ao refinamento em faixas de 10 anos.

```python
resumo = df.groupby('Faixa_Etaria').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)

resumo['Pct_do_Total_de_Saidas'] = (
    resumo['Total_Saidas'] / resumo['Total_Saidas'].sum()
)

resumo
```

O churn não cresce de forma linear com a idade. Ele vai de 7,5% entre 18 e 25 anos, chega ao pico de 50,6% entre 46 e 55, e volta a cair para 13,3% acima de 66. A faixa de 46 a 55 anos reúne 1.311 clientes, cerca de 13% da base, mas responde por 32,5% de todas as saídas.

Cruzando as faixas com `Exited`, dois padrões se repetiram em todas as idades, sem exceção.

```python
df.groupby(['Faixa_Etaria', 'Exited'])[
    ['Balance', 'NumOfProducts', 'IsActiveMember',
     'Tenure', 'CreditScore', 'EstimatedSalary',
     'HasCrCard', 'Satisfaction Score']
].mean()
```

`Balance` foi sempre mais alto entre quem saiu. `IsActiveMember` foi sempre mais baixo, variando de 0,30 a 0,43 entre quem saiu contra 0,51 a 0,96 entre quem ficou.

### Alemanha: um efeito de país, não de idade

O passo seguinte foi verificar se o churn se distribuía igualmente entre os três países.

```python
resumo_geo = df.groupby('Geography').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)

resumo_geo['Pct_do_Total_de_Saidas'] = (
    resumo_geo['Total_Saidas'] / resumo_geo['Total_Saidas'].sum()
)

resumo_geo
```

A Alemanha tem churn de 32,4%, o dobro de França (16,2%) e Espanha (16,7%). A explicação mais óbvia seria uma base alemã mais velha, concentrada na faixa de pico.

```python
pd.crosstab(df['Geography'], df['Faixa_Etaria'],
            normalize='index')

df.groupby(['Geography', 'Faixa_Etaria'])['Exited'].mean().unstack()
```

A composição etária dos três países é parecida. A Alemanha tem 16,1% dos clientes entre 46 e 55 anos, contra 11,9% na França e 12,6% na Espanha, diferença pequena demais para explicar o dobro de churn. E o churn alemão fica acima dos outros dois países em todas as faixas etárias. É um efeito de país, independente da idade.

Um cruzamento mais amplo de país, faixa etária e todas as variáveis revelou outro sinal, isolado aqui numa versão mais enxuta.

```python
df.groupby(['Geography', 'Exited'])[
    ['Balance', 'IsActiveMember']
].mean()
```

O saldo médio alemão fica perto de 120 mil, contra 60 a 72 mil em França e Espanha. Dentro de França e Espanha, quem saiu tem saldo maior que quem ficou. Na Alemanha, essa diferença praticamente desaparece. Os dois sinais apontavam para a mesma suspeita. A distribuição de contas com saldo zero poderia ser diferente entre os países.

### Saldo zerado: uma diferença estrutural

```python
df.groupby('Geography').apply(
    lambda x: (x['Balance'] == 0).mean()
)
```

A Alemanha não tem nenhuma conta com saldo zero. Em França e Espanha, elas são cerca de 48%. Isso explica o saldo médio alemão mais alto e a diferença de saldo por churn nos outros dois países. Antes de seguir, foi testado se conta zerada indicava um cliente menos engajado.

```python
df['Saldo_Zerado'] = df['Balance'] == 0

pd.crosstab(df['Saldo_Zerado'], df['IsActiveMember'],
            normalize='index')
```

Não indicava. A proporção de clientes ativos é praticamente igual com saldo zerado (51,8%) e sem (51,4%). O último passo foi comparar o churn dos dois grupos dentro de cada país.

```python
df.groupby(['Geography', 'Saldo_Zerado'])['Exited'].mean()
```

Na França, clientes com saldo saem a 18,3%, contra 13,9% dos que têm conta zerada. Na Espanha, 19,6% contra 13,6%. Uma hipótese, que os dados não confirmam diretamente, é que quem tem dinheiro relevante na conta tem mais motivo prático para agir diante de uma insatisfação.

### Variáveis sem sinal

Nem toda variável testada revelou algo. Depois de idade, país e saldo, foram testadas `Card Type`, `CreditScore` e `Tenure`.

```python
df.groupby('Card Type').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)

df.groupby(['Card Type', 'Exited'])['Point Earned'].mean()

pd.crosstab(df['Faixa_Etaria'], df['Card Type'],
            normalize='index')
```

Os quatro tipos de cartão têm quase o mesmo número de clientes, e o churn varia só de 19,3% a 21,8%. Os pontos acumulados ficam perto de 600 em todos os grupos, tanto entre quem saiu quanto entre quem ficou. A distribuição por faixa etária também fica próxima de 25% para cada tipo. A maior variação aparece na faixa de 66 a 92 anos, que tem só 264 clientes. O padrão sugere que a coluna foi atribuída de forma aleatória na base fictícia.

```python
df['Faixa_CreditScore'] = pd.cut(df['CreditScore'],
                                 bins=[349, 450, 550, 650, 750, 850],
                                 labels=['350-450', '451-550', '551-650',
                                         '651-750', '751-850'])

df.groupby('Faixa_CreditScore').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

`CreditScore` fica colado na média em quase todas as faixas, entre 19,2% e 21,4% de churn. A exceção é a faixa de 350 a 450, com 32,3%, mas ela reúne só 189 clientes. Amostra pequena demais para tratar como padrão.

```python
df.groupby('Tenure').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

`Tenure` varia entre 17,2% e 23% de churn, sem tendência de alta ou queda conforme o tempo de relacionamento aumenta.

### NumOfProducts: a descoberta que quase passou batido

`NumOfProducts` tinha média perto de 1,5 em todos os recortes simples, inclusive na varredura geral. Parecia irrelevante. O teste seguinte segmentou pelo valor absoluto da coluna, mais para não deixar passar do que por expectativa de achar algo.

```python
df.groupby('NumOfProducts').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

O resultado foi o oposto do esperado. Com 1 produto, o churn é de 27,7%. Com 2, cai para 7,6%. Com 3, sobe para 82,7%. E todos os 60 clientes com 4 produtos saíram. O grupo de 3 ou mais produtos foi olhado de perto primeiro.

```python
df[df['NumOfProducts'] >= 3][['Age', 'Geography', 'Balance',
                              'IsActiveMember', 'Complain',
                              'Exited']].describe()

df[df['NumOfProducts'] >= 3]['Geography'].value_counts()
```

A taxa média de reclamação no grupo é de 0,86, igual à taxa de churn. Isso só reforça a relação entre `Complain` e `Exited` já vista no início. A distribuição por país (133 na França, 120 na Alemanha, 73 na Espanha) reflete o churn alemão estruturalmente mais alto. Com 326 clientes no total, o grupo é pequeno demais para aprofundar. A discrepância relevante estava entre 1 e 2 produtos.

```python
df[df['NumOfProducts'].isin([1, 2])].groupby(['NumOfProducts', 'Exited'])[
    ['Age', 'Balance', 'IsActiveMember']
].mean()
```

No grupo de 2 produtos, o saldo médio de quem sai (90.260) é quase o dobro do de quem fica (48.721). Isso parecia contradizer a ideia de que 2 produtos retém melhor. A primeira suspeita foi o país.

```python
pd.crosstab(df['Geography'], df['NumOfProducts'])
```

Nenhum país concentra o grupo de 2 produtos de forma desproporcional. A suspeita seguinte foi a mesma que explicou o saldo alemão.

```python
df.groupby('NumOfProducts').apply(
    lambda x: (x['Balance'] == 0).mean()
)
```

O grupo de 2 produtos tem 56,6% de contas zeradas, contra 17,8% no grupo de 1 produto. Essa diferença distorce a média de saldo. Para confirmar que ela não explicava também o churn, o padrão foi segmentado pelos dois cenários de saldo.

```python
df.groupby(['Saldo_Zerado', 'NumOfProducts'])['Exited'].mean().unstack()
```

O padrão se mantém nos dois cenários. Com saldo, 2 produtos tem churn de 13,2% contra 25,6% de 1 produto. Sem saldo, 3,3% contra 37,3%. `NumOfProducts` é um fator de risco independente. O saldo zerado explicava a distorção na média, mas não o padrão de churn.

### Gender: uma última variável

Antes de fechar a investigação, uma revisão do notebook mostrou que `Gender` nunca tinha sido testada.

```python
resumo_gender = df.groupby('Gender').agg(
    Total_Clientes=('Exited', 'size'),
    Taxa_Churn=('Exited', 'mean')
)

resumo_gender['Proporcao_Base'] = (
    resumo_gender['Total_Clientes'] / resumo_gender['Total_Clientes'].sum()
)

resumo_gender = resumo_gender[['Proporcao_Base', 'Taxa_Churn']]

resumo_gender
```

Mulheres saem a 25,1%, contra 16,5% dos homens, numa base com 45,4% de mulheres. Antes de tratar isso como um gatilho novo, o efeito foi testado contra os fatores de risco já mapeados, começando por `IsActiveMember`.

```python
pd.crosstab(df['Gender'], df['IsActiveMember'], normalize='index')
```

A proporção de clientes ativos é parecida entre mulheres (50,3%) e homens (52,5%), o que descarta essa explicação. Os outros três fatores foram cruzados numa tabela só.

```python
geo = pd.crosstab(df['Geography'], df['Gender'], normalize='index')
idade = pd.crosstab(df['Faixa_Etaria'], df['Gender'], normalize='index')
produtos = pd.crosstab(df['NumOfProducts'], df['Gender'], normalize='index')

resumo_composicao = pd.concat([geo, idade, produtos],
                              keys=['Geography', 'Faixa_Etaria',
                                    'NumOfProducts'])

resumo_composicao
```

Mulheres são levemente superrepresentadas na Alemanha (47,5% da base alemã), nas faixas de 46 a 55 (49,7%) e 56 a 65 anos (49,1%) e, principalmente, nos grupos de 3 e 4 produtos (56,0% e 63,3%). Nenhum desses fatores, isolado, explica a diferença de 9 pontos. A soma dos três explica a maior parte dela. `Gender` não é um gatilho independente. É um efeito composto dos fatores de risco já confirmados.

## Síntese dos achados

O churn não é aleatório, mas também não tem uma causa única. Os fatores que se sustentaram em múltiplos recortes foram estes.

**`IsActiveMember`** foi o padrão mais consistente da investigação. Em todas as faixas etárias e nos três países, quem saiu tinha esse indicador mais baixo que quem ficou, sem exceção. A base não detalha o que define um cliente como ativo, mas o padrão sugere que manter o cliente engajado reduz a chance de saída, independente de idade ou país.

**`NumOfProducts`** tem um ponto ótimo em 2 produtos. O salto de churn a partir de 3 produtos é grande demais para ser ignorado, mesmo com uma amostra pequena. Isso sugere que empurrar o cliente para um terceiro ou quarto produto pode gerar mais insatisfação do que valor.

**A Alemanha** é um problema estrutural de país, não de composição etária. É também o único país sem contas com saldo zero. Vale investigar se essas duas características têm relação entre si ou se é coincidência de mercado.

**O saldo zerado** teve dois papéis. Explicou distorções ao longo do caminho: do saldo médio alemão ao efeito de `NumOfProducts`. E se mostrou um achado próprio, já que em França e Espanha contas zeradas têm churn menor. Para o banco, a leitura é ambígua. Uma conta zerada é mais fácil de reter, mas também gera menos receita.

**`Complain`** ficou como sintoma, não como causa. Uma relação tão próxima de 100% entre reclamar e sair parece ser um traço da base fictícia. Num cenário real, um padrão parecido indicaria que a insatisfação chega ao ponto de reclamação sem nenhuma etapa intermediária de resolução.

**`Gender`** mostrou diferença de churn, mas explicada pela composição dos fatores de risco anteriores, não como gatilho independente. **`Card Type`**, **`CreditScore`** e **`Tenure`** não mostraram sinal relevante.

No fim, o churn se concentra em clientes menos engajados, num número específico de produtos contratados e num país com comportamento estrutural diferente dos demais, mais do que em qualquer característica isolada de perfil ou de crédito.

## Considerações técnicas

- **Tabelas largas em formato vertical.** Tabelas com muitas colunas ficam ilegíveis na largura de um artigo. Em tabelas de índice simples, como o `describe()`, a solução foi transpor com `.T`. Em tabelas com índice duplo, transpor cria um cabeçalho confuso, então a solução foi reduzir as colunas às que o texto cita.

- **Arredondamento em escalas diferentes.** No `describe()` transposto, colunas como `Balance` (milhares) e `Exited` (proporção) dividem a mesma tabela. Um `.round(2)` igualaria `Exited` (0,2038) e `Complain` (0,2044). Por isso foi usado `.round(4)`.

- **`.unstack()` para comparar dois eixos.** Resultados de `groupby` com dois níveis saem como lista vertical. O `.unstack()` transforma em tabela cruzada, o que deixa padrões como "a Alemanha fica acima em todas as faixas" visíveis de imediato.

- **Caminho relativo.** O CSV é lido com caminho relativo (`../data/Customer-Churn-Records.csv`), para que o notebook rode em qualquer máquina que clone o repositório.

## Estrutura do repositório

```
bank-churn-python/
├── data/
│   └── Customer-Churn-Records.csv
├── notebooks/
│   └── analise_churn.ipynb
├── .gitignore
└── README.md
```

## Como rodar

```bash
python -m venv venv
venv\Scripts\activate
pip install pandas jupyter
```

Depois, abrir `notebooks/analise_churn.ipynb` e executar as células em ordem.

## Ferramentas utilizadas

- Python 3.12
- Pandas
- Jupyter Notebook (no VS Code)

## Links

- [Artigo completo no LinkedIn](https://www.linkedin.com/pulse/an%C3%A1lise-explorat%C3%B3ria-em-python-investigando-o-perfil-de-cruvinel-pes6f/)

---

Confira outros projetos do meu portfólio no [GitHub](https://github.com/GuilhermeCruvinel96).
