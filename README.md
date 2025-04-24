## Sistema de Detecção de Fraudes em Tempo Real  
**Autor:** Lauro Bonometti  
**E-mail:** lauro.f.bonometti@gmail.com  
**LinkedIn:** [linkedin.com/in/laurobonometti](https://www.linkedin.com/in/laurobonometti)  

---

## 📋 Sumário  
1. [Visão Geral](#visão-geral)  
2. [Tecnologias Utilizadas](#tecnologias-utilizadas)  
3. [Arquitetura e Fluxo de Dados](#arquitetura-e-fluxo-de-dados)  
4. [Descrição dos Blocos de Código](#descrição-dos-blocos-de-código)  
5. [Métricas e Resultados Indicativos](#métricas-e-resultados-indicativos)  
6. [Como Executar](#como-executar)  
7. [Próximos Passos](#próximos-passos)  

---

## 🔍 Visão Geral  
Este projeto implementa um pipeline de detecção de fraudes em transações financeiras, que combina dois métodos de detecção de anomalias:  
- **Isolation Forest**  
- **Autoencoder (Deep Learning)**  

Ao final, gera-se um **ensemble** que decide “fraude” com base na média das pontuações normalizadas de cada modelo, usando um threshold calibrado via análise de custo (FP vs. FN).

---

## 🛠 Tecnologias Utilizadas  
- **Python 3.8+**: linguagem principal  
- **Pandas / NumPy**: manipulação de dados  
- **scikit-learn**: IsolationForest, métricas, train_test_split  
- **TensorFlow / Keras**: construção e treino do autoencoder  
- **Matplotlib**: visualização de curvas ROC/PR  
- **Joblib**: persistência de modelos  
- **Google Colab**: ambiente de desenvolvimento e experimentação  
- **Git / GitHub**: versionamento e apresentação do código  

---

## 🏗 Arquitetura e Fluxo de Dados  
1. **Leitura em chunks**  
   - Transações (`train_transaction.csv`) e identidades (`train_identity.csv`) são lidas em pedaços para economizar memória.  
   - Junta-se apenas as identidades cujos IDs aparecem nas transações carregadas.  
2. **Análise exploratória**  
   - Amostragem de até 10 000 registros para inspeção de distribuição de fraudes, tipos de dados e valores nulos.  
3. **Redução de dimensionalidade**  
   - Remove colunas com >70 % de valores nulos.  
   - Descarta variáveis de variância quase zero.  
   - Mantém apenas as top 100 variáveis mais correlacionadas com o rótulo `isFraud`.  
4. **Pré-processamento**  
   - Separa features e alvo.  
   - Factoriza categóricas, imputação de missing (mediana).  
   - Normaliza via `StandardScaler`.  
   - Gera treino/teste (80 / 20 %).  
5. **Treinamento dos Modelos**  
   - **Isolation Forest**  
     - Usa 100 árvores, contaminação = proporção de fraudes no treino.  
     - Gera score de anomalia, normaliza entre 0–1.  
   - **Autoencoder**  
     - Rede densa com codificação de dimensão ≈ 1/3 das features (máx 64), decoder simétrico.  
     - EarlyStopping (patience=5), salva melhor modelo no formato nativo Keras.  
     - Calcula MSE de reconstrução em batches e normaliza.  
6. **Avaliação e Calibração**  
   - Para cada modelo, encontra threshold que maximiza F1 na curva Precision–Recall.  
   - Ensemble threshold = média dos thresholds individuais.  
   - **Simulação de custo**: varia threshold em [0,1], conta FP × custo_FP + FN × custo_FN, escolhe threshold mínimo.  
   - Com custo_FP=10  e custo_FN=100, threshold final = 0.33.  
7. **Detector em Tempo Real**  
   - Função que recebe uma transação (pandas.DataFrame 1×N), aplica mesmíssimo pré-processamento e retorna:  
     ```json
     {
       "fraude": true|false,
       "scores": {"iso":0.x, "ae":0.y, "ens":0.z}
     }
     ```

---

## 📑 Descrição dos Blocos de Código  

| Bloco | Função Principal | Comentários |
|---|---|---|
| **1. Imports & Configuração** | carrega bibliotecas, define `logging`, `limpar_memoria()` | Import único de pacotes, warnings off. |
| **2. `carregar_dados_em_chunks()`** | lê CSVs em pedaços, concatena, amostra até 50 000 registros | Minimiza uso de RAM; trata falha de identity como opcional. |
| **3. `analisar_dados()`** | imprime shape, distribuição de fraudes, nulos e tipos | Ajuda a garantir qualidade antes de modelar. |
| **4. `reduzir_dimensoes()`** | descarta colunas >70 % nulas e baixa variância; retém top N corr. | Reduz de dezenas a poucas centenas de features. |
| **5. `preprocessar_dados()`** | factoriza categóricas, fillna, split treino/teste, normaliza | Gera arrays prontos para sklearn/TensorFlow. |
| **6. `treinar_isolation_forest()`** | treina modelo, normaliza scores de anomalia | Usa até 50 000 amostras do treino para economizar tempo. |
| **7. `treinar_autoencoder()`** | constrói e treina rede, salva melhor, calcula MSE normalizado | EarlyStopping, ModelCheckpoint (.keras), batches de inferência. |
| **8. `avaliar_modelos()`** | encontra thresholds via PR-curve; plota ROC; retorna thresholds | Thresholds iso, ae, ensemble. |
| **9. Calibração de Custo** | `calibrar_por_custo()` varre thresholds contando FP/FN, minimiza custo | Pareto entre alerta falso e fraude não capturada. |
| **10. `implementar_sistema_tempo_real()`** | retorna função `detectar(transacao)` | Pronta para deploy; aceita DataFrame simples. |
| **11. `executar_sistema()`** | orquestra todo o pipeline, retorna `detector` e `X_test` | Ponto de entrada do script / notebook. |
| **12. Teste de Batch & Export** | aplica `detector()` em N exemplos; grava CSV | Facilita auditoria e revisão manual de alertas. |

---

## 📊 Métricas e Resultados Indicativos  
- **Isolated Forest**  
  - AUC-ROC: ~0.83  
  - F1 ótimo: 0.18 @ threshold 0.59  
- **Autoencoder**  
  - AUC-ROC: ~0.86  
  - F1 ótimo: 0.20 @ threshold 0.06  
- **Ensemble**  
  - AUC-ROC: ~0.88  
  - F1 ótimo: 0.19 @ threshold 0.33  
- **Calibração de Custo (custo_FP=10, custo_FN=100)**  
  - **Threshold final:** 0.33  
  - **FP:** 153 alertas falsos  
  - **FN:** 227 fraudes perdidas  
  - **Custo estimado:** R\$ 24 230  

---

## ▶ Como Executar  
1. **Clone o repositório**  
   ```bash
   git clone https://github.com/seu-usuario/bank-churn-analytics-pro.git
   cd bank-churn-analytics-pro
   ```  
2. **Abra no Colab**  
   - Carregue o notebook `detector_fraudes.ipynb`.  
   - Garanta que o diretório `sample_data/dados` contém os CSVs.  
3. **Execute as células na ordem**  
   - Imports → Funções → Pipeline → Calibração → Testes → Exportação.  
4. **Ajuste parâmetros**  
   - `max_rows`, `max_features`, `custo_fp`, `custo_fn`, `threshold_piso` conforme necessidade.  

---

## 📈 Próximos Passos  
- Criar dashboard em Streamlit para enviar transação em tempo real.  
- Adicionar testes unitários (pré-processamento, formatos de entrada) e integração contínua.  
- Configurar monitoramento de drift de dados em produção.  
- Incorporar feedback humano (fraudes confirmadas) para re-treinos periódicos.  

---

> **Contato**: Lauro Bonometti • lauro.f.bonometti@gmail.com • [linkedin.com/in/laurobonometti](https://www.linkedin.com/in/laurobonometti)