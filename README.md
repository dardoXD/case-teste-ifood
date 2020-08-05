# iFood Data Architect Test

O objetivo desse projeto é resolver o case apresentado seguindo as diretrizes abaixo:

## Problema Apresentado
Process semi-structured data and build a datalake that provides efficient storage and performance. The datalake must be organized in the following 2 layers:
* raw layer: Datasets must have the same schema as the source, but support fast structured data reading
* trusted layer: datamarts as required by the analysis team

![Datalake](./datalake.png)

### Requirements

* Source files:
  * Order: s3://ifood-data-architect-test-source/order.json.gz
  * Order Statuses: s3://ifood-data-architect-test-source/status.json.gz
  * Restaurant: s3://ifood-data-architect-test-source/restaurant.csv.gz
  * Consumer: s3://ifood-data-architect-test-source/consumer.csv.gz 
* Raw Layer (same schema from the source):
  * Order dataset.
  * Order Statuses dataset.
  * Restaurant dataset.
  * Consumer dataset.
* Trusted Layer:
  * Order dataset -  one line per order with all data from order, consumer, restaurant and the LAST status from order statuses dataset. To help analysis, it would be a nice to have: data partitioned on the restaurant LOCAL date.
  * Order Items dataset - easy to read dataset with one-to-many relationship with Order dataset. Must contain all data from _order_ items column.
  * Order statuses - Dataset containing one line per order with the timestamp for each registered event: CONCLUDED, REGISTERED, CANCELLED, PLACED.
* For the trusted layer, anonymize any sensitive data.
* At the end of each ETL, use any appropriated methods to validate your data.
* Read performance, watch out for small files and skewed data.

### Non functional requirements
* Data volume increases each day. All ETLs must be built to be scalable.
* Use any data storage you feel comfortable to.
* Document your solution.

## Resolução

### Arquitetura Sugerida

Seguindo a necessidade de armazenamento dos dados, de escalabidade e facilidade de leitura, a arquitetura abaixo é a sugerida:

![Sugestion_Datalake](./sugestion_datalake.png)

* Como pode ser observado na imagem, como os arquivos estão sendo retirados de um S3 da Amazon, resolvi manter a mesma estrutura cloud para o armazenamento dos arquivos e construção do Data Lake, porém seria possível implementar a mesma ideia em outras plataformas cloud, como por exemplo, utilizando o Azure Data Lake.

* Além disso, os arquivos Trusted, dependendo da estrutura da empresa com relação a ferramentas de visualização de dados, podem ser armazenado em um Data Warehouse, no caso da Amazon utilizando o Redshift e no caso da Azure utilizando o Azure Data Warehouse mas também podem podem utilizar o S3 para fazer esse armazenamento em parquet.

* O armazenamento dos arquivos em formato parquet foi utilizada pela necessidade de estruturação dos dados e pelo formato ser otimizado para leitura. Além disso, a maioria das ferramentas de visualização conseguem utilizar o formato para criação de dashboards e análises.

* Todas as ferramentas sugeridas são facilmente escaláveis e conseguem ser integradas com outros sistemas, como por exemplo, o Databricks que consegue ler arquivos de diversas fontes e manipulá-los utilizando diversas linguagens de programação.

* Outras ferramentas de ETL podem ser utilizadas, como o Amazon Glue ou o Azure Data Factory, dependendo da necessidade e recursos da plataforma.

### Estrutura do Data Lake

Seguindo o modelo sugerido construído utilizando as ferramentas da Amazon, teríamos a seguinte distribuição de pastas dentro do bucket no Amazon S3:

<img src="./structure_datalake.png" width="300" height="400" /> 

* **RAW**: Diretório onde serão armazenados os arquivos brutos vindos dos sistemas. Podem ser recebidos em diversos formatos mas se possível já convertido em parquet.

* **CURATED**: Diretório onde serão armazenados os arquivos com algum tipo de modificação ou conversão.
  * **CONVERTED**: Diretório onde serão armazenados os arquivos brutos convertidos que não puderam ser convertidos durante a extração do source.
  * **ENRICHED**: Diretório onde serão armazenados os arquivos com cruzamento de dados ou modelos analíticos. Aqui podem ser armazenados os arquivos trusted para análise.
  
