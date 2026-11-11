## LinkedIn — Artigo + Post do Feed

**Status:** Publicado (atualizado em 11/11/2026 com correções de números e link do repositório no GitHub)

**Formato:** Artigo LinkedIn (Estilo 3 — Investigativo puro) + post curto no feed linkando o artigo + post na comunidade do curso (Bruce)

**Imagens:** prints da saída de cada bloco de código, logo abaixo do código correspondente (marcados como [📎 Print] no texto)

**Capa:** ilustração gerada no Ideogram e ajustada no Canva (1920×1080) — cliente insatisfeito, de costas, se afastando de um prédio com letreiro "BANK", balão de fala com #@!%, trilha pontilhada dourada com uma lupa sobre pontos de dado. Paleta navy + dourado

**Título do artigo:** Análise exploratória em Python: investigando o perfil de quem decide sair de um banco

**Post do feed:**
> Como mapear os padrões por trás do churn? Engajamento, produtos contratados e geografia ajudam a explicar a saída de um cliente.

**Post da comunidade (grupo do curso — Bruce):**
> Boa tarde, pessoal! Acabei de publicar um artigo com uma análise exploratória em Python, buscando entender o perfil e os padrões de clientes que saíram de um banco. Não parti de uma hipótese pronta, o processo foi ir atrás do que os próprios dados indicavam, testando e descartando pistas pelo caminho. O que mais pesa na decisão de um cliente sair do banco? Convido vocês a dar uma olhada!

**Link do artigo:** https://www.linkedin.com/pulse/an%C3%A1lise-explorat%C3%B3ria-em-python-investigando-o-perfil-de-cruvinel-pes6f/

**Repositório:** https://github.com/GuilhermeCruvinel96/bank-churn-python

---

### Texto final publicado

Bancos perdem clientes o tempo todo, mas nem sempre fica claro por quê. Existe um padrão real por trás de quem decide sair, ligado a características do próprio cliente, ou o churn é essencialmente aleatório?

Essa foi a pergunta que guiou esta investigação. A partir de uma base pública do Kaggle com 10 mil clientes de um banco europeu (França, Alemanha e Espanha), com dados fictícios de perfil, produtos contratados e comportamento de conta, busquei padrões e características que se destacassem entre clientes que saíram e os que ficaram. O caminho foi garimpar os dados com Python e Pandas, testar o que parecia promissor, descartar o que não se sustentava, e seguir a pista seguinte que cada resultado abria, até mapear quais características de cliente realmente se relacionam com a saída do banco.

#### O primeiro contato com os dados

O primeiro passo foi configurar o ambiente e olhar a base pela primeira vez: conferir as colunas disponíveis, os tipos de dado e se havia valores nulos.

```python
import pandas as pd

df = pd.read_csv('../Data/Customer-Churn-Records.csv')
df.info()
```

[📎 Print: saída do código acima]

Dez mil clientes, com colunas de perfil (idade, país, gênero), produtos contratados, comportamento de conta e o alvo da análise, Exited. Não havia valor nulo em nenhuma coluna. Segui pra entender a distribuição geral das variáveis numéricas.

```python
df.drop(columns=['RowNumber', 'CustomerId']).describe().round(4).T
```

[📎 Print: saída do código acima]

Dois fatores chamaram atenção: a taxa geral de churn (20,38%) e a taxa de reclamação (20,44%) eram muito próximas, indicando uma possível relação significativa entre as duas. E pelo menos um quarto das contas tinham saldo zerado, um número grande o suficiente pra guardar como suspeita, mesmo sem saber ainda se isso significava algo. A taxa de churn, arredondada para 20,4%, passaria a ser a régua usada para comparar qualquer segmento daqui pra frente.

#### Uma pista forte demais, e por que ela não bastava

Com essa coincidência em mente, o próximo passo natural foi confirmar se havia mesmo uma relação entre reclamação e churn.

```python
df.groupby('Complain')['Exited'].mean()
```

[📎 Print: saída do código acima]

A correlação entre reclamar e sair do banco era de 99,5%. Praticamente todo cliente que registrou uma reclamação, saiu, e o inverso também era verdadeiro. Era um dado bem relevante, mas também raso. Reclamar já é, em si, um sintoma de insatisfação, não uma causa nova, o que torna essa relação quase circular. Descobrir que quem reclama sai não explica por que o cliente chegou a esse ponto. Na prática, esse achado reforça algo mais operacional do que analítico: não deixar o cliente atingir um nível de insatisfação que o leve a reclamar, já que isso é quase sinônimo de perdê-lo.

