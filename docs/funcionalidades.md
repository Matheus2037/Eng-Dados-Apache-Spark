# Funcionalidades

Este documento detalha as principais funcionalidades implementadas no projeto de Engenharia de Dados com Apache Spark, com foco especial nas operações utilizando Delta Lake.

## Visão Geral das Operações com Delta Lake

O Delta Lake é uma camada de armazenamento que adiciona recursos essenciais aos data lakes, trazendo confiabilidade e performance para operações de dados em larga escala. Este projeto demonstra a implementação das seguintes capacidades:

- **ACID Transactions**: Garantia de atomicidade, consistência, isolamento e durabilidade para todas as operações de dados
- **Schema Enforcement**: Prevenção automática contra inserção de dados inválidos
- **Schema Evolution**: Modificação flexível do esquema ao longo do tempo
- **Time Travel**: Acesso a versões anteriores dos dados
- **CRUD Operations**: Suporte completo a Create, Read, Update e Delete 
- **Merge Operations**: Operações de UPSERT eficientes (update + insert)
- **Data Optimization**: Compactação de arquivos e otimização de consultas

## Operações CRUD com Delta Lake

### 1. Criação de Tabelas Delta (Create)

O Delta Lake permite criar tabelas a partir de DataFrames Spark com sintaxe simples, adicionando garantias de transação e capacidades avançadas.

#### Exemplo de Criação:

```python
# Importações necessárias
from pyspark.sql import SparkSession
from pyspark.sql.types import StructType, StructField, StringType, FloatType
from delta import *

# Configuração da sessão Spark com suporte ao Delta Lake
spark = (
    SparkSession
    .builder
    .master("local[*]")
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0")
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
)

# Dados de exemplo
data = [
    ("ID001", "CLIENTE_X", "SP", "ATIVO",   250000.00),
    ("ID002", "CLIENTE_Y", "SC", "INATIVO", 400000.00),
    ("ID003", "CLIENTE_Z", "DF", "ATIVO",   1000000.00)
]

# Definição do schema
schema = (
    StructType([
        StructField("ID_CLIENTE",     StringType(), True),
        StructField("NOME_CLIENTE",   StringType(), True),
        StructField("UF",             StringType(), True),
        StructField("STATUS",         StringType(), True),
        StructField("LIMITE_CREDITO", FloatType(),  True)
    ])
)

# Criação do DataFrame
df = spark.createDataFrame(data=data, schema=schema)

# Escrita no formato Delta
(
    df
    .write
    .format("delta")
    .mode("overwrite")  # Opções: append, overwrite, ignore, errorIfExists
    .save("./RAW/CLIENTES")
)
```

!!! note "Modos de Escrita"
    O Delta Lake suporta diferentes modos de escrita:
    - **overwrite**: Substitui dados existentes
    - **append**: Adiciona novos dados sem modificar existentes
    - **ignore**: Não faz nada se o destino já existir
    - **errorIfExists**: Falha se o destino já existir

### 2. Leitura de Tabelas Delta (Read)

A leitura de tabelas Delta é feita com a mesma API do Spark, mas com benefícios adicionais como consistência de leitura e capacidade de acessar versões anteriores.

#### Exemplo de Leitura:

```python
# Leitura simples de tabela Delta
df_clientes = (
    spark
    .read
    .format("delta")
    .load("./RAW/CLIENTES")
)

# Exibir resultados
df_clientes.show()
```

#### Leitura de Versão Específica (Time Travel):

```python
# Ler versão específica da tabela
df_clientes_versao_anterior = (
    spark
    .read
    .format("delta")
    .option("versionAsOf", 0)  # Versão 0 (inicial)
    .load("./RAW/CLIENTES")
)

# Alternativa usando timestamp
df_clientes_timestamp = (
    spark
    .read
    .format("delta")
    .option("timestampAsOf", "2023-01-01 00:00:00")
    .load("./RAW/CLIENTES")
)
```

### 3. Atualização de Dados (Update)

O Delta Lake suporta atualizações de dados de forma eficiente, permitindo modificar registros existentes sem reescrever toda a tabela.

#### Exemplo de Atualização:

```python
from delta.tables import DeltaTable

# Carregar tabela Delta para operações
deltaTable = DeltaTable.forPath(spark, "./RAW/CLIENTES")

# Atualizar registros
(
    deltaTable.update(
        condition="STATUS = 'ATIVO'",
        set={"LIMITE_CREDITO": "LIMITE_CREDITO * 1.1"}  # Aumento de 10% no limite
    )
)
```

### 4. Exclusão de Dados (Delete)

Operações de exclusão permitem remover registros que atendem a determinadas condições.

#### Exemplo de Exclusão:

```python
# Deletar registros com condição
deltaTable.delete("LIMITE_CREDITO < 400000.0")
```

## Merge Operations (UPSERT)

Uma das operações mais poderosas do Delta Lake é o `merge`, que permite combinar operações de atualização e inserção em uma única transação atômica.

### Exemplo de MERGE:

```python
# Novos dados para merge
new_data = [
    ("ID001", "CLIENTE_X", "SP", "INATIVO", 0.00),         # Update - cliente existente
    ("ID002", "CLIENTE_Y", "SC", "ATIVO",   400000.00),    # Update - cliente existente
    ("ID004", "CLIENTE_Z", "DF", "ATIVO",   5000000.00)    # Insert - novo cliente
]

# Criar DataFrame com novos dados
df_new = spark.createDataFrame(data=new_data, schema=schema)

# Carregar tabela Delta
deltaTable = DeltaTable.forPath(spark, "./RAW/CLIENTES")

# Executar operação de merge
(
    deltaTable.alias("dados_atuais")
    .merge(
        df_new.alias("novos_dados"),
        "dados_atuais.ID_CLIENTE = novos_dados.ID_CLIENTE"  # Condição de merge
    )
    .whenMatchedUpdateAll()      # Atualiza todos os campos quando encontra correspondência
    .whenNotMatchedInsertAll()   # Insere novo registro quando não encontra correspondência
    .execute()
)
```

!!! tip "Controle Granular"
    Você também pode ter controle mais granular usando:
    ```python
    .whenMatched(condition="novos_dados.LIMITE_CREDITO > 0")
    .updateExpr({"STATUS": "novos_dados.STATUS", "LIMITE_CREDITO": "novos_dados.LIMITE_CREDITO"})
    .whenNotMatched(condition="novos_dados.LIMITE_CREDITO > 1000000")
    .insertExpr({
        "ID_CLIENTE": "novos_dados.ID_CLIENTE",
        "NOME_CLIENTE": "novos_dados.NOME_CLIENTE",
        "UF": "novos_dados.UF",
        "STATUS": "novos_dados.STATUS",
        "LIMITE_CREDITO": "novos_dados.LIMITE_CREDITO"
    })
    ```

## Recursos Avançados

### 1. Time Travel (Viagem no Tempo)

O Delta Lake mantém um histórico de todas as modificações, permitindo acessar e consultar versões anteriores dos dados.

#### Exemplos de Time Travel:

```python
# Listar histórico de versões
deltaTable.history().show()

# Ler versão específica
df_versao_1 = (
    spark
    .read
    .format("delta")
    .option("versionAsOf", 1)
    .load("./RAW/CLIENTES")
)

# Restaurar para versão anterior
(
    spark
    .read
    .format("delta")
    .option("versionAsOf", 1)
    .load("./RAW/CLIENTES")
    .write
    .format("delta")
    .mode("overwrite")
    .save("./RAW/CLIENTES")
)
```

### 2. Otimizações de Dados

O Delta Lake oferece comandos para otimizar o layout dos arquivos e melhorar o desempenho das consultas.

#### Compactação de Arquivos:

```python
# Compactar arquivos pequenos
spark.sql(f"OPTIMIZE delta.`./RAW/CLIENTES`")

# Compactar e ordenar por coluna específica (Z-Order)
spark.sql(f"OPTIMIZE delta.`./RAW/CLIENTES` ZORDER BY (UF, STATUS)")
```

#### Vacuum (Limpeza de Arquivos Antigos):

```python
# Remover arquivos antigos que não são mais necessários para time travel
# Mantendo apenas 7 dias de histórico
deltaTable.vacuum(retentionHours=24*7)
```

