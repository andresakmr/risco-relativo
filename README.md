# Projeto: Risco Relativo
Projeto realizado durante meu bootcamp no Laboratoria no qual pratiquei os conceitos de risco relativo, matriz de confusão e regressão logística. Ferramentas: BigQuery, Looker Studio e Google Colab.

**Caso**

Num contexto recente, a descida das taxas de juro no mercado desencadeou um aumento notável na demanda por pedidos de crédito. Os clientes viram este movimento do mercado, como uma oportunidade favorável para financiar grandes compras ou consolidar dívidas existentes, o que elevou o fluxo de pedidos de empréstimo no banco “Super Caja” (Super Caixa, em tradução livre). A equipe de análise de crédito do banco enfrenta um fardo esmagador devido à análise manual necessária para cada solicitação de empréstimo de clientes individuais. Esta metodologia manual resultou num processo ineficiente e atrasado, que afetou negativamente a eficiência e a velocidade com que os pedidos de empréstimo são processados. A situação tornou-se mais crítica devido à preocupação crescente com a taxa de inadimplência, um problema que afeta cada vez mais o setor financeiro, e aumenta a pressão sobre os bancos para identificar e mitigar riscos associados ao crédito.

Para enfrentar esse desafio, a proposta do banco é a automação do processo de análise de crédito usando técnicas de análise avançadas de dados, com o objetivo de melhorar a eficiência, precisão e rapidez na avaliação de pedidos de crédito. Além disso, o banco já tem uma métrica para identificar clientes com pagamento atrasado, o que poderia ser uma ferramenta valiosa para integrar na classificação de risco no novo sistema automatizado.

O objetivo da análise é **identificar o perfil de clientes com risco de inadimplência, montar uma pontuação de crédito através da análise de dados e avaliar o risco relativo, possibilitando assim, classificar os clientes e futuros clientes em diferentes categorias de risco com base na sua probabilidade de inadimplência.** Esta classificação permitirá ao banco tomar decisões informadas sobre a quem conceder crédito, reduzindo assim o risco de empréstimos não reembolsáveis. Além disso, a integração destas métricas fortalecerá a capacidade do modelo de identificar riscos, contribuindo para a solidez financeira e a eficiência operacional do Banco.

## **Processamento e preparo do banco de dados**

### **Conectar/importar dados para ferramentas**

Criei as tabelas com os nomes originais: default, loans_detail, loans_outstanding, user_info. 

Ambas tem 36 mil linhas/valores/clientes, com exceção da loans_outstanding (são 305.335 empréstimos).

Também notei que a id do cliente varia entre 3 a 5 algorismos.

**Identificar e tratar valores nulos**

- Tabela default, loans_detail, loans_outstanding: nenhum nulo encontrado
- Tabela user_info: 7199 valores nulos em last_month_salary e 943 em number_dependents

**Identificar e tratar valores duplicados**
- Tabela default, loans_detail, user_info: nenhum duplicado encontrado
- Tabela loans_outstanding: nenhum duplicado encontrado*. Encontrei 34.510 user_id repetidos, mas é natural pois a mesma pessoa pode solicitar empréstimo mais de uma vez. Mas imagino que vou precisar tratar esses valores, agrupando.

**Identificar e gerenciar dados fora do escopo da análise**
- Tabela loans_detail: achei 3 variáveis correlatas. Apliquei o desvio padrão das 3, vi que a que tinha maior desvio era a number_times_delayed_payment_loan_30_59_days (4.14) e salvei numa view. 
- Tabela user_info: eliminei a variável sex da minha análise por ser discriminatória para o objetivo do projeto e atualizei a tabela manipulada

**Identificar e tratar dados discrepantes em variáveis categóricas**
- Tabela loans_outstanding: ajustei a variável string loan_type deixando tudo em lowercase, corrigi o “others” e criei uma v1 de tabela limpa

**Identificar e tratar dados discrepantes em variáveis numéricas**
- Tabela default, loans_detail, loans_outstanding: sem dados discrepantes
- Tabela user_info: ao verificar a variável age, identifiquei 3 idades muito extremas. Resolvi substituir pela mediana. Primeiro calculei ela, entre os não-outliers, que deu 52 anos e substitui em uma nova coluna.

**Criar novas variáveis**
- Tabela loans_detail: decidi criar uma variável que soma os atrasos
- Tabela loans_outstanding: decidi aproveitar a variável de loan_id agrupado que criei antes para adicionar à tabela.

**Unir tabelas**

Como a tabela de clientes é a “mais completa”, no sentido de que as outras são dados sobre seus empréstimos, decido fazer um LEFT JOIN pra ela. 
Mas primeiro, precisei deixar todas com o mesmo número de linhas (ids dos clientes) com outra query.

## **Análise Exploratória**

**Agrupar e ver dados de acordo com variáveis categóricas**