A partir daí, o ponto da investigação passava a ser outro: entender o que leva um cliente a chegar nesse nível de insatisfação. Guardei Complain como sintoma relevante, mas não como o pilar da investigação, e segui procurando um padrão que explicasse algo além disso.

#### Uma primeira varredura geral

Com essa ressalva em mente, o passo seguinte foi comparar, de forma ampla, as médias de todas as variáveis entre quem saiu e quem ficou.

```python
df.groupby('Exited')[['Age', 'Balance', 'NumOfProducts', 'IsActiveMember', 'Tenure', 'CreditScore', 'EstimatedSalary', 'HasCrCard', 'Satisfaction Score']].mean().T
```

[📎 Print: saída do código acima]

Duas variáveis chamaram atenção nessa primeira granularidade. IsActiveMember foi a que se destacou mais. Clientes que saíram tinham uma média bem mais baixa desse indicador. Balance também chamou atenção, com quem saiu apresentando saldo médio mais alto que quem ficou.

#### O saldo não é sobre idade, é sobre churn

Como a idade média de quem ficou também era menor, isso levantou uma hipótese a testar: será que clientes mais novos tendem a ter saldo menor e, por consequência, uma taxa de churn menor?

```python
jovem = df[df['Age'].between(18, 35)]
meia_idade = df[df['Age'].between(36, 50)]
senior = df[df['Age'].between(51, 92)]

print(jovem.groupby('Exited')['Balance'].mean())
print(meia_idade.groupby('Exited')['Balance'].mean())
print(senior.groupby('Exited')['Balance'].mean())
```

[📎 Print: saída do código acima]

A resposta é não. O saldo médio ficava estável dentro de cada faixa etária, tanto entre quem ficou quanto entre quem saiu. A diferença de saldo não vinha da idade, vinha do churn em si.

```python
print(jovem.shape[0])
print(meia_idade.shape[0])
print(senior.shape[0])
print(jovem['Exited'].mean())
print(meia_idade['Exited'].mean())
print(senior['Exited'].mean())
```

[📎 Print: saída do código acima]

O churn, esse sim, variava bastante: 8,36% entre os jovens (18-35), 24,6% na meia-idade (36-50) e 44,6% entre os seniores (51-92). Mas junto desse número apareceu outro problema: a quantidade de clientes em cada faixa era bem diferente entre si, 4.153 jovens, 4.586 de meia-idade e só 1.261 seniores. Uma divisão desigual demais pra confiar no resultado sem checar mais de perto. Refiz a segmentação em faixas de dez anos, buscando um recorte mais equilibrado.

```python
faixa_18_25 = df[df['Age'].between(18, 25)]
faixa_26_35 = df[df['Age'].between(26, 35)]
faixa_36_45 = df[df['Age'].between(36, 45)]
faixa_46_55 = df[df['Age'].between(46, 55)]
faixa_56_65 = df[df['Age'].between(56, 65)]
faixa_66_92 = df[df['Age'].between(66, 92)]

for nome, faixa in [('18-25', faixa_18_25), ('26-35', faixa_26_35),
                    ('36-45', faixa_36_45), ('46-55', faixa_46_55),
                    ('56-65', faixa_56_65), ('66-92', faixa_66_92)]:
    print(nome, '- Qtd:', faixa.shape[0],
          '| Churn:', round(faixa['Exited'].mean(), 3),
          '| Balance médio:', round(faixa['Balance'].mean(), 2))
```

[📎 Print: saída do código acima]

O churn não cresce de forma linear com a idade, ele tem formato de sino. Vai de 7,5% na faixa de 18-25 anos, sobe gradualmente até 19,7% em 36-45, dispara para 50,6% em 46-55, o pico absoluto, se mantém alto em 56-65 (48,3%), e volta a cair para 13,3% em 66-92. O saldo médio, por outro lado, seguiu estável, variando pouco entre 71 mil e 82 mil, sem acompanhar essa curva de sino. Com esse recorte mais fino em mãos, voltei a cruzar as faixas com Exited e com as demais variáveis do dataset, pra ver se algo mais aparecia.

