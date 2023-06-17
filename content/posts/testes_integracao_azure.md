---
title: "Como criar, configurar e executar testes de integração em .NET utilizando Docker, PostgreSQL e Azure DevOps"
date: 2023-06-17
categories:
- Tutoriais
tags:
- .NET
- Postgresql
- Docker
- Azure
- DevOps
autoThumbnailImage: false
thumbnailImagePosition: "top"
thumbnailImage: https://raw.githubusercontent.com/eduardoanj/blog/main/static/images/CDockerAzure.jpg
coverImage: https://raw.githubusercontent.com/eduardoanj/blog/main/static/images/fotin.jpg
metaAlignment: center
coverMeta: out
---
Criar e executar testes de integração pode ser algo complexo dependendo da infraestrutura do seu projeto, neste tutorial vou mostrar todo o caminho para criar testes de integração úteis e confiáveis desde a sua configuração básica até a sua execução em pipelines Azure.
<!--more-->

## Como funciona a Infraestrutura de testes de integração?

O Primeiro passo para criar testes de integração seria estruturar uma infra de testes, temos dois pontos importantes para pensarmos logo de início. 

1. Precisamos ter uma infraestrutura que crie um banco de dados independente do banco da aplicação.
2. Os dados salvos em um teste não deve influenciar os resultados e os dados salvos em outro teste, portanto a base deve ser limpa sempre que um teste for executado. 

Dados mockados são fortemente reduzidos em testes de integração, as informações do testes serão realmente salvas em nosso banco de teste, os asserts serão com dados retirados de nosso banco que realmente passaram em nosso fluxo de teste, mocks só serão utilizados em casos de dados que podem mudar dependendo da execução, como datas por exemplo. 

## Como executar o banco de dados dos testes de integração?

Vamos utilizar um arquivo docker-compose.yaml para executar a imagem docker do nosso PostgreSQL, optei por utilizar o docker-compose para ficar mais facil adicionar novas imagens futuramente caso os testes precisem de mais cenários, a adição de um Broker por exemplo.