No Looker Studio, vi as infos de idade, tipo de empréstimo e status (adimplente ou inadimplente). Vi que:
- A minoria dos clientes estão classificados como inadimplentes (1,8%). Isso significa que temos alguma mostra validada/classificada pelo banco, o que servirá de referência.
- A maioria dos clientes faz empréstimos para bens diversos, a minoria não solicitou nenhum tipo de empréstimo.

**Aplicar medidas de tendência central (moda, média, mediana)**

- Notei que a idade dos clientes é bem distribuída, pois a média e a mediana coincidiram (52 anos). O cliente com idade de 36 anos é até mais frequente, mas ainda assim os mais velhos estão em quantidade suficiente para puxar a amostra para cima.
- Em relação aos salários, vi que a média é de $6.420, tendo como valor máximo $1.560.100, o que podemos considerar outlier mas notei outros valores próximos então não diria que é discrepante. A mediana é de $5400, decidi usar ela como substituto para os valores nulos.

**Ver distribuição**

- Fiz um boxplot de idade, usando como dimensão a variavel de default_flag que informa quem é inadimplente e adimplente. Os inadimplentes tendem a ser mais jovens.
- Também fiz mais um boxplot com o número de dependentes. O grupo de clientes inadimplentes tende a ter um número de dependentes ligeiramente maior, o que pode indicar uma maior carga financeira. Enquanto a mediana para adimplentes é 0, a mediana para inadimplentes é 1.

**Aplicar medidas de dispersão (desvio padrão)**

Calculei o desvio padrão das medidas que eu já tinha começado a analisar, como idade, salário e atrasos. O dp do salário é bem relevante, corroborando com a análise anterior. 

**Calcular quartis, decis ou percentis**

Calculei os quartis para idade e já agrupei com a minha coluna de default_flag_limpo para entender os grupos de maus pagadores. 

**Calcular correlação entre variáveis numéricas**

Fiz alguns cruzamentos no BigQuery e, entre as variáveis em si, só a default_flag que apresentou uma correlação (fracamente positiva) com as variáveis de atraso, como atraso de 90 dias, e 30 a 59 dias (entre 0,299 e 0,3 o índice de Pearson). Juntei os quartis em uma view para correlacioná-los também.

## **Aplicar técnica de análise**

**Calcular risco relativo**

Calculei para os fatores que apresentaram maior correlação

**Aplicar segmentação por Score**

Numa primeira aplicação, a performance foi boa para acurácia e recall, mas não para precisão e F1 Score. Crio um modelo 2 com uma nota de corte maior do score. Apesar de algumas melhorias, ainda continuo com problemas na precisão; impactando no f1 score geral. Como nosso objetivo é minimizar o risco de crédito no Super Caja e capturar o máximo de maus pagadores possível (priorizando o Recall), decido ficar com o modelo 1, mesmo com a precisão baixa. 

**Regressão Logística com o Google Colab**

O modelo melhorou. Mas para os inadimplentes, a precisão ainda apresentou baixa. Para equilibrar melhor o modelo, decido abrir mão da precisão para aumentar um pouco o recall por meio do ajuste no limiar de decisão (predict()). Nesse modelo, se a probabilidade de um cliente ser um mau pagador for maior que 0.5, o modelo o classifica como 1 (mau pagador); caso contrário, o classifica como 0 (bom pagador). Então reduzo para 0.3 pois meu objetivo é sempre captar os maus pagadores para evitar prejuízo ao banco, ainda que abra mão de oportunidades de concessão de crédito a clientes adimplentes. Resultado:
- Precisão: Caiu de 0.59 para 0.47
- Recall: Subiu de 0.29 para 0.50
- F1-Score: Subiu de 0.39 para 48

O modelo atual é o melhor porque tem uma precisão razoável (47%): de cada 100 alertas de "mau pagador", quase metade são corretos. E agora temos um Recall mais alto (50%): capturando metade dos verdadeiros maus pagadores.

Aproveitei para gerar a curva ROC também, que mostrou um AUC de 0.9.

## Próximos passos

- A base é bem desbalanceado, 137 maus pagadores em 7200 amostras, então eu precisaria estudar e praticar outras técnicas para lidar com isso.
- A base tinha ~20% de valores desconhecidos, o que pode ter comprometido o treinamento do algoritmo pois aquele valor da mediana talvez não refletia a realidade.
- Criar novas variáveis para aprimorar as categorias.
- Testar outros modelos, além da regressão.

## Links de interesse

📊 [Dashboard no Looker Studio](https://lookerstudio.google.com/reporting/ec070952-b2e6-4805-bdd2-f5c6651d9bf6)

👩‍💻 [Notebook no Google Colab](https://colab.research.google.com/drive/1FIsln1LyWPQjA0OvieYWLRA3nD2i5evf?usp=sharing)

💻  [Apresentação](https://docs.google.com/presentation/d/1eV5NguvDQ0GSNeLW0EHGwVS-eH8Q1AD2HkbuRo8EvzQ/edit?usp=sharing)







