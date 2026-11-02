# Inteligência Artificial aplicada a previsão de chuvas intensas em Belo Horizonte
MeteoPredict BH
Protótipo de análise retrospectiva de previsões probabilísticas de chuva intensa em Belo Horizonte (MG), desenvolvido com Python, Machine Learning e Streamlit.

O MeteoPredict BH investiga a possibilidade de estimar, a partir de observações meteorológicas disponíveis até um instante de referência, a probabilidade de precipitação acumulada ≥ 30 mm nas 24 horas seguintes na estação automática Pampulha (A521), do Instituto Nacional de Meteorologia (INMET).

[!NOTA]
Não é um sistema de previsão em tempo real nem um serviço de alerta meteorológico. A interface apresenta resultados retrospectivos previamente calculados para 2024–2025. O desempenho de detecção de chuvas intensas foi baixo no teste independente, e a estação Pampulha não representa toda a área de Belo Horizonte. Para informações e alertas oficiais, consulte o INMET, a Defesa Civil e o Cemaden.

Objetivo

Explorar o uso de aprendizado de máquina na estimativa de probabilidades de chuva intensa e avaliar criticamente o comportamento dos modelos diante de dados históricos, com atenção à qualidade dos registros, à validação cronológica, a eventos relativamente raros e às limitações da aplicação.

Dados e definição do problema

| Aspecto | Descrição |
|---|---|
| Fonte observacional | INMET — estação automática Pampulha (A521), Belo Horizonte/MG |
| Série de dados analisada | 2015–2025, em resolução horária |
| Alvo | Soma das precipitações de **t+1 até t+24 horas ≥ 30 mm** |
| Unidade de previsão | Cada instante horário elegível, e não uma tempestade independente |
| Treinamento | 2015–2021 |
| Validação / seleção | 2022–2023 |
| Teste reservado | 2024–2025 |
| Referência temporal do processamento | UTC; horários do painel convertidos para `America/Sao_Paulo` (UTC−3) |

O conjunto de modelagem exige 24 horas anteriores de precipitação válida e 24 horas futuras válidas para verificar o alvo. A série horária é reindexada e faltas de observações são mantidas como ausentes, sem preencher artificialmente períodos sem chuva. A divisão dos conjuntos é cronológica, com verificação de que as janelas de alvo não ultrapassem a fronteira entre períodos.

Importante: o resultado futuro é usado para construir o rótulo observado e avaliar as previsões, não como variável preditora. A atribuição do rótulo é necessariamente retrospectiva neste estudo.

Metodologia

O notebook reúne cinco etapas principais:

1. Preparação e auditoria: leitura do CSV horário do INMET, verificação de metadados da A521, conversão de horários e números, identificação de duplicidades e lacunas, checagens de plausibilidade e exploração da sazonalidade.
2. Construção do problema: criação de janelas retrospectivas de 6, 12 e 24 horas, definição explícita do alvo futuro de 24 horas, divisão temporal e implementação de um baseline probabilístico sazonal por mês, estimado com o conjunto de treinamento.
3. Treinamento e seleção: comparação entre regressão logística, Random Forest e XGBoost. A escolha do modelo utiliza a maior Average Precision (AP) na validação; o limiar de classificação é definido nesse mesmo período a partir do F1, sem otimizá-lo no teste.
4. Diagnóstico: comparação probabilística com o baseline, análise por ano, diagrama de confiabilidade descritivo, matriz de confusão, falsos negativos e exploração de janelas positivas sobrepostas.
5. Dashboard: apresentação, em Streamlit, das probabilidades e classificações já calculadas e congeladas, com filtros, gráficos, consulta por horário e exportação em CSV.

Variáveis utilizadas nos modelos
   
- Precipitação acumulada nas 6, 12 e 24 horas anteriores.
- Médias retrospectivas de temperatura do ar, umidade relativa, pressão atmosférica e velocidade do vento nas mesmas janelas.
- Mês e hora do instante de referência (UTC).
São 17 preditores ao todo. As variáveis meteorológicas e as janelas de chuva são construídas a partir do instante de referência e do passado; campos como chuva futura e classificação observada são explicitamente excluídos das entradas do modelo.

Resultados no período de teste (2024–2025)

