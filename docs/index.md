# Engenharia de Dados com Apache Spark

Bem-vindo à documentação oficial do projeto de **Engenharia de Dados com Apache Spark**. Este projeto demonstra a implementação de soluções de processamento de dados utilizando tecnologias modernas de big data.

## Visão Geral do Projeto

Este projeto demonstra como utilizar o **Apache Spark** em conjunto com **Delta Lake** para criar, manipular e gerenciar dados de forma eficiente, confiável e escalável. A combinação dessas tecnologias proporciona recursos avançados como:

- Transações ACID em dados armazenados em data lakes
- Controle de versão e viagem no tempo (time travel)
- Upserts e merges eficientes
- Otimização de esquema e compactação de arquivos
- Integração perfeita com o ecossistema Spark

!!! info "Sobre o Delta Lake"
    O [Delta Lake](https://delta.io/) é uma camada de armazenamento de código aberto que traz confiabilidade ao seu data lake. Ele fornece transações ACID, manipulação de metadados escalável e unifica processamento de dados em lote e streaming.

## Por que usar Apache Spark com Delta Lake?

O Apache Spark é uma das ferramentas mais populares para processamento de big data, oferecendo:

- **Velocidade**: Execução rápida de jobs tanto em memória quanto em disco
- **Facilidade de uso**: APIs amigáveis em Python, Java e Scala
- **Generalidade**: Suporte a várias cargas de trabalho como SQL, streaming, machine learning e processamento de grafos
- **Processamento unificado**: Uma única engine para diversos casos de uso

Quando combinado com o Delta Lake, o Spark ganha recursos adicionais que resolvem muitos desafios comuns em data lakes tradicionais:

- **Confiabilidade**: Transações ACID garantem consistência de dados mesmo em caso de falhas
- **Qualidade de dados**: Restrições de esquema evitam problemas com dados mal-formados
- **Auditoria**: Histórico completo de alterações nos dados
- **Performance**: Otimizações específicas para consultas e ingestões de dados

## Estrutura da Documentação

Esta documentação está organizada nas seguintes seções:

1. [Guia de Instalação](guia-instalacao.md) - Instruções detalhadas para configurar o ambiente
2. [Funcionalidades](funcionalidades.md) - Descrição das principais funcionalidades implementadas
3. [Exemplos](exemplos.md) - Exemplos práticos de uso do projeto
4. [Sobre](sobre.md) - Informações sobre o projeto e seus autores

## Como Começar

Para começar a utilizar o projeto, siga para o [Guia de Instalação](guia-instalacao.md) e configure seu ambiente de desenvolvimento.