!!! warning "Cuidado com Vacuum"
    O comando `vacuum` remove permanentemente arquivos antigos, limitando o time travel. Por padrão, o Delta Lake impede a remoção de arquivos com menos de 7 dias para evitar problemas com consultas ativas.

### 3. Schema Evolution (Evolução de Esquema)

O Delta Lake permite adicionar, remover ou modificar colunas de maneira controlada.

#### Exemplo de Evolução de Esquema:

```python
# Adicionar nova coluna ao esquema
from pyspark.sql.functions import lit

# Ler dados existentes
df_atual = spark.read.format("delta").load("./RAW/CLIENTES")

# Adicionar nova coluna
df_com_nova_coluna = df_atual.withColumn("DATA_ATUALIZACAO", lit("2024-04-23"))

# Escrever de volta com esquema atualizado
(
    df_com_nova_coluna
    .write
    .format("delta")
    .mode("overwrite")
    .option("mergeSchema", "true")  # Habilita evolução de esquema
    .save("./RAW/CLIENTES")
)
```

## Apache Iceberg

O Apache Iceberg é um formato de tabela aberta para conjuntos de dados analíticos de grande escala. Assim como o Delta Lake, o Iceberg oferece transações ACID, controle de versão de dados, e evolução de esquema, mas com algumas diferenças em sua implementação e recursos.

### Visão Geral do Apache Iceberg

O Iceberg foi projetado para resolver os problemas comuns encontrados em data lakes, oferecendo:

- **Transações ACID**: Garantia de atomicidade, consistência, isolamento e durabilidade
- **Schema Evolution**: Suporte para mudanças de esquema sem comprometer dados existentes
- **Time Travel**: Acesso a versões anteriores dos dados (histórico)
- **Particionamento Oculto**: O particionamento é gerenciado pelo Iceberg, sem exigir que os usuários conheçam os detalhes
- **Compactação de Arquivos**: Otimização automática de arquivos pequenos
- **Formatos de Dados Abertos**: Suporte para formatos como Parquet, Avro e ORC

### Configuração do Apache Iceberg com PySpark

Para utilizar o Apache Iceberg com PySpark, é necessário configurar a sessão Spark adequadamente:

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

!!! note "Dependências do Iceberg"
    O Iceberg requer dependências específicas baseadas na versão do Spark. Para o Spark 3.4, utilizamos `iceberg-spark-runtime-3.4_2.12:1.3.0`.

### Operações CRUD com Apache Iceberg

#### 1. Criação de Tabelas Iceberg (Create)

A criação de tabelas Iceberg é semelhante à criação de tabelas Delta, mas com a API específica do Iceberg:

```python
from pyspark.sql.types import StructType, StructField, StringType, FloatType

# Dados de exemplo
data = [
    ("ID001", "CLIENTE_X", "SP", "ATIVO",   250000.00),
    ("ID002", "CLIENTE_Y", "SC", "INATIVO", 400000.00),
    ("ID003", "CLIENTE_Z", "DF", "ATIVO",   1000000.00)
]

# Definição do schema
schema = StructType([
    StructField("ID_CLIENTE", StringType(), True),
    StructField("NOME_CLIENTE", StringType(), True),
    StructField("UF", StringType(), True),
    StructField("STATUS", StringType(), True),
    StructField("LIMITE_CREDITO", FloatType(), True)
])

# Criação do DataFrame
df = spark.createDataFrame(data=data, schema=schema)

# Criar uma tabela Iceberg
df.writeTo("spark_catalog.default.clientes_iceberg").using("iceberg").createOrReplace()
```

#### 2. Inserção de Dados (Insert)

O Iceberg permite adicionar novos registros a uma tabela existente:

```python
# Novos dados para inserção
new_data = [
    ("ID004", "CLIENTE_NEW", "RJ", "ATIVO", 999999.00)
]

# Criar DataFrame com novos dados
df_new = spark.createDataFrame(data=new_data, schema=schema)

# Inserir dados na tabela Iceberg existente
df_new.writeTo("spark_catalog.default.clientes_iceberg").append()
```

