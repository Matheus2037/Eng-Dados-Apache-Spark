# Eng-Dados-Apache-Spark

Este projeto utiliza **Apache Spark** em conjunto com **Delta Lake** para manipulação e gerenciamento de dados. Ele também pode ser estendido para uso com **Apache Iceberg**, embora o foco atual esteja no Delta Lake.

## Requisitos

- Python 3.10 ou superior
- Dependências especificadas no arquivo `pyproject.toml`:
  - `pyspark==3.4.2`
  - `delta-spark==2.4.0`
  - `jupyterlab>=4.4.0,<5.0.0`


## Funcionalidades

### 1. Criação de Tabelas Delta
O notebook [`delta.ipynb`](pyspark-delta/delta.ipynb) demonstra como criar tabelas Delta a partir de um DataFrame Spark.

### 2. Operações de Inserção e Atualização
O projeto inclui exemplos de como realizar operações de `MERGE` para inserir ou atualizar dados em tabelas Delta.

### 3. Exclusão de Dados
Também é possível realizar exclusões condicionais em tabelas Delta.

## Como Executar

Configure o Java 11 no Ubuntu (WSL)

Para garantir que aplicações baseadas em Spark ou outras ferramentas que dependem do Java funcionem corretamente, siga os passos abaixo para instalar e configurar o Java 11 no seu ambiente Ubuntu/WSL:

### 1. Instalação do Java 11

Execute o seguinte comando no terminal:

    ```bash
    sudo apt install openjdk-11-jdk -y

### 2. Configuração das variáveis de ambiente

Adicione as variáveis `JAVA_HOME` e `PATH` ao final do seu arquivo `~/.bashrc`:

    ```bash
    echo 'export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"' >> ~/.bashrc
    echo 'export PATH="$JAVA_HOME/bin:$PATH"' >> ~/.bashrc

### 3. Reinicie o seu terminal

Caso seja o WSL use o comando:

    ```bash
      source ~/.bashrc

1. Certifique-se de que todas as dependências estão instaladas. Você pode usar o Poetry para gerenciar as dependências:
   ```bash 
   poetry install
2. Inicie o JupyterLab para executar o notebook:

3. Abra o arquivo `delta.ipynb` e execute as células para testar as funcionalidades.

# Autores
 - Matheus da Silva Gastaldi matheusdasilvagastaldi@gmail.com

 - Gabriel Morona Coelho gabrielmorona0229@gmail.com

 - João Carlos Rodrigues Martins joaocarlosrm2004@gmail.com