```python
df['Faixa_Etaria'] = pd.cut(df['Age'],
                              bins=[17, 25, 35, 45, 55, 65, 92],
                              labels=['18-25', '26-35', '36-45', '46-55',
                                      '56-65', '66-92'])

df.groupby(['Faixa_Etaria', 'Exited'])[['Balance', 'NumOfProducts',
            'IsActiveMember', 'Tenure', 'CreditScore', 'EstimatedSalary',
            'HasCrCard', 'Satisfaction Score']].mean()
```

[📎 Print: saída do código acima]

Esse cruzamento trouxe dois padrões consistentes. Balance seguiu mais alto entre quem saiu do que entre quem ficou, em todas as faixas etárias, sem exceção, confirmando de novo que a diferença de saldo era sobre churn, não sobre idade. IsActiveMember mostrou o mesmo nível de consistência, também sem exceção em nenhuma faixa, mas com uma diferença bem mais expressiva entre os dois grupos: variando de 0,30 a 0,43 entre quem saiu contra 0,51 a 0,96 entre quem ficou. Um sinal que reaparecia em cada recorte testado, e que voltaria a ser central mais à frente.

```python
resumo = df.groupby('Faixa_Etaria').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)

resumo['Pct_do_Total_de_Saidas'] = resumo['Total_Saidas'] /
                                   resumo['Total_Saidas'].sum()

resumo
```

[📎 Print: saída do código acima]

Em relação ao volume, a faixa de 36-45 é a que mais contribui em número absoluto de saídas, com 734, seguida de perto por 46-55, com 663. O dado alarmante está aí: a faixa de 46-55 concentra apenas 13% de toda a base de clientes, mas é responsável por 32,5% de todas as saídas do banco.

#### A Alemanha não é uma questão de composição etária

Com o padrão de idade mapeado, fazia sentido verificar se ele se distribuía igualmente entre os três países da base.

```python
resumo_geo = df.groupby('Geography').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)

resumo_geo['Pct_do_Total_de_Saidas'] = resumo_geo['Total_Saidas'] /
                                       resumo_geo['Total_Saidas'].sum()

resumo_geo
```

[📎 Print: saída do código acima]

Não se distribuía. A Alemanha tinha uma taxa de churn de 32,4%, o dobro de França (16,2%) e Espanha (16,7%). Antes de tratar isso como um problema específico do país, era preciso descartar a explicação mais óbvia: será que a Alemanha só tinha uma base de clientes mais velha, e o efeito era só o pico de idade que eu já tinha encontrado?

```python
pd.crosstab(df['Geography'], df['Faixa_Etaria'], normalize='index')
```

[📎 Print: saída do código acima]

Não era o caso. A composição etária dos três países era parecida, com a Alemanha só levemente mais concentrada na faixa de 46-55, o que não bastava pra explicar uma diferença dessa magnitude.

```python
df.groupby(['Geography', 'Faixa_Etaria'])['Exited'].mean().unstack()
```

[📎 Print: saída do código acima]

Cruzando país e faixa etária ponto a ponto, o churn alemão se mantinha alto em todas as faixas, o que indicava um efeito de país independente da idade, não uma distorção demográfica. Com esse padrão confirmado, resolvi cruzar país, faixa etária e todas as outras variáveis do dataset de uma vez, pra não deixar nenhum sinal passar batido. Essa tabela ficou grande demais pra valer como print aqui, mas os achados que mais importaram nela cabem numa versão mais enxuta, isolando só país, saldo e IsActiveMember.

```python
df.groupby(['Geography', 'Exited'])[['Balance', 'IsActiveMember']].mean()
```

[📎 Print: saída do código acima]

O que chamou atenção foi o saldo médio. Na Alemanha, ele era visivelmente mais alto que em França e Espanha. Esse padrão se repetia em todas as faixas etárias, não era um efeito pontual. Além disso, dentro de França e Espanha, quem saiu tinha saldo médio maior que quem ficou, uma diferença que praticamente desaparecia na Alemanha.

IsActiveMember , esse sim, seguiu o padrão que já vinha se repetindo desde a análise por faixa etária: mais baixo entre quem saiu nos três países, sem exceção. O saldo era o sinal novo aqui. Os dois pontos juntos, o saldo alemão desproporcionalmente alto e a diferença de saldo por churn sumindo dentro dele, apontavam pra mesma suspeita: será que a distribuição de contas com saldo zero era diferente entre os países?

#### O número que escondia uma diferença estrutural

Fui checar.

```python
df.groupby('Geography').apply(lambda x: (x['Balance'] == 0).mean())
```

[📎 Print: saída do código acima]