* **LAB ZONE**: Diretório utilizado para a realização de testes e análises exploratórias pelos cientistas de dados.

* **TRANSIENT**: Diretório para armazenar arquivos temporários.

Dentro de cada diretório deve haver separações por em base, ano, mês e dia, devido ao controle de entrada de arquivos e separações por bases. Além disso, é recomendável que o nome dos sejam no formato YYYYmmddHHMMSS-nomedoarquivo.parquet dependendo do número de partições.

Exemplo: \<nome-do-bucket\>/raw/order/2020/08/05/20200805000000-order.parquet

### ETL

Conforme citado acima, utilizei o Databricks para fazer o processamento dos dados e a linguagem python para facilitar o tratamento das informações. Além disso, foi utilizada a biblioteca pyspark para paralelizar o processamento no cluster do Databricks e otimizar o processamento.

O código foi dividido em 4 pastas:
* **Datasets**: Diretório onde está a classe que processa os datasets, servirá para a leitura e escrita dos arquivos no S3 da Amazon.
* **Executables**: Diretório que contém os notebooks que fazem a execução dos 3 principais passos para o processo de ETL e funcionamento do Data Lake.
* **Flows**: Diretório onde serão armazenados os códigos para o cálculo de cada base trusted.
* **Utils**: Diretório que armazena funções úteis que serão utilizadas no código dos demais diretórios.

Como o intuito do projeto era demonstrar o funcionamento do processo de ETL em conjunto com a construção do Data Lake, o armazenamento dos arquivos trusted não foi realizado no Data Warehouse e sim no próprio S3 da Amazon, como é o recomendado, mas funcionaria da mesma forma, só alterando a conexão.

### Pré-Requisitos

Antes de executar os Notebooks do Databricks é necessário criar uma conta na Amazon, pegar a access key e a secret key do usuário, e criar um bucket.

* Tutorial para pegar a access key e a secret key: https://supsystic.com/documentation/id-secret-access-key-amazon-s3/
* Tutorial para criar bucket na AWS: https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html

Depois desses passos, insira essas informações como as seguintes secrets do Databricks no escopo "aws":
* **aws-access-key** = access key
* **aws-secret-key** = secret key
* **aws-bucket-name** = nome do bucket criado

* Tutorial de como inserir secrets no Databricks: https://docs.databricks.com/security/secrets/index.html

### Funcionamento

Para executar o código, a primeira etapa é cadastrar um novo Dataset, para isso é preciso executar o Notebook contido na pasta Executables chamado RegisterDataset passando os seguintes parâmetros:
* **bucket_name**: Nome do bucket onde está armazenado o arquivo.
* **delimiter**: Delimitador utilizado, caso não haja, deixe em branco.
* **extension**: Extensão do arquivo utilizado. Pode ser json, csv ou parquet.
* **name**: Nome que será dado ao Dataset.
* **path**: Caminho no bucket onde está armazenado o arquivo.

Após a execução com sucesso do cadastramento, para que o dataset seja lido do source e armazenado na raw do Data Lake, é só executar o Notebook na pasta Executables chamado de RunDatasets passando o seguinte parâmetro:
* **datasets**: Nome do Dataset que deverá ser carregado na raw. Pode ser passado mais de um dataset separado por virgula.
  * Exemplo 1: Order
  * Exemplo 2: Order,OrderStatuses

Se o código foi executado corretamente, o arquivo do Source já estará na pasta correta da raw no seu Data Lake.

Para gerar as bases trusted, primeiro será necessário construir o código fazendo o cruzamento dos datasets e as transformações necessárias.
* Entre na pasta Flows e criei um novo Notebook com o nome **FlowNomeDaBase**
  * Por exemplo, FlowOrderStatuses
* Crie uma classe com o mesmo nome do Notebook gerado e como parâmetro herde a classe **FlowBase**
  ```python
  class FlowOrderStatuses(FlowBase):
  ```
* Implemente o método run com as transformações e o cruzamento de datasets necessários.
  * Muitas funções já foram implementadas e podem ser utilizadas para facilitar essa manipulação, principalmente, para leitura e escrita dos Datasets.
* O retorno do método run deve ser um dataframe spark no formato que será gravado no Data Lake para análise.

Outras informações e documentações podem ser encontradas nos Notebooks do Databricks.
