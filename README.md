# 🚀 Projeto Completo de Dados no GCP: Pipeline, IA e Visualização

Este guia documenta a execução de um projeto completo de dados na Google Cloud Platform (GCP), utilizando a Arquitetura Medalhão (Bronze, Prata, Ouro), Workload Identity Federation (WIF) para CI/CD seguro, Vertex AI (Gemini 2.5 Flash) para enriquecimento de dados e Plotly no Vertex AI Workbench para visualização analítica.

---

## 🛠️ ETAPA 1: Repositório GitHub & Autenticação Segura (WIF)

Em vez de usar chaves de Service Account (`.json`) — que representam riscos graves de vazamento —, utilizamos o **Workload Identity Federation (WIF)** para permitir que as GitHub Actions se autentiquem no GCP via tokens temporários (OIDC).

### 1.1. Estruturação do Repositório
Crie o repositório com a seguinte estrutura de arquivos:

```text
citi-data-pipeline-gcp/
├── .github/
│   └── workflows/
│       └── deploy_pipeline.yml  # Pipeline CI/CD com WIF
├── notebooks/
│   └── 04_visualizacao.ipynb    # Jupyter Notebook para o Workbench
├── src/
│   ├── 01_bronze_ingestion.py   # Ingestão de dados brutos
│   ├── 02_silver_enrichment.py  # Limpeza e chamada ao Vertex AI
│   └── 03_gold_aggregation.sql  # Queries de transformação e Pareto
├── .gitignore
├── requirements.txt
└── README.md
```

### 1.2. Configurando o Workload Identity Federation (GCP Cloud Shell)
Execute os comandos abaixo para configurar a confiança entre o repositório do GitHub e o GCP:

```bash
# 1. Variáveis globais
export PROJECT_ID="seu-projeto-gcp"
export WORKPOOL_NAME="github-actions-pool"
export PROVIDER_NAME="github-provider"
export REPO_PATH="seu-usuario-ou-org/citi-data-pipeline-gcp"

# 2. Criar a Service Account da Pipeline
gcloud iam service-accounts create sa-github-pipeline \
    --display-name="SA para GitHub Actions Pipeline"

# 3. Criar o Workload Identity Pool e Provider
gcloud iam workload-identity-pools create $WORKPOOL_NAME \
    --location="global" \
    --display-name="GitHub Actions Pool"

WORKPOOL_ID=$(gcloud iam workload-identity-pools describe $WORKPOOL_NAME \
    --location="global" --format="value(name)")

gcloud iam workload-identity-pools providers create-oidc $PROVIDER_NAME \
    --location="global" \
    --workload-identity-pool=$WORKPOOL_NAME \
    --issuer-uri="https://token.actions.githubusercontent.com" \
    --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# 4. Conceder acesso do repositório GitHub à Service Account
gcloud iam service-accounts add-iam-policy-binding "sa-github-pipeline@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/iam.workloadIdentityUser" \
    --member="principalSet://iam.googleapis.com/${WORKPOOL_ID}/attribute.repository/${REPO_PATH}"

# 5. Conceder permissões de BigQuery, Storage e Vertex AI para a SA
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:sa-github-pipeline@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/bigquery.admin"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:sa-github-pipeline@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/aiplatform.user"
```

---

## 🥉 ETAPA 2: Camada Bronze — Ingestão Sem Estruturação (`01_bronze_ingestion.py`)

A camada Bronze armazena os dados brutos como recebidos das fontes (APIs externas, bancos de staging ou landing zone do Cloud Storage).