A distribuição não era uniforme. Enquanto o saldo zerado era 0% na Alemanha, ele chegava a cerca de 48% em França e Espanha. Essa não é uma característica de comportamento de cliente. Era uma diferença estrutural de produto ou de mercado entre os países. Ela explicava os dois sinais anteriores: o saldo médio alemão mais alto (sem contas zeradas puxando a média pra baixo) e a diferença de saldo por churn em França e Espanha, que se via de outro ângulo.

Antes de seguir, testei se essa diferença de saldo tinha alguma relação com IsActiveMember , já que uma conta zerada poderia, em teoria, ser sinal de um cliente menos engajado.

```python
df['Saldo_Zerado'] = df['Balance'] == 0

pd.crosstab(df['Saldo_Zerado'], df['IsActiveMember'], normalize='index')
```

[📎 Print: saída do código acima]

Não havia relação relevante entre as duas variáveis. A proporção de clientes ativos era praticamente igual entre quem tinha saldo zerado (51,8%) e quem não tinha (51,4%), o que descartava mais essa possível confusão. Com isso, fui investigar a diferença de churn entre uma conta zerada e uma conta com saldo.

```python
df.groupby(['Geography', 'Saldo_Zerado'])['Exited'].mean()
```

[📎 Print: saída do código acima]

Olhando só para França e Espanha, clientes com saldo positivo saem entre 31% e 44% mais do que clientes com saldo zerado. O que sugere, sem que os dados confirmem isso diretamente, que quem tem dinheiro relevante na conta tem mais motivo prático para agir diante de uma insatisfação. Uma conta zerada, por outro lado, pode ser secundária para o cliente.

#### Variáveis sem sinal

Nem toda variável testada revelou algo. Depois de mapear idade, país e saldo, testei outras colunas que ainda não tinham mostrado nenhum sinal: Card Type, CreditScore e Tenure.

```python
df.groupby('Card Type').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

[📎 Print: saída do código acima]

Card Type teve distribuição quase uniforme entre os quatro tipos, com churn variando só de 19,3% a 21,8%.

```python
df.groupby(['Card Type', 'Exited'])['Point Earned'].mean()
```

[📎 Print: saída do código acima]

Nem os pontos acumulados por tipo de cartão mostraram diferença relevante entre quem saiu e quem ficou, girando sempre perto de 600 em todos os grupos.

```python
pd.crosstab(df['Faixa_Etaria'], df['Card Type'], normalize='index')
```

[📎 Print: saída do código acima]

Mesmo cruzando com faixa etária, a distribuição dos quatro tipos de cartão seguiu uniforme em todas as idades, o que reforça que essa coluna provavelmente foi atribuída de forma aleatória na base fictícia. Card Type foi descartada como fator relevante.

```python
df['Faixa_CreditScore'] = pd.cut(df['CreditScore'],
                                    bins=[349, 450, 550, 650, 750, 850],
                                    labels=['350-450', '451-550',
                                            '551-650', '651-750',
                                            '751-850'])

df.groupby('Faixa_CreditScore').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

[📎 Print: saída do código acima]

CreditScore ficou colado na média em quase todas as faixas, entre 19,2% e 21,4% de churn. A única exceção foi a faixa mais baixa (350-450), com 32,3%, mas essa faixa reúne só 189 clientes, uma amostra pequena demais pra tratar como um padrão confiável. No geral, CreditScore também foi descartado.

```python
df.groupby('Tenure').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

[📎 Print: saída do código acima]

Tenure ficou ainda mais estável. A taxa de churn variou entre 17,2% e 23%, sem nenhuma tendência clara de queda ou alta conforme o tempo de relacionamento com o banco aumenta. Mais uma variável testada, mais uma sem sinal relevante.

#### A descoberta que quase passou batido

NumOfProducts tinha média quase idêntica, por volta de 1,5, em praticamente todo recorte simples testado até aqui, incluindo a comparação geral por Exited lá no início. Por essa razão, parecia que ela seria irrelevante na análise. Decidi testá-la segmentando pelo valor absoluto da coluna, mais para não deixar passar batido do que por expectativa real de achar algo.

```python
df.groupby('NumOfProducts').agg(
    Total_Clientes=('Exited', 'size'),
    Total_Saidas=('Exited', 'sum'),
    Taxa_Churn=('Exited', 'mean')
)
```

[📎 Print: saída do código acima]

O resultado foi o oposto do esperado. Clientes com 1 produto tinham churn de 27,7%, com 2 caía para 7,6%, o ponto ótimo aparente, 3 produtos disparava para 82,7%, e 4 produtos chegava a 100%. Todos os 60 clientes com 4 produtos haviam saído. Antes de seguir, quis entender as características desse grupo de churn quase certo (3 e 4 produtos).

```python
df[df['NumOfProducts'] >= 3][['Age', 'Geography', 'Balance',
                              'IsActiveMember', 'Complain']].describe()