O modelo escolhido pela validação foi o XGBoost_d3_n180, com 180 estimadores e profundidade máxima de 3. O limiar de classificação selecionado na validação e mantido no teste foi 0,25022 (aproximadamente 25,02%).

O conjunto de teste contém 17.262 instantes horários elegíveis, dos quais 689 têm precipitação acumulada observada ≥ 30 mm nas 24 horas seguintes (3,99%).

| Métrica | XGBoost | Baseline sazonal |
|---|---:|---:|
| Average Precision (AP) | **0,1131** | 0,0821 |
| ROC-AUC | **0,8279** | 0,7649 |
| Brier Score (menor é melhor) | 0,0372 | **0,0371** |
| Precisão (*precision*) | **9,26%** | 8,23% |
| Sensibilidade (*recall*) | 4,35% | **34,11%** |
| F1-score | 0,0592 | **0,1326** |

As métricas de classificação usam o limiar congelado de cada método, determinado previamente na validação; portanto, não representam uma comparação sob o mesmo limiar numérico.

Matriz de confusão do XGBoost no teste

| Resultado observado | Previsto: sem evento | Previsto: com evento |
|---|---:|---:|
| Sem evento | 16.279 | 294 |
| Com evento | 659 | 30 |

Interpretação: embora tenha apresentado AP superior à do baseline sazonal, o XGBoost identificou somente 30 de 689 janelas positivas no teste, deixando 659 sem sinalização pelo limiar adotado. Além disso, das 324 classificações positivas, apenas 30 corresponderam a janelas realmente positivas. Portanto, os resultados não sustentam o uso operacional do protótipo como ferramenta de alerta.

A AP descreve o desempenho de ordenação de exemplos em diferentes limiares, enquanto precisão, recall e F1 dependem do limiar adotado. A pequena diferença de Brier Score em relação ao baseline também não demonstra superioridade na qualidade das probabilidades. Os valores devem ser interpretados em conjunto, especialmente porque o fenômeno-alvo é pouco frequente.

Dashboard interativo
A aplicação Streamlit permite:

Filtrar os registros de teste pelo ano exibido no horário de Belo Horizonte.
Visualizar a probabilidade média do XGBoost, a probabilidade do baseline sazonal e a frequência efetivamente observada, agrupadas por mês.
Consultar indicadores no recorte escolhido: número de registros, janelas positivas, AP, recall e precisão.
Examinar a matriz de confusão e as métricas probabilísticas no subconjunto filtrado.
Selecionar um instante histórico para comparar as probabilidades calculadas e o resultado posteriormente observado.
Exportar as linhas filtradas em formato CSV.

O filtro altera apenas a visualização e as métricas descritivas do subconjunto; ele não retreina o modelo nem altera seu limiar. A avaliação principal do projeto permanece a do conjunto completo de teste.

Como executar

O arquivo principal do repositório é [`Modelo_IA_previsão.ipynb`](./Modelo_IA_previs%C3%A3o.ipynb), desenvolvido para execução no Google Colab com arquivos armazenados no Google Drive.

Dependências principais

- Python; pandas; NumPy; Matplotlib; scikit-learn; XGBoost; joblib.
- Streamlit e Plotly, para a interface interativa.
- Google Colab/Google Drive, para o fluxo e os caminhos de arquivo atualmente implementados.
- cloudflared, utilizado no notebook para disponibilização temporária do Streamlit por um túnel Cloudflare.

Reprodução do fluxo atual

1. Obtenha os dados horários oficiais do INMET para a estação A521, correspondentes a 2015–2025. O notebook espera inicialmente o arquivo dados_A521_H_2015-01-01_2025-12-31.csv em /content/drive/MyDrive/Dados_MG/.
2. Abra o notebook no Colab e conecte/mantenha acessível o Google Drive. Confira os caminhos definidos nas células antes da execução.
3. Execute as etapas 1 a 5 em ordem. As etapas produzem arquivos intermediários em /content/drive/MyDrive/Resultados_Pampulha/, particularmente nas subpastas Notebook_02, Notebook_03, Notebook_04 e Notebook_05.
4. A etapa de treinamento exporta o modelo selecionado, suas métricas, metadados e as probabilidades históricas do teste. A etapa do Streamlit consome principalmente Notebook_03/previsoes_teste_modelo_escolhido.csv e Notebook_03/metodologia_modelo.json.
5. O notebook gera Notebook_05/app_meteopredict_bh.py, executa o Streamlit e pode abrir um túnel HTTPS temporário para demonstrar o painel enquanto a sessão do Colab permanecer ativa.

