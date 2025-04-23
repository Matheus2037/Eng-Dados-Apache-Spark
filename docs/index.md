# Engenharia de Dados com Apache Spark

Bem-vindo à documentação oficial do projeto de **Engenharia de Dados com Apache Spark**. Este projeto demonstra a implementação de soluções de processamento de dados utilizando tecnologias modernas de big data.

## Visão Geral do Projeto

Este projeto demonstra como utilizar o **Apache Spark** em conjunto com **Delta Lake** e **Apache Iceberg** para criar, manipular e gerenciar dados de forma eficiente, confiável e escalável. A combinação dessas tecnologias proporciona recursos avançados como:

- Transações ACID em dados armazenados em data lakes
- Controle de versão e viagem no tempo (time travel)
- Upserts e merges eficientes
- Evolução de esquema e compactação de arquivos
- Integração perfeita com o ecossistema Spark
- Diferentes abordagens para gerenciamento de tabelas em data lakes

!!! info "Sobre o Delta Lake"
    O [Delta Lake](https://delta.io/) é uma camada de armazenamento de código aberto que traz confiabilidade ao seu data lake. Ele fornece transações ACID, manipulação de metadados escalável e unifica processamento de dados em lote e streaming.

!!! info "Sobre o Apache Iceberg"
    O [Apache Iceberg](https://iceberg.apache.org/) é um formato de tabela aberta para conjuntos de dados analíticos de grande escala. Ele oferece controle de versão, evolução de esquema, operações transacionais ACID e particionamento oculto.
## Por que usar Apache Spark com Delta Lake e Apache Iceberg?

O Apache Spark é uma das ferramentas mais populares para processamento de big data, oferecendo:

- **Velocidade**: Execução rápida de jobs tanto em memória quanto em disco
- **Facilidade de uso**: APIs amigáveis em Python, Java e Scala
- **Generalidade**: Suporte a várias cargas de trabalho como SQL, streaming, machine learning e processamento de grafos
- **Processamento unificado**: Uma única engine para diversos casos de uso

Quando combinado com formatos de tabela modernos como Delta Lake e Apache Iceberg, o Spark ganha recursos adicionais que resolvem muitos desafios comuns em data lakes tradicionais:

### Benefícios do Delta Lake:

- **Confiabilidade**: Transações ACID garantem consistência de dados mesmo em caso de falhas
- **Qualidade de dados**: Restrições de esquema evitam problemas com dados mal-formados
- **Auditoria**: Histórico completo de alterações nos dados
- **Performance**: Otimizações específicas para consultas e ingestões de dados

### Benefícios do Apache Iceberg:

- **Formato aberto**: Projeto Apache de código aberto com ampla adoção na indústria
- **Particionamento oculto**: O particionamento é gerenciado automaticamente pelo Iceberg
- **Evolução de esquema**: Suporte robusto para alterações de esquema sem comprometer dados existentes
- **SQL nativo**: Operações avançadas disponíveis diretamente via SQL
- **Performance**: Otimizações específicas para consultas e ingestões de dados

## Estrutura da Documentação

Esta documentação está organizada nas seguintes seções:

1. [Guia de Instalação](guia-instalacao.md) - Instruções detalhadas para configurar o ambiente
2. [Funcionalidades](funcionalidades.md) - Descrição das principais funcionalidades implementadas
3. [Exemplos](exemplos.md) - Exemplos práticos de uso do projeto
4. [Sobre](sobre.md) - Informações sobre o projeto e seus autores

## Como Começar

Para começar a utilizar o projeto, siga para o [Guia de Instalação](guia-instalacao.md) e configure seu ambiente de desenvolvimento.