```

[📎 Print: saída do código acima]

A taxa média de reclamação nesse grupo era de 0,86. Um número alto que chama atenção à primeira vista, mas que já era esperado. O churn desse mesmo grupo já estava entre 82,7% e 100%. Logo, uma taxa de reclamação próxima disso só reforçava a relação entre Complain e Exited que eu já tinha identificado antes, não era um achado novo.

```python
df[df['NumOfProducts'] >= 3]['Geography'].value_counts()
```

[📎 Print: saída do código acima]

A distribuição geográfica também fazia sentido: França com 133 clientes, Alemanha com 120 e Espanha com 73. A Alemanha ter um volume quase igual ao da França chama atenção, considerando que sua base total é cerca de metade da francesa, mas isso é reflexo do próprio churn alemão, que já era estruturalmente mais alto. Como 3 ou 4 produtos representa churn quase certo, é esperado que o país com mais churn concentre proporcionalmente mais clientes nesse grupo.

Com apenas 326 clientes no grupo inteiro, de qualquer forma, era uma amostra pequena demais para valer a pena aprofundar por esse ângulo. O que realmente importava investigar era a discrepância entre 1 e 2 produtos, já que 3 e 4 praticamente garantiam a saída do cliente.

```python
df[df['NumOfProducts'].isin([1, 2])].groupby(['NumOfProducts', 'Exited'])[['Age', 'Balance', 'IsActiveMember']].mean()
```

[📎 Print: saída do código acima]

Chamou atenção que, dentro do grupo de 2 produtos, o saldo médio de quem sai (90.260) era quase o dobro do saldo médio de quem fica (48.720), uma diferença bem maior que a observada no grupo de 1 produto. Fui checar se isso tinha relação com o país.

```python
pd.crosstab(df['Geography'], df['NumOfProducts'])
```

[📎 Print: saída do código acima]

Não tinha. A distribuição de clientes por país e número de produtos ficou perto do esperado, sem nenhum país concentrando desproporcionalmente o grupo de 2 produtos. A próxima suspeita foi a mesma que já tinha resolvido o mistério do saldo alemão: talvez houvesse muito mais conta zerada num grupo do que no outro.

```python
df.groupby('NumOfProducts').apply(lambda x: (x['Balance'] == 0).mean())
```

[📎 Print: saída do código acima]

Era isso. O grupo de 2 produtos tinha 56,6% de contas zeradas, contra 17,8% no grupo de 1 produto. Essa diferença de composição distorcia a média de saldo, não o padrão de churn em si. Pra confirmar isso de vez, segmentei o churn por NumOfProducts dentro de cada cenário de saldo, zerado ou não.

```python
df.groupby(['Saldo_Zerado', 'NumOfProducts'])['Exited'].mean().unstack()
```

[📎 Print: saída do código acima]

Nos dois cenários, o padrão se manteve: 2 produtos tinha o menor risco de saída, tanto com saldo (13,2%) quanto sem saldo (3,3%), bem abaixo de 1 produto nos dois casos (25,6% e 37,3%, respectivamente). Isso confirmou que NumOfProducts é, de fato, um fator de risco independente. A diferença de saldo entre os grupos explicava a distorção na média, mas não explicava o padrão de churn em si, que se repetia com ou sem saldo zerado.

#### Uma última variável

Antes de fechar a investigação, resolvi voltar ao notebook pra conferir se alguma variável tinha ficado de fora. Percebi que Gender nunca tinha sido testada.

```python
resumo_gender = df.groupby('Gender').agg(
    Total_Clientes=('Exited', 'size'),
    Taxa_Churn=('Exited', 'mean')
)

resumo_gender['Proporcao_Base'] = resumo_gender['Total_Clientes'] / resumo_gender['Total_Clientes'].sum()

resumo_gender = resumo_gender[['Proporcao_Base', 'Taxa_Churn']]

