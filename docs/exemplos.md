# Exemplos Práticos

Esta seção apresenta exemplos práticos e casos de uso realistas para utilização do Apache Spark com Delta Lake em projetos de engenharia de dados.

## Workflows Completos de Processamento de Dados

### Exemplo 1: Pipeline ETL para Dados de Vendas

Este exemplo demonstra um pipeline ETL (Extract, Transform, Load) completo para processar dados de vendas:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, date_format, sum as spark_sum, when, current_timestamp
from delta.tables import DeltaTable

# Configuração da sessão Spark
spark = (
    SparkSession.builder
    .appName("ETL Pipeline de Vendas")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# 1. EXTRAÇÃO - Carregar dados de diferentes fontes
# Dados de vendas (simulando CSV)
vendas_df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("./dados/vendas.csv")

# Dados de produtos (simulando JSON)
produtos_df = spark.read.format("json") \
    .load("./dados/produtos.json")

# Dados de clientes (simulando Delta existente)
clientes_df = spark.read.format("delta") \
    .load("./RAW/CLIENTES")

# 2. TRANSFORMAÇÃO - Processar e enriquecer os dados
# Limpeza de dados
vendas_limpas_df = vendas_df \
    .dropDuplicates(["ID_VENDA"]) \
    .filter(col("VALOR") > 0) \
    .withColumn("DATA", date_format(col("DATA"), "yyyy-MM-dd"))

# Join com dados de produtos
vendas_com_produtos_df = vendas_limpas_df \
    .join(produtos_df, vendas_limpas_df.ID_PRODUTO == produtos_df.ID, "left") \
    .select(
        vendas_limpas_df["*"], 
        col("NOME_PRODUTO"), 
        col("CATEGORIA"),
        col("PRECO_UNITARIO")
    )

# Join com dados de clientes
vendas_completas_df = vendas_com_produtos_df \
    .join(clientes_df, vendas_com_produtos_df.ID_CLIENTE == clientes_df.ID_CLIENTE, "left") \
    .select(
        vendas_com_produtos_df["*"],
        col("NOME_CLIENTE"),
        col("UF")
    )

# Agregações para análise
resumo_por_regiao = vendas_completas_df \
    .groupBy("UF") \
    .agg(
        spark_sum("VALOR").alias("VALOR_TOTAL"),
        spark_sum(when(col("CATEGORIA") == "Eletrônicos", col("VALOR")).otherwise(0)).alias("VALOR_ELETRONICOS")
    )

# 3. CARGA - Salvar dados processados em formato Delta
# Salvar dados detalhados
(
    vendas_completas_df
    .write
    .format("delta")
    .mode("overwrite")
    .partitionBy("UF", "DATA")  # Particionar por região e data
    .save("./SILVER/VENDAS_DETALHADAS")
)

# Salvar agregações
(
    resumo_por_regiao
    .write
    .format("delta")
    .mode("overwrite")
    .save("./GOLD/RESUMO_VENDAS_REGIAO")
)

# 4. AUDITORIA - Registrar metadados do processo
audit_data = [
    (
        "PIPELINE_VENDAS", 
        current_timestamp(), 
        vendas_df.count(), 
        vendas_completas_df.count(),
        "SUCCESS"
    )
]

audit_df = spark.createDataFrame(
    audit_data, 
    ["PIPELINE_ID", "TIMESTAMP", "REGISTROS_ENTRADA", "REGISTROS_SAIDA", "STATUS"]
)

(
    audit_df
    .write
    .format("delta")
    .mode("append")
    .save("./METADATA/AUDIT_LOGS")
)

print("Pipeline ETL de vendas concluído com sucesso!")
```

### Exemplo 2: Processo de Incremento Diário de Dados

Este exemplo demonstra como implementar um processo de carga incremental diária:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, current_date, date_sub
from delta.tables import DeltaTable

# Configuração da sessão Spark
spark = (
    SparkSession.builder
    .appName("Carga Incremental")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# Data para filtragem incremental
data_atual = current_date()
data_inicio = date_sub(data_atual, 1)  # Dados de ontem

# Carregar apenas registros novos da fonte
novos_dados = (
    spark.read.format("jdbc")
    .option("url", "jdbc:postgresql://localhost:5432/database")
    .option("dbtable", "vendas")
    .option("user", "usuario")
    .option("password", "senha")
    .option("driver", "org.postgresql.Driver")
    .option("query", f"SELECT * FROM vendas WHERE data_venda >= '{data_inicio}'")
    .load()
)

# Verificar se a tabela Delta já existe
try:
    deltaTable = DeltaTable.forPath(spark, "./DELTA/VENDAS_HISTORICO")
    
    # Executar merge para atualizar registros existentes e inserir novos
    (
        deltaTable.alias("destino")
        .merge(
            novos_dados.alias("origem"),
            "destino.ID_VENDA = origem.ID_VENDA"
        )
        .whenMatchedUpdateAll()
        .whenNotMatchedInsertAll()
        .execute()
    )
    
    print(f"Carga incremental concluída. {novos_dados.count()} registros processados.")
    
except:
    # Se a tabela não existir, criar pela primeira vez
    (
        novos_dados
        .write
        .format("delta")
        .mode("overwrite")
        .save("./DELTA/VENDAS_HISTORICO")
    )
    
    print(f"Primeira carga concluída. {novos_dados.count()} registros processados.")

# Otimizar a tabela após a carga
spark.sql("OPTIMIZE delta.`./DELTA/VENDAS_HISTORICO`")
```

## Casos de Uso Comuns em Engenharia de Dados

### Caso 1: Processamento de Dados de Sensores IoT

Este caso de uso demonstra como processar dados de sensores IoT que chegam em lotes frequentes:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json, window, avg, max as spark_max, min as spark_min
from pyspark.sql.types import StructType, StructField, StringType, FloatType, TimestampType
from delta.tables import DeltaTable

# Configuração da sessão Spark
spark = (
    SparkSession.builder
    .appName("Processamento IoT")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# Definir schema dos dados de sensores
schema_sensores = StructType([
    StructField("device_id", StringType(), False),
    StructField("timestamp", TimestampType(), False),
    StructField("temperatura", FloatType(), True),
    StructField("umidade", FloatType(), True),
    StructField("pressao", FloatType(), True),
    StructField("localizacao", StringType(), True)
])

# Carregar lote de dados de sensores
dados_sensores = (
    spark.read.format("json")
    .schema(schema_sensores)
    .load("./dados/sensores_lote_atual.json")
)

# Processar dados - calcular médias por janela de tempo e dispositivo
metricas_por_dispositivo = (
    dados_sensores
    .groupBy(
        "device_id",
        window(col("timestamp"), "1 hour")
    )
    .agg(
        avg("temperatura").alias("temp_media"),
        spark_max("temperatura").alias("temp_max"),
        spark_min("temperatura").alias("temp_min"),
        avg("umidade").alias("umidade_media"),
        avg("pressao").alias("pressao_media")
    )
)

# Tabela delta para armazenar dados raw
(
    dados_sensores
    .write
    .format("delta")
    .mode("append")
    .partitionBy("device_id")
    .save("./IOT/RAW_SENSORES")
)

# Tabela delta para armazenar métricas agregadas
(
    metricas_por_dispositivo
    .write
    .format("delta")
    .mode("append")
    .partitionBy("device_id")
    .save("./IOT/METRICAS_SENSORES")
)

# Detectar anomalias (exemplo: temperatura muito alta)
anomalias = (
    dados_sensores
    .filter(col("temperatura") > 90.0)
    .select("device_id", "timestamp", "temperatura", "localizacao")
)

if anomalias.count() > 0:
    (
        anomalias
        .write
        .format("delta")
        .mode("append")
        .save("./IOT/ANOMALIAS")
    )
    print(f"Detectadas {anomalias.count()} anomalias de temperatura!")
```

### Caso 2: Data Warehouse Incremental

Este exemplo demonstra como construir e manter um data warehouse usando o padrão SCD (Slowly Changing Dimension) Tipo 2 com Delta Lake:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, current_timestamp, lit
from delta.tables import DeltaTable

# Configuração da sessão Spark
spark = (
    SparkSession.builder
    .appName("Data Warehouse SCD2")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# Carregar novos dados da dimensão clientes
novos_clientes = (
    spark.read.format("csv")
    .option("header", "true")
    .option("inferSchema", "true")
    .load("./dados/novos_clientes.csv")
    .withColumn("data_atualizacao", current_timestamp())
    .withColumn("ativo", lit(True))
)

# Implementar SCD Tipo 2 (mantém histórico de mudanças)
try:
    # Verificar se tabela dimensão já existe
    dim_clientes = DeltaTable.forPath(spark, "./DW/DIM_CLIENTES")
    
    # Identificar registros novos e alterados
    # Atualizações são identificadas por chave igual, mas valores diferentes
    atualizado_df = dim_clientes.toDF().alias("atual").join(
        novos_clientes.alias("novo"),
        "ID_CLIENTE"
    ).where(
        """
        atual.ativo = true AND
        (
            atual.NOME_CLIENTE != novo.NOME_CLIENTE OR
            atual.UF != novo.UF OR
            atual.LIMITE_CREDITO != novo.LIMITE_CREDITO
        )
        """
    ).select("atual.ID_CLIENTE")
    
    # Expirar registros atuais (marcar como inativos)
    (
        dim_clientes.alias("atual")
        .merge(
            atualizado_df.alias("atualizado"),
            "atual.ID_CLIENTE = atualizado.ID_CLIENTE AND atual.ativo = true"
        )
        .whenMatched()
        .updateExpr({"ativo": "false", "data_fim": "current_timestamp()"})
        .execute()
    )
    
    # Inserir todos os novos registros
    (
        novos_clientes
        .write
        .format("delta")
        .mode("append")
        .save("./DW/DIM_CLIENTES")
    )
    
except Exception as e:
    # Se a tabela não existir, criar pela primeira vez
    # Adicionar colunas de controle para SCD Tipo 2
    (
        novos_clientes
        .withColumn("data_inicio", current_timestamp())
        .withColumn("data_fim", lit(None).cast("timestamp"))
        .write
        .format("delta")
        .mode("overwrite")
        .save("./DW/DIM_CLIENTES")
    )

# Otimizar a tabela após as operações
spark.sql("OPTIMIZE delta.`./DW/DIM_CLIENTES`")
```

## Integração com Diferentes Fontes de Dados

### Exemplo 1: Integração com Bancos de Dados Relacionais

Este exemplo demonstra como integrar dados de um banco PostgreSQL com Delta Lake:

```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable

# Configuração da sessão Spark
spark = (
    SparkSession.builder
    .appName("Integração PostgreSQL")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0,org.postgresql:postgresql:42.3.1")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# Carregar dados do PostgreSQL
jdbc_params = {
    "url": "jdbc:postgresql://localhost:5432/meu_banco",

