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