```python
import os
from google.cloud import bigquery

def run_bronze():
    project_id = os.getenv("GCP_PROJECT_ID")
    dataset_id = "db_feedbacks_medalhao"
    table_id = f"{project_id}.{dataset_id}.bronze_feedbacks_raw"

    client = bigquery.Client(project=project_id)

    # Garante a existência do Dataset por região
    dataset = bigquery.Dataset(f"{project_id}.{dataset_id}")
    dataset.location = "us-central1"
    client.create_dataset(dataset, exists_ok=True)

    # Simulação de ingestão de API de Feedbacks de Clientes
    raw_data = [
        {"id": 101, "cliente": "Empresa A", "comentario": "O atendimento foi rápido, mas a API cai aos finais de semana.", "data_envio": "2026-03-01"},
        {"id": 102, "cliente": "Empresa B", "comentario": "Excelente suporte técnico! Resolveram nosso problema de infra em minutos.", "data_envio": "2026-03-01"},
        {"id": 103, "cliente": "Empresa C", "comentario": "Não entendi a cobrança desse mês na fatura de nuvem.", "data_envio": "2026-03-02"}
    ]

    job_config = bigquery.LoadJobConfig(
        write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
        autodetect=True
    )

    job = client.load_table_from_json(raw_data, table_id, job_config=job_config)
    job.result()
    print(f"✅ Camada BRONZE concluída: Tabela {table_id} atualizada.")

if __name__ == "__main__":
    run_bronze()
```

---

## 🥈 ETAPA 3: Camada Prata — Enriquecimento com Vertex AI (`02_silver_enrichment.py`)

A camada Prata é responsável pelo tratamento e pelo uso do **Gemini 2.5 Flash** (via Vertex AI SDK) para ler a linguagem natural dos comentários e extrair dados estruturados.

```python
import os
import pandas as pd
from google.cloud import bigquery
import vertexai
from vertexai.generative_models import GenerativeModel

def run_silver():
    project_id = os.getenv("GCP_PROJECT_ID")
    region = "us-central1"
    
    client = bigquery.Client(project=project_id)
    vertexai.init(project=project_id, location=region)
    model = GenerativeModel("gemini-2.5-flash")

    # 1. Leitura da Camada Bronze
    query_bronze = f"SELECT * FROM `{project_id}.db_feedbacks_medalhao.bronze_feedbacks_raw`"
    df = client.query(query_bronze).to_dataframe()

    # 2. Chamada ao Gemini para Enriquecimento
    categorias = []
    sentimentos = []

    for index, row in df.iterrows():
        prompt = f"""
        Analise o feedback abaixo e responda estritamente no formato "CATEGORIA|SENTIMENTO".
        Opções para CATEGORIA: [Suporte, Preço/Billing, Estabilidade, Usabilidade].
        Opções para SENTIMENTO: [Positivo, Neutro, Negativo].

        Comentário: "{row['comentario']}"
        """
        response = model.generate_content(prompt)
        res_text = response.text.strip()
        
        try:
            cat, sent = res_text.split('|')
            categorias.append(cat.strip())
            sentimentos.append(sent.strip())
        except:
            categorias.append("Não Identificado")
            sentimentos.append("Neutro")

    # 3. Adicionar colunas tratadas
    df['categoria_ia'] = categorias
    df['sentimento_ia'] = sentimentos
    df['processado_em'] = pd.Timestamp.now()

    # 4. Gravação na Camada Prata
    silver_table_id = f"{project_id}.db_feedbacks_medalhao.silver_feedbacks_cleaned"
    job_config = bigquery.LoadJobConfig(write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE)
    
    job = client.load_table_from_dataframe(df, silver_table_id, job_config=job_config)
    job.result()
    print(f"✅ Camada PRATA concluída: Tabela {silver_table_id} enriquecida via Vertex AI.")

if __name__ == "__main__":
    run_silver()
```

---

## 🥇 ETAPA 4: Camada Ouro — Agregação e Regras de Negócio (`03_gold_aggregation.sql`)

A camada Ouro compõe as tabelas analíticas. Aqui é aplicada a **Análise de Pareto (Princípio 80/20)** sobre os feedbacks negativos.

