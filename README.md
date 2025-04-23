# Eng-Dados-Apache-Spark

Este projeto utiliza **Apache Spark** em conjunto com **Delta Lake** para manipulação e gerenciamento de dados. Ele também pode ser estendido para uso com **Apache Iceberg**, embora o foco atual esteja no Delta Lake.

## Requisitos

- Python 3.9.9
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

## Documentação

Este projeto inclui documentação completa em português brasileiro usando MkDocs com o tema Material.

### Instalação das dependências da documentação

O projeto já inclui as dependências necessárias para a documentação no `pyproject.toml`, mas caso precise instalá-las manualmente:

```bash
pip install mkdocs-material mkdocstrings pymdown-extensions
```

### Como acessar a documentação

Para servir a documentação localmente:

```bash
# Instalar dependências se ainda não tiver feito
poetry install

# Servir a documentação localmente
poetry run mkdocs serve
```

Acesse a documentação em seu navegador em `http://127.0.0.1:8000/`.

### Construir documentação para deploy

Para gerar uma versão estática da documentação para publicação:

```bash
poetry run mkdocs build
```

Os arquivos HTML gerados estarão disponíveis na pasta `site/`.

### Recursos da documentação

Nossa documentação utiliza os seguintes recursos:

- **Material for MkDocs**: Tema moderno e responsivo com muitas funcionalidades
- **Plugins**:
  - `search`: Funcionalidade de busca na documentação
  - `mkdocstrings`: Geração de documentação a partir de docstrings do código
- **Extensões Markdown**:
  - `admonition`: Blocos de alertas e notas destacadas
  - `pymdownx.details`: Componentes expansíveis
  - `pymdownx.superfences`: Suporte aprimorado para blocos de código
  - `pymdownx.tabbed`: Abas para alternar entre diferentes conteúdos
  - `pymdownx.highlight`: Realce de sintaxe para códigos

### Estrutura da documentação

A documentação está organizada nas seguintes seções:

- **Página Inicial**: Visão geral do projeto
- **Guia de Instalação**: Instruções detalhadas para configuração do ambiente
- **Funcionalidades**: Descrição completa das capacidades do Delta Lake
- **Exemplos**: Casos de uso práticos e exemplos de código
- **Sobre**: Informações sobre o projeto e seus autores

### Personalização da documentação

Para personalizar a documentação, edite o arquivo `mkdocs.yml` na raiz do projeto. Você pode modificar:

- **Tema e cores**: Altere as configurações de `theme` para modificar a aparência
- **Navegação**: Atualize a seção `nav` para alterar a estrutura do menu
- **Extensões**: Adicione ou remova extensões conforme necessário

# Autores
 - Matheus da Silva Gastaldi matheusdasilvagastaldi@gmail.com

 - Gabriel Morona Coelho gabrielmorona0229@gmail.com

 - João Carlos Rodrigues Martins joaocarlosrm2004@gmail.com