#### 3. Atualização de Dados (Update)

O Iceberg suporta atualizações SQL para modificar registros existentes:

```python
# Atualizar registros com SQL
spark.sql("""
    UPDATE spark_catalog.default.clientes_iceberg
    SET STATUS = 'INATIVO', LIMITE_CREDITO = 0.00
    WHERE ID_CLIENTE = 'ID001'
""")
```

#### 4. Exclusão de Dados (Delete)

Exclusões podem ser realizadas com expressões SQL:

```python
# Excluir registros com SQL
spark.sql("""
    DELETE FROM spark_catalog.default.clientes_iceberg
    WHERE LIMITE_CREDITO < 400000.0
""")
```

#### 5. Consulta de Dados (Query)

Consultar dados de tabelas Iceberg é feito com SQL padrão:

```python
# Consultar todos os registros
spark.sql("SELECT * FROM spark_catalog.default.clientes_iceberg").show()
```

### Recursos Avançados do Apache Iceberg

#### 1. Time Travel

O Iceberg mantém o histórico de transações, permitindo acessar versões anteriores dos dados:

```python
# Consultar uma versão específica da tabela por ID de snapshot
spark.read.format("iceberg").option("snapshot-id", "8485094958438453408").load("spark_catalog.default.clientes_iceberg")

# Consultar por timestamp
spark.read.format("iceberg").option("as-of-timestamp", "2023-04-23 10:00:00").load("spark_catalog.default.clientes_iceberg")
```

#### 2. Evolução de Esquema (Schema Evolution)

O Iceberg suporta alterações de esquema como adicionar, renomear, ou remover colunas:

```python
# Adicionar uma nova coluna
spark.sql("""
    ALTER TABLE spark_catalog.default.clientes_iceberg 
    ADD COLUMN data_cadastro TIMESTAMP
""")

# Renomear uma coluna
spark.sql("""
    ALTER TABLE spark_catalog.default.clientes_iceberg 
    RENAME COLUMN UF TO estado
""")
```

#### 3. Otimização e Manutenção

O Iceberg fornece comandos para otimizar e manter tabelas:

```python
# Compactar arquivos pequenos
spark.sql("""
    CALL spark_catalog.system.rewrite_data_files(table => 'default.clientes_iceberg')
""")

# Remover snapshots antigos
spark.sql("""
    CALL spark_catalog.system.expire_snapshots(table => 'default.clientes_iceberg', 
                                             older_than => TIMESTAMP '2023-04-01 00:00:00')
""")
```

### Comparação entre Delta Lake e Apache Iceberg

| Característica | Delta Lake | Apache Iceberg |
| -------------- | ---------- | -------------- |
| Transações ACID | ✓ | ✓ |
| Time Travel | ✓ | ✓ |
| Schema Evolution | ✓ | ✓ |
| Compatibilidade | Integrado ao Databricks | Projeto Apache, ampla adoção |
| Metadados | Parquet + JSON | Formato próprio (avro ou json) |
| Particionamento | Explícito | Oculto / Gerenciado |
| Suporte para nomes de colunas | Case insensitive | Case sensitive |
| Otimização | Z-Order e Optimize | Ordenação e Compactação |

## Integração com SQL

O Delta Lake se integra perfeitamente com SQL, permitindo usar sintaxe familiar para operações:

```python
# Registrar tabela para uso com SQL
spark.sql(f"CREATE TABLE clientes USING DELTA LOCATION './RAW/CLIENTES'")

# Consultar dados com SQL
resultado = spark.sql("SELECT * FROM clientes WHERE UF = 'SP'")
resultado.show()

# Atualizar com SQL
spark.sql("UPDATE clientes SET LIMITE_CREDITO = 300000 WHERE ID_CLIENTE = 'ID001'")

# Merge com SQL
spark.sql("""
MERGE INTO clientes AS target
USING updates AS source
ON target.ID_CLIENTE = source.ID_CLIENTE
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *
""")
```

## Próximos Passos

Agora que você conhece as principais funcionalidades do Delta Lake implementadas neste projeto, explore a seção de [Exemplos](exemplos.md) para ver casos de uso completos e aplicações práticas destas funcionalidades.