Reprodutibilidade: publicar somente o .ipynb não disponibiliza automaticamente o CSV original, os arquivos exportados para o Drive nem um dashboard hospedado permanentemente. A execução integral depende de obter os dados e configurar os caminhos. Este README descreve o fluxo implementado, mas não afirma que uma instalação automatizada e independente do Colab já esteja disponível.

Limitações e cuidados de interpretação

1. Retrospectivo, não operacional. O painel lê probabilidades previamente calculadas sobre 2024–2025. Não consulta observações recentes e não produz previsões prospectivas em tempo real.
2. Baixa sensibilidade no teste. Com o limiar congelado, o XGBoost deixou de identificar 659 das 689 janelas positivas (recall de 4,35%). Não é apropriado para alertas de risco à população.
3. Representatividade espacial restrita. A modelagem usa observações de uma estação. Eventos localizados em outros bairros podem ter comportamento distinto; os resultados não constituem previsão para toda Belo Horizonte.
4. Janelas horárias sobrepostas. Horários consecutivos podem se referir ao mesmo episódio de chuva. Assim, 689 janelas positivas não equivalem a 689 tempestades independentes, e as métricas horárias não devem ser interpretadas como métricas por evento.
5. Eventos pouco frequentes e mudança temporal. Apenas 3,99% das janelas do teste foram positivas. A queda da AP de 0,2640 na validação para 0,1131 no teste, bem como a queda do recall, evidencia dificuldade de generalização temporal, sem identificar isoladamente sua causa.
6. Probabilidades não recalibradas. O notebook inclui análise descritiva de confiabilidade, mas não aplica um procedimento de recalibração validado em dados independentes. As saídas numéricas não devem ser tratadas como probabilidades operacionais comprovadamente calibradas.
7. Dados ausentes e qualidade da medição. Lacunas e valores inválidos afetam a quantidade de janelas elegíveis. Os filtros implementados não substituem uma auditoria meteorológica completa dos dados observacionais.
8. Dependência do ambiente. O fluxo utiliza diretórios específicos no Google Drive e um túnel de demonstração ligado à sessão do Colab; não há, neste notebook, um serviço autônomo continuamente executado.
9. Avaliação limitada ao recorte estudado. O teste cobre 2024–2025 e uma única estação; ainda são necessários testes prospectivos, avaliação em outros períodos/estações e métricas por episódio antes de qualquer aplicação operacional.

Próximos passos

- Automatizar a ingestão, verificação e atualização de observações meteorológicas.
- Implementar previsões prospectivas, garantindo disponibilidade real de cada variável no momento da emissão.
- Ampliar a avaliação com novos períodos, outras estações e validação fora da amostra original.
- Investigar calibração probabilística, estabilidade temporal e critérios de decisão apropriados ao custo de falsos negativos e falsos positivos.
- Avaliar episódios independentes de chuva, além das janelas horárias sobrepostas.
- Desacoplar o aplicativo do Colab/Drive, organizar os artefatos reproduzíveis e preparar uma implantação persistente, com observabilidade e controles de acesso.

  Tecnologias
  
Python · pandas · NumPy · scikit-learn · XGBoost · Matplotlib · Plotly · Streamlit · Google Colab

Fonte dos dados e escopo de uso

- Fonte: dados meteorológicos observacionais do Instituto Nacional de Meteorologia (INMET), estação automática Pampulha (A521), Belo Horizonte/MG.
- Finalidade: estudo e demonstração acadêmica em Ciência de Dados e Inteligência Artificial aplicada à meteorologia.
- Aviso: este projeto não é afiliado a serviços oficiais de alerta, não substitui avaliação meteorológica especializada e não deve ser utilizado isoladamente para decisões de proteção civil.

MeteoPredict BH — um estudo experimental sobre o potencial e, principalmente, os limites do aprendizado de máquina na estimativa de eventos de chuva intensa.