resumo_gender
```

[📎 Print: saída do código acima]

A diferença era grande. A taxa de churn entre mulheres era de 25,1%, contra 16,5% entre os homens, quase 9 pontos percentuais de diferença, numa base razoavelmente equilibrada entre os dois grupos (54,5% homens, 45,5% mulheres). Antes de tratar isso como um gatilho novo, fazia sentido testar se o efeito não vinha dos fatores de risco que a investigação já tinha mapeado. IsActiveMember não mostrou diferença relevante entre os gêneros, o que descartava essa explicação.

```python
geo = pd.crosstab(df['Geography'], df['Gender'], normalize='index')
idade = pd.crosstab(df['Faixa_Etaria'], df['Gender'], normalize='index')
produtos = pd.crosstab(df['NumOfProducts'], df['Gender'], normalize='index')

resumo_composicao = pd.concat([geo, idade, produtos], keys=['Geography', 'Faixa_Etaria', 'NumOfProducts'])

resumo_composicao
```

[📎 Print: saída do código acima]

Os outros três mostraram sinal. Mulheres eram levemente super-representadas na Alemanha (que já tinha o churn estruturalmente mais alto entre os três países), nas faixas etárias de maior risco (46-55 e 56-65) e, principalmente, nos grupos de 3 e 4 produtos, onde chegavam a 56% e 63% da composição, respectivamente. Nenhum desses três, isolado, era grande o suficiente pra explicar a diferença de 9 pontos entre os gêneros. Mas a soma dos três explicava a maior parte dela. Gender não era um gatilho independente. Era um efeito composto, resultado de pequenas super-representações nos mesmos fatores de risco que a investigação já tinha confirmado.

#### O quadro completo

IsActiveMember foi o padrão mais consistente da investigação inteira. Em toda combinação de país e faixa etária testada, quem saiu tinha esse indicador mais baixo que quem ficou, sem exceção. A base não detalha o que exatamente define um cliente como ativo, mas o padrão é claro o suficiente pra sugerir algo prático: manter o cliente engajado parece reduzir de forma consistente a chance de ele sair, independente de idade ou país.

NumOfProducts confirmou que existe um ponto ótimo, e ele é 2 produtos. A base também não detalha o que são esses produtos, mas o salto de churn ao passar de 2 para 3 ou mais é grande demais pra ser ignorado. Isso sugere que empurrar o cliente pra um terceiro ou quarto produto pode estar gerando mais insatisfação do que valor, possivelmente a ponto de se tornar o próprio motivo da reclamação que antecede a saída.

A Alemanha se confirmou como um problema estrutural de país, não uma distorção demográfica, com churn de 32,4% contra 16,2% e 16,7% de França e Espanha. Curiosamente, é também o único dos três países sem nenhuma conta com saldo zerado. Vale a pena entender o que diferencia essa estrutura das outras filiais.

O saldo zerado, por sua vez, acabou tendo dois papéis na investigação. Foi a chave que explicou várias distorções ao longo do caminho: do saldo médio alemão ao efeito de NumOfProducts. Mas também se revelou um achado próprio. Em França e Espanha, clientes com saldo positivo saem entre 31% e 44% mais do que clientes com conta zerada. Vale pesar se isso é positivo ou negativo pro banco. Uma conta zerada é mais fácil de reter, mas também gera menos receita, e talvez o esforço de retenção valha mais a pena onde o cliente tem saldo relevante.

Já Complain, apesar da correlação de 99,5% com Exited, ficou como sintoma de insatisfação, não como causa raiz. Essa correlação tão alta também levanta uma questão: parece ser um traço da própria base fictícia. Dificilmente, em um cenário real, essa taxa seria encontrada. No entanto, em situações de alta correlação entre reclamação e churn, valeria investigar por que a insatisfação chega nesse ponto quase sempre acompanhada da saída, sem meio-termo. Isso pode indicar a ausência de um canal de suporte pra resolver dúvidas ou frustrações menores antes que elas evoluam pra uma reclamação formal e, dali, pro cancelamento.

Card Type, CreditScore e Tenure foram testados com o mesmo rigor das demais variáveis e não mostraram nenhum sinal relevante.

No fim, o churn não é aleatório, mas também não tem uma causa única. Ele se concentra em clientes menos engajados, num conjunto específico de produtos contratados, e num país com um comportamento estrutural diferente dos demais, mais do que em qualquer característica isolada de perfil ou de crédito.

---

📁 Repositório completo no GitHub: https://github.com/GuilhermeCruvinel96/bank-churn-python

#Python #Pandas #DataAnalysis #BusinessAnalytics #AnaliseDeDados
