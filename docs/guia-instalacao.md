# Guia de Instalação

Este guia fornece instruções detalhadas para configurar seu ambiente de desenvolvimento e instalar todas as dependências necessárias para trabalhar com o projeto de Engenharia de Dados com Apache Spark.

## Requisitos do Sistema

Antes de iniciar a instalação, certifique-se de que seu sistema atende aos seguintes requisitos:

### Requisitos de Hardware
- **Processador**: Mínimo de 2 cores (recomendado 4+ cores)
- **Memória RAM**: Mínimo de 4GB (recomendado 8GB+ para cargas de trabalho maiores)
- **Espaço em Disco**: Mínimo de 2GB disponíveis

### Pré-requisitos de Software
- **Sistema Operacional**: Windows 10/11, macOS (10.15+), ou Linux (Ubuntu 20.04+, CentOS 7+)
- **Python**: Versão 3.9.9
- **Java**: JDK 8 ou superior (necessário para Apache Spark)
- **Poetry**: Ferramenta de gerenciamento de dependências para Python

!!! warning "Importante sobre Java"
    O Apache Spark requer Java para funcionar. Certifique-se de ter o JDK instalado e a variável de ambiente `JAVA_HOME` configurada corretamente.

## Instalação do Python

Se você ainda não tem Python 3.9.9 instalado:

### Para Windows
1. Baixe o instalador do Python em [python.org](https://www.python.org/downloads/)
2. Execute o instalador e marque a opção "Add Python to PATH"
3. Conclua a instalação seguindo as instruções na tela

### Para macOS
Utilizando o Homebrew:
```bash
brew install python@3.9
```

### Para Linux (Ubuntu/Debian)
```bash
sudo apt update
sudo apt install python3.9 python3.9-venv python3.9-dev
```

## Instalação do Poetry

O Poetry é a ferramenta recomendada para gerenciar dependências e ambientes virtuais neste projeto.

### Para Windows, macOS e Linux
```bash
curl -sSL https://install.python-poetry.org | python3 -
```

Após a instalação, adicione o Poetry ao seu PATH seguindo as instruções exibidas no final da instalação.

Para verificar a instalação, execute:
```bash
poetry --version
```

## Clonando o Repositório

Clone o repositório do projeto utilizando Git:

```bash
git clone https://github.com/gabrielcoelho/Eng-Dados-Apache-Spark.git
cd Eng-Dados-Apache-Spark
```

## Instalando Dependências com Poetry

Uma vez dentro do diretório do projeto, instale todas as dependências:

```bash
poetry install
```

Este comando criará um ambiente virtual e instalará as seguintes dependências principais:
- **pyspark (3.4.2)**: Motor de processamento distribuído
- **delta-spark (2.4.0)**: Extensão do Spark para Delta Lake
- **iceberg-spark-runtime-3.4_2.12 (1.3.0)**: Extensão do Spark para Apache Iceberg
- **jupyterlab (4.4.0+)**: Ambiente interativo para execução de notebooks

## Ativando o Ambiente Virtual

Para ativar o ambiente virtual criado pelo Poetry:

```bash
poetry shell
```

Ou você pode executar comandos diretamente dentro do ambiente virtual sem ativá-lo permanentemente:

```bash
poetry run jupyter lab
```

## Verificando a Instalação

Para verificar se tudo foi instalado corretamente:

1. Ative o ambiente virtual: `poetry shell`
2. Abra o Python REPL: `python`
3. Tente importar as bibliotecas principais:

```python
import pyspark
from delta import *
import importlib

print(f"PySpark versão: {pyspark.__version__}")
print("Delta Lake disponível!")

# Verificar disponibilidade do Apache Iceberg
iceberg_module = importlib.util.find_spec("pyiceberg")
if iceberg_module is not None:
    print("Apache Iceberg disponível!")
else:
    print("Apache Iceberg não encontrado. Verifique a instalação.")
```

Se não ocorrer nenhum erro, sua instalação está funcionando corretamente.

## Iniciando o JupyterLab

Para iniciar o JupyterLab e acessar os notebooks do projeto:

```bash
poetry run jupyter lab
```

Isso abrirá o JupyterLab no seu navegador padrão. Navegue até a pasta `pyspark-delta` para acessar os notebooks de exemplo.

## Troubleshooting

### Problema: Erro ao instalar o PySpark

**Sintoma**: Erros relacionados a dependências do Java durante a instalação do PySpark.

**Solução**: Certifique-se de que o Java (JDK 8+) está instalado e configurado:
```bash
# Verifique a instalação do Java
java -version

# Configure a variável JAVA_HOME se necessário
export JAVA_HOME=/caminho/para/seu/jdk
```

### Problema: Dependências não encontradas após instalação

**Sintoma**: Módulos não encontrados mesmo após instalação com Poetry.

**Solução**: Certifique-se de que está usando o ambiente virtual correto:
```bash
# Verifique se o ambiente virtual está ativo
poetry env info

# Ative o ambiente se necessário
poetry shell
```

### Problema: O JupyterLab não consegue encontrar dependências

**Sintoma**: Erros de importação ao executar células no JupyterLab.

**Solução**: Certifique-se de que o Jupyter está usando o kernel correto:
1. No JupyterLab, verifique no canto superior direito qual kernel está sendo usado
2. Selecione o kernel correspondente ao seu ambiente Poetry
3. Se o kernel não estiver disponível, instale o ipykernel no ambiente:
   ```bash
   poetry run pip install ipykernel
   poetry run python -m ipykernel install --user --name=eng-dados-spark
   ```

### Problema: Erros relacionados ao Delta Lake

**Sintoma**: `ClassNotFoundException` ou erros similares ao tentar usar funcionalidades do Delta Lake.

**Solução**: Verifique se as dependências do Delta Lake estão corretamente configuradas:
```python
from pyspark.sql import SparkSession

# Configure a sessão Spark com suporte ao Delta Lake
spark = SparkSession.builder \
    .appName("DeltaLakeTest") \
    .config("spark.jars.packages", "io.delta:delta-core_2.12:2.4.0") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()
```

### Problema: Erros relacionados ao Apache Iceberg

**Sintoma**: `ClassNotFoundException`, `NoClassDefFoundError` ou erros similares ao tentar usar funcionalidades do Iceberg.

**Solução**: Verifique se as dependências do Iceberg estão corretamente configuradas:
```python
from pyspark.sql import SparkSession

# Configure a sessão Spark com suporte ao Apache Iceberg
spark = SparkSession.builder \
    .appName("IcebergTest") \
    .config("spark.jars.packages", "org.apache.iceberg:iceberg-spark-runtime-3.4_2.12:1.3.0") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.iceberg.spark.SparkSessionCatalog") \
    .config("spark.sql.catalog.local", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.local.type", "hadoop") \
    .config("spark.sql.catalog.local.warehouse", "warehouse_path") \
    .getOrCreate()
```

### Problema: Tabelas Iceberg não são reconhecidas

**Sintoma**: Erros do tipo `Table not found` ou tabelas Iceberg não podem ser lidas/escritas.

**Solução**: Certifique-se de que os catálogos do Iceberg estão configurados corretamente:
```python
# Verifique se o catálogo está configurado
spark.sql("SHOW NAMESPACES").show()

# Crie um namespace se necessário
spark.sql("CREATE NAMESPACE IF NOT EXISTS local.db")

# Certifique-se de usar o caminho correto para as tabelas
spark.sql("CREATE TABLE local.db.example (id INT, data STRING) USING iceberg")
```

## Próximos Passos

Após completar a instalação, você está pronto para explorar as [funcionalidades](funcionalidades.md) do projeto e executar os [exemplos](exemplos.md) disponíveis.
