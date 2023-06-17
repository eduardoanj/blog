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

Vamos utilizar um arquivo docker-compose.yaml para executar a imagem Docker do nosso PostgreSQL, optei por utilizar o docker-compose para ficar mais facil adicionar novas imagens futuramente caso os testes precisem de mais cenários.

Primeiramente precisamos do Docker instalado na maquina, maquinas windows podem rodar docker através do app [Docker Desktop](https://www.docker.com/products/docker-desktop/) ou através do wsl2, que pode ser instalado seguindo os passos [deste tutorial](https://docs.docker.com/desktop/windows/wsl/)  **pode ser que seja necessário instalar o plugin docker-compose separadamente.**

Será necessário criar um arquivo docker-compose.yaml que será necessário para executar o nosso PostgreSQL em um container Docker como no exemplo abaixo.

{{< codeblock "docker-compose.yaml" >}}
version: '3.9'
services:
  postgres:
    image: postgres:latest
    volumes:
      - postgres:/data/postgres
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-admin}
      PGDATA: /data/postgres
    restart: unless-stopped
    hostname: postgres
    healthcheck:
      test: exit 0

volumes:
  postgres:
{{< /codeblock >}}

Após o Docker instalado é necessário abrir o terminal do ubuntu e executar o comando:

{{< codeblock "bash" >}}
$ sudo service docker start
{{< /codeblock >}}

Em seguida é necessario executar o nosso docker-compose para fazer o download da imagem mais recente do PostgreSQL e executa-la no container.
{{< codeblock "bash" >}}
$ docker compose up -d
{{< /codeblock >}}

![wsl2](/img/composeUp.jpg)

Agora o nosso PostgreSQL já está sendo executado em um container Docker, é possível ver quais containers estão sendo executados através do comando.

{{< codeblock "bash" >}}
$ docker ps
{{< /codeblock >}}

![wsl2](/img/containersEmExecucao.jpg)

## Iniciando nossa Infraestrutura de testes

Inicialmente precisamos de uma classe que, sempre quando um teste for executado, esta classe suba uma base para testes e de um "TRUNCATE" para limpar a base, este truncate é necessário para que não fiquem registros que possam atrapalhar no resultado do teste. 

{{< codeblock "IntegrationTestBase.cs" "cs" "http://underscorejs.org/#compact" >}}
using System.Diagnostics;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Registration.UserRegistrationEnterpriseExample.Infrastructure.PostgreSql;
using Registration.UserRegistrationEnterpriseExample.Tests.Common;
using Registration.UserRegistrationEnterpriseExample.Tests.TestHelpers;

namespace Registration.UserRegistrationEnterpriseExample.Tests.Application;

public abstract class IntegrationTestBase
{
    private const string DatabaseName = "UserRegistrationEnterpriseExample_IntegrationTests";

    private readonly Lazy<IServiceProvider> _serviceProvider;

    static IntegrationTestBase()
    {
        var serviceCollection = TestServiceCollecionFactory.BuildIntegrationTestInfrastructure(
            DatabaseName,
            () => IntegrationTestClock
        );

        TestDatabaseManager.RebuildDatabase(GetServiceProvider(serviceCollection));
    }

    protected IntegrationTestBase()
    {
        var serviceCollection = TestServiceCollecionFactory.BuildIntegrationTestInfrastructure(
            DatabaseName,
            () => IntegrationTestClock
        );
        _serviceProvider = new Lazy<IServiceProvider>(() => GetServiceProvider(serviceCollection));

        TestDatabaseManager.TruncateAllTables(GetServiceProvider(serviceCollection));
        IntegrationTestClock = TestServiceCollecionFactory.BuildTestClock();
    }
    
    protected static IntegrationTestClock IntegrationTestClock { get; private set; }

    public static IServiceProvider GetServiceProvider(IServiceCollection serviceCollection)
    {
        var defaultServiceProviderFactory = new DefaultServiceProviderFactory(new ServiceProviderOptions());
        return defaultServiceProviderFactory.CreateServiceProvider(serviceCollection);
    }
    
    [DebuggerStepThrough]
    protected Task<TResponse> Handle<TRequest, TResponse>(TRequest request)
        where TRequest : IRequest<TResponse>
    {
        var scope = _serviceProvider.Value.CreateScope();
        var mediator = scope.ServiceProvider.GetService<IMediator>();
        return mediator!.Send(request, CancellationToken.None);
    }
    
    protected DbSet<TEntity> GetTable<TEntity>()
        where TEntity : class
    {
        var scope = _serviceProvider.Value.CreateScope();
        var databaseContext = scope.ServiceProvider.GetService<DatabaseContext>();
        return databaseContext!.Set<TEntity>();
    }

    protected async Task<TEntity> GetByIdAsync<TEntity>(Guid id)
        where TEntity : class
    {
        var scope = _serviceProvider.Value.CreateScope();
        var databaseContext = scope.ServiceProvider.GetService<DatabaseContext>();
        return await databaseContext!.FindAsync<TEntity>(id);
    }
}
{{< /codeblock >}}