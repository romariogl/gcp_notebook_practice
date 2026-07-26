\# 🚀 Prática Guiada GCP: Ingestão no BigQuery, Classificação via LLM (Vertex AI) e Pareto



Este repositório contém o guia passo a passo para a realização da prática guiada de Ciência de Dados no Google Cloud Platform. 



Nesta atividade, você aprenderá a criar um ambiente analítico na nuvem, importar dados via API, processar textos usando o \*\*Gemini 1.5 Flash (LLM)\*\* e construir uma visualização interativa com \*\*Plotly\*\*.



\---



\## 📌 Sumário

1\. \[Criando o Dataset no BigQuery (Interface Visual)](#1-criando-o-dataset-no-bigquery-interface-visual)

2\. \[Abrindo o Vertex AI Workbench (Interface Visual)](#2-abrindo-o-vertex-ai-workbench-interface-visual)

3\. \[Configurando e Conectando o Notebook (Sem Chaves/JSON)](#3-configurando-e-conectando-o-notebook)

4\. \[Executando o Roteiro da Prática](#4-executando-o-roteiro-da-prática)



\---



\## 1. Criando o Dataset no BigQuery (Interface Visual)



Antes de rodar qualquer código, precisamos de um "banco de dados" (dataset) para salvar nossas tabelas no BigQuery.



1\. Acesse o \[Console do Google Cloud](https://console.cloud.google.com/).

2\. Na barra de pesquisa superior, digite \*\*BigQuery\*\* e selecione o serviço \*\*BigQuery\*\*.

3\. No menu lateral esquerdo (painel \*Explorer\*):

&#x20;  \* Localize o ID do seu projeto GCP.

&#x20;  \* Clique nos \*\*três pontos verticais (⋮)\*\* ao lado do ID do projeto e selecione \*\*Criar conjunto de dados\*\* (\*Create dataset\*).

4\. Na tela de criação, preencha:

&#x20;  \* \*\*ID do conjunto de dados\*\*: `dataset\_pratica`

&#x20;  \* \*\*Local dos dados (Region)\*\*: Selecione `us-central1` (ou a região informada pelo instrutor).

5\. Clique no botão azul \*\*Criar conjunto de dados\*\* (\*Create dataset\*) no rodapé da página.



\---



\## 2. Abrindo o Vertex AI Workbench (Interface Visual)



Utilizaremos uma instância gerenciada de JupyterLab hospedada diretamente no Google Cloud.



1\. No menu de navegação do GCP (canto superior esquerdo) ou na barra de busca, acesse \*\*Vertex AI\*\*.

2\. No menu lateral da Vertex AI, clique em \*\*Workbench\*\*.

3\. Na aba \*\*Notebooks gerenciados pelo usuário\*\* (\*User-managed notebooks\*) ou \*\*Instâncias\*\*:

&#x20;  \* Localize a instância criada para a prática.

&#x20;  \* Clique no botão \*\*ABRIR O JUPYTERLAB\*\* (\*OPEN JUPYTERLAB\*) ao lado do nome da máquina.

4\. Uma nova aba no navegador será aberta com a interface do JupyterLab.

5\. \*\*Para clonar este repositório no Workbench:\*\*

&#x20;  \* No JupyterLab, abra o \*\*Terminal\*\* (menu \*File > New > Terminal\*).

&#x20;  \* Cole o comando de clone do repositório:

&#x20;    ```bash

&#x20;    git clone \[https://github.com/romariogl/gcp\_notebook\_practice.git](https://github.com/romariogl/gcp\_notebook\_practice.git)

&#x20;    ```

&#x20;  \* Abra a pasta clonada no menu lateral esquerdo e acesse o arquivo `.ipynb`.



\---



\## 3. Configurando e Conectando o Notebook



Uma das grandes vantagens de executar o código \*\*dentro do Vertex AI Workbench\*\* é a \*\*autenticação nativa\*\*: você \*\*NÃO precisa\*\* criar chaves JSON, importar arquivos `.env` ou expor senhas. 



O Workbench utiliza a \*\*Service Account\*\* associada à máquina virtual. O SDK do Python detecta essas credenciais automaticamente.



\### Como garantir que tudo conecte corretamente:



Verifique se a primeira célula de código do seu notebook inicializa as SDKs do GCP indicando o seu `PROJECT\_ID` e a região correspondente:



```python

import vertexai

from google.cloud import bigquery



\# CONFIGURAÇÕES DO AMBIENTE

\# Substitua 'seu-projeto-gcp-id' pelo ID real do projeto que você está utilizando

PROJECT\_ID = "seu-projeto-gcp-id"  

LOCATION = "us-central1"



\# 1. Conexão com o BigQuery (Autenticação automática do Workbench)

client = bigquery.Client(project=PROJECT\_ID)



\# 2. Conexão com o Vertex AI e LLM Gemini (Autenticação automática)

vertexai.init(project=PROJECT\_ID, location=LOCATION)



print("✅ Conexão estabelecida com sucesso com o BigQuery e Vertex AI!")

