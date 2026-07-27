# Pipeline de Dados: Arquitetura Medalhão com Gemini e BigQuery

Este projeto demonstra a implementação de um pipeline de dados ponta a ponta utilizando a **Arquitetura Medalhão** (Bronze, Silver e Gold) para processar feedbacks de clientes de uma API pública, enriquecê-los com Inteligência Artificial e gerar insights de negócio.

## 🏗️ Arquitetura

O pipeline está dividido em três camadas lógicas:

1.  **Camada Bronze (Raw)**: Ingestão de dados brutos da API [DummyJSON](https://dummyjson.com/) diretamente para o Google BigQuery, preservando a fidelidade da origem.
2.  **Camada Silver (Enriched)**: Processamento e enriquecimento dos dados. Utilizamos o modelo **Gemini 2.5 Flash** (via Vertex AI) para classificar o sentimento e a categoria dos comentários de forma automatizada.
3.  **Camada Gold (Business)**: Agregação dos dados para consumo final. Aqui, realizamos uma **Análise de Pareto** para identificar as categorias de feedback mais recorrentes, visualizadas através de um gráfico interativo com Plotly.

## 🛠️ Tecnologias Utilizadas

*   **Linguagem**: Python
*   **Armazenamento/Data Warehouse**: Google BigQuery
*   **IA Generativa**: Google Gemini 2.5 Flash (Vertex AI)
*   **Processamento de Dados**: Pandas
*   **Visualização**: Plotly

## 🚀 Como Executar

1.  **Configuração do GCP**: Certifique-se de ter um projeto no Google Cloud Platform e o ID do projeto em mãos.
2.  **Variáveis**: Na primeira célula de código, substitua a variável `PROJECT_ID` pelo seu ID do GCP.
3.  **Autenticação**: O notebook utiliza as credenciais nativas do Colab para acessar o BigQuery e a Vertex AI.
4.  **Execução**: Rode as células em ordem sequencial (Bronze -> Silver -> Gold).

---