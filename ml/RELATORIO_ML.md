# Relatório de Machine Learning · Colecity

**Pergunta:** essa reclamação de transporte vai ser resolvida?

**Integrantes:** 
Guilherme Oliveira Da Silva -  558797
Rafael Panhoca - 555014
Matheus Dantas - 558804
Victor rodriguez – 559094
Silas Alves  - 555020

---

## 1. Problema

O Colecity parte de uma observação simples: muitas reclamações que o cidadão faz sozinho à Prefeitura não andam. A proposta do app é juntar relatos parecidos em uma ação coletiva, como um abaixo-assinado.

Para saber **quais reclamações mais precisam dessa força coletiva**, transformamos o problema em uma pergunta de Machine Learning:

> A partir do tipo de reclamação, do local e da data, conseguimos prever se uma reclamação de **transporte** feita ao SP156 será **resolvida** (status "Finalizada") ou **não**?

É um problema de **classificação binária**: 1 = resolvida, 0 = não resolvida. O foco em transporte segue o MVP do Colecity.

## 2. Dados

- **Fonte:** Dados do SP156 – Solicitações, Portal de Dados Abertos da Prefeitura de São Paulo (https://dados.prefeitura.sp.gov.br/dataset/dados-do-sp156).
- **Arquivo:** "Dados do SP156 – 2º TRI 2026", aba "Solicitações".
- **Período:** reclamações abertas entre **01/04/2026 e 30/06/2026**. Segundo o portal, o status de cada reclamação é o de **19/10/2026**, data da extração.
- **Tipo:** dados **reais e públicos**, com informações pessoais já removidas pela Prefeitura.
- **Tamanho:** 463.686 reclamações de todos os temas. Depois de filtrar o tema "Trânsito e Transporte" e limpar, ficaram **87.238**.

Colunas usadas: data de abertura, status, tema, assunto, serviço, subprefeitura ("Prefeitura Operacional"), distrito e meio de comunicação (canal).

## 3. Preparação dos dados

1. **Filtro:** só reclamações do tema *Trânsito e Transporte*.
2. **Alvo:**
   - `1` = **Finalizada**;
   - `0` = **Aberto** ou **Em Andamento**. Como a extração foi em outubro, uma reclamação de abril a junho ainda aberta está parada há mais de 3 meses;
   - **Cancelada** foi removida (355 reclamações), porque foi o próprio cidadão que desistiu.

   Resultado: 74.276 finalizadas, 12.890 abertas e 72 em andamento. A **taxa geral de resolução é de 85,1%**, ou seja, as classes são **desbalanceadas**.
3. **Valores faltantes:** cerca de **54%** das reclamações não têm subprefeitura nem distrito. Em vez de apagar mais da metade dos dados, preenchemos com "NAO INFORMADO".
4. **Variáveis da data:** a data foi separada em dia do mês e dia da semana. O arquivo não traz a hora.
5. **Texto para números:** as variáveis de texto (assunto, serviço, subprefeitura, distrito e canal) foram convertidas com **OneHotEncoder**. Categorias com menos de 50 ocorrências foram agrupadas.
6. **Vazamento de dados evitado:** **não** usamos *Data de Finalização* nem *Órgão*, porque essas informações só existem depois que a Prefeitura responde. Usá-las seria dar a resposta ao modelo.

**Divisão treino/teste por tempo, não aleatória:** treinamos com o passado e testamos com o futuro, como aconteceria de verdade no app.

| Conjunto | Período | Reclamações |
|---|---|---|
| Treino | abril e maio de 2026 | 57.957 |
| Teste | junho de 2026 | 29.281 |

## 4. Modelos e treinamento

Testamos **3 modelos**, cada um com **12 combinações de 3 hiperparâmetros**:

| Modelo | Hiperparâmetros testados |
|---|---|
| Random Forest | `n_estimators` (50, 100) · `max_depth` (5, 10, 20) · `min_samples_leaf` (1, 20) |
| XGBoost | `n_estimators` (100, 200) · `max_depth` (3, 6, 10) · `learning_rate` (0,05, 0,3) |
| LightGBM | `n_estimators` (100, 200) · `num_leaves` (15, 31, 63) · `learning_rate` (0,05, 0,3) |

As regras foram iguais para os três:

- **Validação cruzada só no treino**, com `TimeSeriesSplit` (3 rodadas, sempre treinando no passado e validando no período seguinte), usando `GridSearchCV`.
- **Métrica de escolha: F1 macro**, a média do F1 das duas classes. Com 85% de reclamações resolvidas, a acurácia sozinha enganaria: um modelo que chutasse "resolvida" para tudo já teria cerca de 85%.
- **Desbalanceamento:** `class_weight='balanced'` no Random Forest e no LightGBM, e `scale_pos_weight` no XGBoost.

**Melhores combinações:**
- Random Forest: `n_estimators=100`, `max_depth=20`, `min_samples_leaf=1`.
- XGBoost: `n_estimators=200`, `max_depth=10`, `learning_rate=0,05`.
- LightGBM: `n_estimators=100`, `num_leaves=63`, `learning_rate=0,05`.

No Random Forest, as árvores rasas (`max_depth=5`) tiveram o pior F1 de validação (entre 0,872 e 0,874), contra 0,898 da melhor combinação.

## 5. Escolha do modelo

| Modelo | F1 validação | F1 treino | F1 teste | Diferença treino − teste |
|---|---|---|---|---|
| **Random Forest** | **0,898** | 0,928 | 0,893 | 0,035 |
| XGBoost | 0,890 | 0,897 | 0,894 | 0,003 |
| LightGBM | 0,888 | 0,891 | 0,895 | −0,004 |

Escolhemos o **Random Forest** porque:

- **Teve o maior F1 na validação (0,898).** A escolha é feita pela validação, e não pelo teste, que serve como "prova final".
- **No teste os três ficaram praticamente empatados** (de 0,893 a 0,895). Ou seja, o Random Forest generaliza tão bem quanto os outros.
- **Explica a decisão:** mostra diretamente quais variáveis mais pesam, o que ajuda a traduzir o resultado para o cidadão.
- **É mais simples:** já vem no scikit-learn.

Ponto de atenção: o Random Forest teve um pouco mais de overfitting que os outros dois (0,035 contra quase zero). Discutimos isso na próxima seção.

LSTM e KMeans + PCA não foram testados porque não se encaixam no problema. A LSTM é para sequências no tempo, e o KMeans é não supervisionado, enquanto nós temos um alvo claro para prever.

## 6. Resultados em treino e teste (Random Forest)

| Métrica (macro) | Treino | Teste |
|---|---|---|
| Acurácia | 0,964 | 0,940 |
| Precisão | 0,903 | 0,906 |
| Recall | 0,959 | 0,881 |
| **F1** | **0,928** | **0,893** |

**Matriz de confusão (teste, junho de 2026):**

| | Previsto: não resolvida | Previsto: resolvida |
|---|---|---|
| **Real: não resolvida** (5.120) | **4.046** | 1.074 |
| **Real: resolvida** (24.161) | 683 | **23.478** |

O modelo acertou **79%** das reclamações que **não foram resolvidas** (4.046 de 5.120) e **97%** das que foram (23.478 de 24.161).

**Overfitting:** o F1 caiu de 0,928 no treino para 0,893 no teste, uma diferença de **0,035**. É uma queda pequena, o que mostra que o modelo aprendeu padrões que se repetem no mês seguinte, em vez de decorar o treino. A maior queda está no recall (de 0,959 para 0,881), principalmente nas reclamações não resolvidas: 1.074 delas foram previstas como resolvidas.

**O que mais pesa na previsão:** as duas variáveis mais importantes são o assunto *Veículos abandonados* e o serviço *Solicitar remoção de veículo/carcaça abandonado na via pública*. Em seguida vêm o local não informado e o assunto *Carga a Frete*. O dia do mês aparece com peso pequeno.

## 7. Conclusão para o cidadão

- **O tipo de problema é o que mais decide se a reclamação é resolvida.** No período estudado, apenas cerca de **22%** das reclamações de *veículos abandonados* foram finalizadas, enquanto assuntos como *Estacionamento* e *Sinalização* passaram de 90%.
- **O lugar também importa.** Segundo o modelo, uma reclamação de transporte tem a **menor chance de ser resolvida** em **Vila Prudente (34%), Butantã (45%), Freguesia/Brasilândia (45%), Sapopemba (45%) e Ipiranga (46%)**. Os dados reais de junho confirmam: Butantã finalizou 40% e Vila Prudente 44%.

**O que isso significa na prática:** no Colecity, quando um morador de uma dessas regiões registrar um problema que historicamente não é resolvido, como um veículo abandonado, o app pode mostrar que reclamar sozinho tem pouca chance de dar resultado e convidá-lo a entrar em um abaixo-assinado com os vizinhos. O modelo indica **onde e em quais problemas a mobilização coletiva é mais necessária**.

## 8. Limitações

- **"Finalizada" não é o mesmo que "resolvida".** O status indica que a Prefeitura **fechou o protocolo**, não que o problema acabou. Reclamações sobre ônibus, por exemplo, aparecem quase 100% como finalizadas. Na aba "Satisfação" do mesmo arquivo, das 90 pessoas que responderam se a Prefeitura atendeu sua reclamação de transporte, **42 disseram "Não"**. A amostra é pequena demais para treinar um modelo, mas reforça a ideia do Colecity de deixar o próprio cidadão confirmar se o problema foi resolvido.
- **Só 3 meses de dados** (2º trimestre de 2026). Com mais trimestres, o modelo poderia captar variações ao longo do ano.
- **54% das reclamações não têm local**, o que limita a análise por região.
- **Uma reclamação pode não ser um problema único:** segundo a própria Prefeitura, várias pessoas podem abrir protocolos para o mesmo problema.
