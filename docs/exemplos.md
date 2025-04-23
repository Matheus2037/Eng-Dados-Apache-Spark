# Exemplos Práticos

Esta seção apresenta exemplos práticos implementados neste projeto, demonstrando o uso do Apache Spark com Delta Lake e Apache Iceberg para gerenciamento de dados.

## Exemplo com Delta Lake

Este exemplo demonstra as operações básicas com o Delta Lake, conforme implementado no notebook `pyspark-delta/delta.ipynb`.

### Configuração do Ambiente Spark com Delta Lake

```python
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, FloatType
from delta import *

spark = ( 
    SparkSession
    .builder
    .master("local[*]")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate() 
)
```

### Criação do DataFrame com Dados de Clientes

```python
data = [
    ("ID001", "CLIENTE_X","SP","ATIVO",   250000.00),
    ("ID002", "CLIENTE_Y","SC","INATIVO", 400000.00),
    ("ID003", "CLIENTE_Z","DF","ATIVO",   1000000.00)
]

schema = (
    StructType([
        StructField("ID_CLIENTE",     StringType(),True),
        StructField("NOME_CLIENTE",   StringType(),True),
        StructField("UF",             StringType(),True),
        StructField("STATUS",         StringType(),True),
        StructField("LIMITE_CREDITO", FloatType(), True)
    ])
)

df = spark.createDataFrame(data=data,schema=schema)
df.show(truncate=False)
```

### Criação da Tabela Delta

```python
( 
    df
    .write
    .format("delta")
    .mode('overwrite')
    .save("./RAW/CLIENTES")
)
```

### Operações de Insert/Update usando Merge

```python
# Dados para merge (atualização e inserção)
new_data = [
    ("ID001","CLIENTE_X","SP","INATIVO", 0.00),       # Atualização - cliente existente
    ("ID002","CLIENTE_Y","SC","ATIVO",   400000.00),  # Atualização - cliente existente
    ("ID004","CLIENTE_Z","DF","ATIVO",   5000000.00)  # Inserção - novo cliente
]

# Criar DataFrame com os novos dados
df_new = spark.createDataFrame(data=new_data, schema=schema)

# Obter referência à tabela Delta
deltaTable = DeltaTable.forPath(spark, "./RAW/CLIENTES")

# Executar operação de merge
(
    deltaTable.alias("dados_atuais")
    .merge(
        df_new.alias("novos_dados"),
        "dados_atuais.ID_CLIENTE = novos_dados.ID_CLIENTE"
    )
    .whenMatchedUpdateAll()    # Atualiza todos os campos quando encontra correspondência
    .whenNotMatchedInsertAll() # Insere novo registro quando não encontra correspondência
    .execute()
)
```

### Operação de Delete

```python
# Deletar registros com condição (limite de crédito menor que 400000.0)
deltaTable.delete("LIMITE_CREDITO < 400000.0")
```

## Exemplo com Apache Iceberg

Este exemplo demonstra as operações básicas com o Apache Iceberg, conforme implementado no notebook `pyspark-iceberg/pyspark-iceberg.ipynb`.

### Configuração do Ambiente Spark com Apache Iceberg

```python
from pyspark.sql import SparkSession

spark = (
    SparkSession.builder
    .appName("IcebergExample")
    .master("local[*]")
    .config("spark.jars.packages", "org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.3.0")
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.spark_catalog", "org.apache.iceberg.spark.SparkSessionCatalog")
    .config("spark.sql.catalog.spark_catalog.type", "hadoop")
    .config("spark.sql.catalog.spark_catalog.warehouse", "./warehouse")
    .getOrCreate()
)
```

### Criação do DataFrame com Dados de Clientes

```python
from pyspark.sql.types import StructType, StructField, StringType, FloatType

data = [
    ("ID001", "CLIENTE_X", "SP", "ATIVO",   250000.00),
    ("ID002", "CLIENTE_Y", "SC", "INATIVO", 400000.00),
    ("ID003", "CLIENTE_Z", "DF", "ATIVO",   1000000.00)
]

schema = StructType([
    StructField("ID_CLIENTE", StringType(), True),
    StructField("NOME_CLIENTE", StringType(), True),
    StructField("UF", StringType(), True),
    StructField("STATUS", StringType(), True),
    StructField("LIMITE_CREDITO", FloatType(), True)
])

df = spark.createDataFrame(data=data, schema=schema)
df.show()
```

### Criação da Tabela Iceberg

```python
# Criar tabela Iceberg usando a API DataFrame
df.writeTo("spark_catalog.default.clientes_iceberg").using("iceberg").createOrReplace()
```

### Operação de Insert (Append)

```python
# Dados para inserção
new_data = [
    ("ID004", "CLIENTE_NEW", "RJ", "ATIVO", 999999.00)
]

# Criar DataFrame com os novos dados
df_new = spark.createDataFrame(data=new_data, schema=schema)

# Inserir os novos dados na tabela Iceberg
df_new.writeTo("spark_catalog.default.clientes_iceberg").append()
```

### Operação de Update usando SQL

```python
# Atualizar registros usando SQL
spark.sql("""
    UPDATE spark_catalog.default.clientes_iceberg
    SET STATUS = 'INATIVO', LIMITE_CREDITO = 0.00
    WHERE ID_CLIENTE = 'ID001'
""")
```

### Operação de Delete usando SQL

```python
# Excluir registros usando SQL
spark.sql("""
    DELETE FROM spark_catalog.default.clientes_iceberg
    WHERE LIMITE_CREDITO < 400000.0
""")
```

### Operação de Consulta usando SQL

```python
# Consultar dados da tabela Iceberg
spark.sql("SELECT * FROM spark_catalog.default.clientes_iceberg").show()
```