```sql
CREATE OR REPLACE TABLE `db_feedbacks_medalhao.gold_pareto_feedbacks` AS
WITH Agrupado AS (
    SELECT 
        categoria_ia,
        COUNT(1) AS total_chamados
    FROM `db_feedbacks_medalhao.silver_feedbacks_cleaned`
    WHERE sentimento_ia = 'Negativo'
    GROUP BY categoria_ia
),
CalculoAcumulado AS (
    SELECT 
        categoria_ia,
        total_chamados,
        SUM(total_chamados) OVER() AS total_geral,
        SUM(total_chamados) OVER(
            ORDER BY total_chamados DESC 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS acumulado
    FROM Agrupado
)
SELECT 
    categoria_ia,
    total_chamados,
    ROUND((total_chamados / total_geral) * 100, 2) AS pct_individual,
    ROUND((acumulado / total_geral) * 100, 2) AS pct_acumulado
FROM CalculoAcumulado
ORDER BY total_chamados DESC;
```

---

## 📊 ETAPA 5: Visualização Interativa com Python no Vertex AI Workbench

Dentro da instância do **Vertex AI Workbench** (JupyterLab), consumimos a camada Ouro com **Plotly**. Crie o arquivo `notebooks/04_visualizacao.ipynb` e insira o código:

```python
import os
import pandas as pd
from google.cloud import bigquery
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# O BigQuery Client recupera o projeto das credenciais nativas da instância
client = bigquery.Client()

query = """
SELECT categoria_ia, total_chamados, pct_acumulado
FROM `db_feedbacks_medalhao.gold_pareto_feedbacks`
ORDER BY total_chamados DESC
"""

df = client.query(query).to_dataframe()

# Renderização do Gráfico Duplo de Pareto (Barras + Linha Acumulada)
fig = make_subplots(specs=[[{"secondary_y": True}]])

# Adiciona as Barras (Frequência por Categoria)
fig.add_trace(
    go.Bar(
        x=df['categoria_ia'], 
        y=df['total_chamados'], 
        name="Total de Reclamações", 
        marker_color='#6366f1'
    ),
    secondary_y=False
)

# Adiciona a Linha (Porcentagem Acumulada)
fig.add_trace(
    go.Scatter(
        x=df['categoria_ia'], 
        y=df['pct_acumulado'], 
        name="% Acumulado (Pareto)", 
        mode="lines+markers", 
        line=dict(color='#ef4444', width=3)
    ),
    secondary_y=True
)

fig.update_layout(
    title_text="<b>Gráfico de Pareto: Ofensores de Experiência do Cliente</b>",
    template="plotly_dark",
    legend=dict(x=0.8, y=1.1)
)

fig.update_yaxes(title_text="Quantidade de Feedbacks Negativos", secondary_y=False)
fig.update_yaxes(title_text="% Acumulado", range=[0, 105], secondary_y=True)

fig.show()
```

---

## 🚀 ETAPA 6: CI/CD com GitHub Actions e Workload Identity Federation

Configure o pipeline em `.github/workflows/deploy_pipeline.yml`. Ele executa a arquitetura utilizando o token **WIF sem expor senhas**.

```yaml
name: Deploy Pipeline de Dados Medalhao GCP

on:
  push:
    branches: [ "main" ]

jobs:
  run-pipeline:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Necessário para autenticação OIDC via WIF

    steps:
    - name: Checkout Código
      uses: actions/checkout@v3

    - name: Autenticar no GCP via Workload Identity Federation
      uses: google-github-actions/auth@v1
      with:
        workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
        service_account: ${{ secrets.GCP_SA_EMAIL }}

    - name: Configurar Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'

    - name: Instalar Dependências
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt

    - name: Executar Pipeline Medalhão
      env:
        GCP_PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
      run: |
        python src/01_bronze_ingestion.py
        python src/02_silver_enrichment.py
        # Executa a query de agregação Ouro
        python -c "
        import os
        from google.cloud import bigquery
        client = bigquery.Client(project=os.getenv('GCP_PROJECT_ID'))
        with open('src/03_gold_aggregation.sql', 'r') as f:
            query = f.read()
        client.query(query).result()
        print('✅ Camada OURO compilada via CI/CD!')
        "
```