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
Criar e executar testes de integração pode ser algo complexo dependendo da infraestrutura do seu projeto. Neste tutorial vou mostrar todo o caminho para criar testes de integração úteis e confiáveis desde a sua configuração básica até a sua execução em pipelines Azure.
<!--more-->

## Como funciona a Infraestrutura de testes de integração?

O Primeiro passo para criar testes de integração seria estruturar uma infra de testes, temos dois pontos importantes para pensarmos logo de início. 

1. Precisamos ter uma infraestrutura que crie um banco de dados independente do banco da aplicação.
2. Os dados salvos em um teste não deve influenciar os resultados e os dados salvos em outro teste, portanto a base deve ser limpa sempre que um teste for executado. 

Dados mockados são fortemente reduzidos em testes de integração, as informações do testes serão realmente salvas em nosso banco de teste. Os asserts serão com dados retirados de nosso banco que realmente passaram em nosso fluxo de teste. Mocks só serão utilizados em casos de dados que podem mudar dependendo da execução, como datas por exemplo. 

## Como executar o banco de dados dos testes de integração?

Vamos utilizar um arquivo docker-compose.yaml para executar a imagem Docker do nosso PostgreSQL. Optei por utilizar o docker-compose para ficar mais facil adicionar novas imagens futuramente caso os testes precisem de mais cenários.

Primeiramente precisamos do Docker instalado na maquina, maquinas windows podem rodar docker através do app [Docker Desktop](https://www.docker.com/products/docker-desktop/) ou através do wsl2, que pode ser instalado seguindo os passos [deste tutorial](https://docs.docker.com/desktop/windows/wsl/)  **pode ser que seja necessário instalar o plugin docker-compose separadamente.**

Será necessário criar um arquivo docker-compose.yaml para executar o nosso PostgreSQL em um container Docker como no exemplo abaixo.

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

Após o Docker instalado é necessário abrir o terminal do ubuntu (wsl2) e executar o comando:

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

## Arquitetura

Em nosso projeto eu utilizei arquitetura limpa (Clean Architecture) com DDD para ter uma separação de camadas e responsabilidades, nosso **Domain** é onde estão localizadas nossas regras de negocio e tudo que se refere ao domínio como Entidades etc, nosso **Application** é onde se refere à aplicação e é a camada que tem acesso à infra, a parte da **Infrastructure** é onde deixamos códigos relacionados à infra, como comunicação com banco de dados e brokers, e por fim a **Presentation**, que é onde estão nossos controllers e tudo o que tem de relacionado à comunicação externa, como segue nas imagens a seguir.

Fluxo projetos:
![CleanArchitectureFluxo](/img/CleanArchitectureFluxo.png)

Projetos:
![CleanArchitecture](/img/CleanArchitecture.jpg)

## Cenário de teste

Nosso cenário de teste consiste em salvar uma entidade USER no banco e verificar se realmente foi salva.
{{< codeblock "User.cs" "cs" "http://underscorejs.org/#compact" >}}
namespace Registration.UserRegistrationEnterpriseExample.Domain.Entidades;

public class User 
{
    public User(int userNumber)
    {
        UserNumber = userNumber;
    }
    
    public int UserNumber { get; set; }
    public string Document { get; set; }
}
{{< /codeblock >}}

Nossa request veio do Presentation para o Application utiliza-la, salva-la como a entidade User no banco através do método handle.
{{< codeblock "SaveUserRequest.cs" "cs" "http://underscorejs.org/#compact" >}}
using MediatR;

namespace Registration.UserRegistrationEnterpriseExample.Application.Users.PersistirUser;

public class SaveUserRequest : IRequest<Guid>
{
    public DateTime TimestampUtc { get; set; }
    public int UserNumber { get; set; }
    public string Document { get; set; }
}
{{< /codeblock >}}

{{< codeblock "SaveUserRequestHandler.cs" "cs" "http://underscorejs.org/#compact" >}}
using MediatR;
using Registration.UserRegistrationEnterpriseExample.Application.Common.Exceptions;
using Registration.UserRegistrationEnterpriseExample.Application.Common.Extensions;
using Registration.UserRegistrationEnterpriseExample.Application.Common.Interfaces;
using Registration.UserRegistrationEnterpriseExample.Domain.Entidades;

namespace Registration.UserRegistrationEnterpriseExample.Application.Users.PersistirUser;

internal class SaveUserRequestHandler : IRequestHandler<SaveUserRequest, Guid>
{
    private readonly IUsers _users;

    public SaveUserRequestHandler(IUsers users)
    {
        _users = users;
    }

    public async Task<Guid> Handle(SaveUserRequest request, CancellationToken cancellationToken)
    {
        var user = await _users.SelecionarPorChave(request.UserNumber);

        if (user == null)
            user = new User(request.UserNumber);
        else if (request.TimestampUtc.IsOlderThan(user.OriginTimestampUtc))
            throw new OldEventException("UserRecebido", request.UserNumber);
        
        user.OriginTimestampUtc = request.TimestampUtc;
        user.Document = request.Document;

        await _users.InserirOuAtualizar(user);
        return user.Id;
    }
}
{{< /codeblock >}}

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
        var serviceCollection = TestIntegrationServiceCollectionFactory.BuildIntegrationTestInfrastructure(
            DatabaseName,
            () => IntegrationTestClock
        );

        TestIntegrationDatabaseManager.RebuildIntegrationDatabase(GetServiceProvider(serviceCollection));
    }

    protected IntegrationTestBase()
    {
        var serviceCollection = TestIntegrationServiceCollectionFactory.BuildIntegrationTestInfrastructure(
            DatabaseName,
            () => IntegrationTestClock
        );
        _serviceProvider = new Lazy<IServiceProvider>(() => GetServiceProvider(serviceCollection));

        TestIntegrationDatabaseManager.TruncateAllDatabaseTables(GetServiceProvider(serviceCollection));
        IntegrationTestClock = TestIntegrationServiceCollectionFactory.BuildIntegrationTestClock();
    }
    
    protected static IntegrationTestClock IntegrationTestClock { get; private set; }

    // busca o ServiceProvider (Necessário para ter acesso a instâncias específicas injetadas no código) 
    public static IServiceProvider GetServiceProvider(IServiceCollection serviceCollection)
    {
        var defaultServiceProviderFactory = new DefaultServiceProviderFactory(new ServiceProviderOptions());
        return defaultServiceProviderFactory.CreateServiceProvider(serviceCollection);
    }
    
    // Os métodos Handle, GetTable e GetByIdAsync são helpers de teste para fazer execuções ou insersões na database que auxiliem
    // por exemplo: caso meu teste salve algum dado no banco e eu precise verificar se realmente foi salvo, eu posso utilizar o GetByIdAsync para buscar a
    // entidade do banco pelo id e fazer os asserts
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

A classe TestIntegrationServiceCollectionFactory é a responsável por criar uma lógica de injeção de dependência para testes, ela é responsável tambem por criar um IntegrationTestClock para substituir o Datetime.now() da aplicação para ser sempre um DateTime estático. O IntegrationTestClock é extremamente importante para assegurar a acertividade dos testes, quando falamos de testes de integração o tempo é algo que nunca deve mudar, o nosso IntegrationTestClock basicamente "Mocka" a nossa interface de tempo IClock e faz com que datas utilizadas no fluxo fiquem imutáveis.

{{< codeblock "TestIntegrationServiceCollectionFactory.cs" "cs" "http://underscorejs.org/#compact" >}}
using System.Globalization;
using FluentAssertions.Extensions;
using Microsoft.Extensions.DependencyInjection;
using Registration.UserRegistrationEnterpriseExample.Application;
using Registration.UserRegistrationEnterpriseExample.Domain;
using Registration.UserRegistrationEnterpriseExample.Infrastructure;
using Registration.UserRegistrationEnterpriseExample.Tests.TestHelpers;

namespace Registration.UserRegistrationEnterpriseExample.Tests.Common;

public static class TestIntegrationServiceCollectionFactory
{
    public static IServiceCollection BuildIntegrationTestInfrastructure(string testDatabaseName,
        Func<IntegrationTestClock> getIntegrationTestClock)
    {
        var services = new ServiceCollection();
        services.AddIntegrationTestDatabase(testDatabaseName);
        services.AddIntegrationTestLogs();
        services.AddDomain();
        services.AddApplication();
        services.AddInfrastructure();
        services.AddIntegrationTestClock(getIntegrationTestClock);

        return services;
    }

    public static IntegrationTestClock BuildIntegrationTestClock()
    {
        return new IntegrationTestClock(3.September(2019).At(10.Hours(21.Minutes(45.Seconds()))));
    }
}
{{< /codeblock >}}

{{< codeblock "IntegrationTestClock.cs" "cs" "http://underscorejs.org/#compact" >}}
using NSubstitute;
using Registration.UserRegistrationEnterpriseExample.Domain.Common;

namespace Registration.UserRegistrationEnterpriseExample.Tests.TestHelpers;

public class IntegrationTestClock : IClock
{
    private readonly IClock _mockedClock;

    public IntegrationTestClock(DateTime seedDateTime)
    {
        _mockedClock = Substitute.For<IClock>();
        SetIntegrationClock(seedDateTime);
    }

    public DateTime Now => _mockedClock.Now;

    public void AdvanceBy(TimeSpan timeSpan)
    {
        SetIntegrationClock(_mockedClock.Now + timeSpan);
    }

    private void SetIntegrationClock(DateTime dateTime)
    {
        _mockedClock.Now.Returns(dateTime);
    }
}
{{< /codeblock >}}

Interface a ser "Mockada" pelo IntegrationTestClock
{{< codeblock "IClock.cs" "cs" "http://underscorejs.org/#compact" >}}
namespace Registration.UserRegistrationEnterpriseExample.Domain.Common;

public interface IClock
{
    DateTime Now { get; }
}
{{< /codeblock >}}

A Classe SystemClock é usada em todo o código para atribuir datas, caso não seja substituido no teste, o teste que faz asserts com datas falhará a cada nova execução (pois sempre terá um datetime diferente). Por conta disso o nosso IntegrationTestClock é tão importante, pois faz com que toda a execução tenha o mesmo datetime.
{{< codeblock "SystemClock.cs" "cs" "http://underscorejs.org/#compact" >}}
using Registration.UserRegistrationEnterpriseExample.Domain.Common;
using TimeZoneConverter;

namespace Registration.UserRegistrationEnterpriseExample.Infrastructure.DateAndTime;

public class SystemClock : IClock
{
    public DateTime Now => DateTime.UtcNow;
    public DateTime Today => DateTime.UtcNow.Date;

    public DateTime NowTZ(TimeZoneInfo timeZoneInfo)
    {
        return TimeZoneInfo.ConvertTimeFromUtc(Now, timeZoneInfo);
    }

    public DateTime NowTZ(string timeZone)
    {
        var timeZoneInfo = TZConvert.GetTimeZoneInfo(timeZone);
        return NowTZ(timeZoneInfo);
    }
}
{{< /codeblock >}}

A classe TestIntegrationDatabaseManager é a grande responsável por fazer o "ReBuild" da database de testes e realizar o Truncate das tabelas a cada teste realizado, O Truncate é o responsável por limpar a base antes da execução do teste de integração.

{{< codeblock "TestIntegrationServiceCollecionFactory.cs" "cs" "http://underscorejs.org/#compact" >}}
using System.Collections;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Registration.UserRegistrationEnterpriseExample.Infrastructure.PostgreSql;
using Registration.UserRegistrationEnterpriseExample.Infrastructure.PostgreSql.Common;
using Registration.UserRegistrationEnterpriseExample.Infrastructure.PostgreSql.Mappings;

namespace Registration.UserRegistrationEnterpriseExample.Tests;

public static class TestIntegrationDatabaseManager
{
    public static void RebuildIntegrationDatabase(IServiceProvider serviceProvider)
    {
        var databaseContext = serviceProvider.GetService<DatabaseContext>();
        databaseContext!.Database.EnsureDeleted();
        databaseContext.Database.Migrate();
    }

    public static void TruncateAllDatabaseTables(IServiceProvider serviceProvider)
    {
        var databaseContext = serviceProvider.GetService<DatabaseContext>();

        var tableNames = GetTableNames();
        foreach (var tableName in tableNames)
            databaseContext!.Database.ExecuteSqlRaw($"TRUNCATE TABLE {tableName} CASCADE");
    }

    private static IEnumerable GetTableNames()
    {
        var tableNames = typeof(UserMapping).Assembly
            .GetTypes()
            .Where(x => typeof(IBaseMapping).IsAssignableFrom(x))
            .Where(x => x.IsAbstract is false)
            .Select(x => (IBaseMapping) Activator.CreateInstance(x))
            .Select(x => x!.TableName)
            .ToList();
        return tableNames;
    }
}
{{< /codeblock >}}

## Nossos testes de integração

Nossos testes de integração possuem 3 cenários, o primeiro deve inserir um User no banco, o segundo deve atualizar um usuário já inserido (um evento de atualização), e o terceiro deve lançar erro caso o usuário for um usuário antigo (sem ser um evento de atualização), os testes usam o metodo Handle genérico fornecido pela IntegrationTestBase.cs para executar a request, o decorator **Collection** foi atribuido para o parâmetro **Sequential** para executar as classes de teste sequancialmente.
{{< codeblock "SaveUserRequestTests.cs" "cs" "http://underscorejs.org/#compact" >}}
using FluentAssertions;
using FluentAssertions.Extensions;
using Microsoft.EntityFrameworkCore;
using Registration.UserRegistrationEnterpriseExample.Application.Users.PersistirUser;
using Registration.UserRegistrationEnterpriseExample.Application.Common.Exceptions;
using Registration.UserRegistrationEnterpriseExample.Domain.Entidades;
using Registration.UserRegistrationEnterpriseExample.Tests.Builders.Application.Users.PersistirUser;
using Xunit;

namespace Registration.UserRegistrationEnterpriseExample.Tests.Application.Users.PersistirUser;

[Collection("Sequential")]
public class SaveUserRequestTests : IntegrationTestBase
{
    private readonly SaveUserRequest _userRequest;

    public SaveUserRequestTests()
    {
        _userRequest = new SaveUserRequestBuilder()
            .WithUserNumber(11111)
            .WithDocument("6768757576")
            .Build();
    }

    [Fact]
    public async void It_should_insert_a_new_user()
    {
        var id = await Handle<SaveUserRequest, Guid>(_userRequest);

        id.Should().NotBeEmpty();
        var user = await GetByIdAsync<User>(id);
        user.UserNumber.Should().Be(_userRequest.UserNumber);
        user.OriginTimestampUtc.Should().Be(_userRequest.TimestampUtc);
        user.Document.Should().Be(_userRequest.Document);
    }
    
    [Fact]
    public async void It_should_update_a_user()
    {
        var id = await Handle<SaveUserRequest, Guid>(_userRequest);
        IntegrationTestClock.AdvanceBy(1.Minutes());
        _userRequest.TimestampUtc = IntegrationTestClock.Now;
        _userRequest.Document = "487557418666";

        var secondId = await Handle<SaveUserRequest, Guid>(_userRequest);

        var usersDbSet = GetTable<User>();
        var users = await usersDbSet.ToListAsync();
        users.Should().HaveCount(1);
        secondId.Should().Be(id);
        var user = await GetByIdAsync<User>(id);
        user.OriginTimestampUtc.Should().Be(_userRequest.TimestampUtc);
        user.Document.Should().Be("487557418666");
    }

    [Fact]
    public async void It_should_ignore_a_user_with_an_old_timestamp()
    {
        await Handle<SaveUserRequest, Guid>(_userRequest);
        _userRequest.TimestampUtc -= 1.Minutes();
        IntegrationTestClock.AdvanceBy(-1.Minutes());

        var handleCommand = async () => { await Handle<SaveUserRequest, Guid>(_userRequest); };
        
        await handleCommand.Should().ThrowAsync<OldEventException>();
    }
}
{{< /codeblock >}}

Com isso nossos testes de integração passarão com sucesso utilizando o nosso banco de testes. 

![tests](/img/tests.jpg)

## Como rodar os testes de integração na pipeline CI do Azure DevOps com o nosso banco de testes?

Inicialmente, para a execução dos testes na pipeline azure precisamos de um arquivo .yaml que terá os passos para a execução do teste pelo agente da pipeline azure,
por questão de organização decidi separar esses passos em dois arquivos .yaml, o azure-pipelines.yaml, que terá os triggers e configurações iniciais e o backend-build-and-test-stage.yaml que terá a execução dos testes em sí.

{{< codeblock "azure-pipelines.yaml" >}}
name: "$(Rev:r)"

variables:
  - name: "MinhaVariavel"               ## Variaveis necessárias para o teste
    value: "valorDaMinhaVariavel"

trigger:                                ## O "Gatilho" com configurações de qual branchs especificas a pipeline deve ser executada
  batch: true
  branches:
    include:
      - "main"
      - "release"
      - "develop"
      - "feature/*"
      - "hotfix/*"
      - "fix/*"
  paths:
    include:
      - "$(WorkingDirectory)"           
    exclude:                                ## Extensões de arquivos que devem ser ignorados no projeto
      - "$(WorkingDirectory)/**/*.js"       
      - "$(WorkingDirectory)/**/*.md"
      - "$(WorkingDirectory)/**/*.py"

stages:                          ## Configuração dos stages que são basicamente "passos" a serem executados pela pipeline (pode ser incluídos outros tipos de testes por exemplo)
  - stage: "BackendBuildAndTest"
    displayName: "Backend build and test"
    dependsOn: []                 ## Adicionar dependências (por exemplo, o stage de testeBackend só pode ser executado quando o stage de testeFrontend estiver sido finalizado)
    jobs:
      - job: "BackendBuildAndTestJob"       ## nome do job específico
        pool:
          vmImage: "ubuntu-20.04"           ## Versão da vm linux que o teste será executado
          demands:
            - "vstest"
          timeoutInMinutes: 10
        steps:
          - template: "backend-build-and-test-stage.yaml"               ## Introduzido o caminho para o nosso segundo arquivo onde os testes serão realizados 
            parameters:                                                 ## Variáveis de ambiente "encaminhadas" para serem usadas no nosso template de testes
              BuildNumber: "$(Build.BuildNumber)"
              SolutionName: "$(SolutionName)"
              WorkingDirectory: "$(WorkingDirectory)/backend"
              ProjectPrefix: "$(ProjectPrefix)"
              MinhaVariavel: "$(MinhaVariavel)"
{{< /codeblock >}}

Após nosso arquivo azure-pipelines.yaml finalizado, deveremos implementar nosso template, onde os testes de integração de fato serão executados.

{{< codeblock "backend-build-and-test-stage.yaml" >}}
parameters:                                                 ## Variáveis utilizadas nos testes, (recebidas do azure-pipelines.yaml ou novas necessárias para a execução dos testes)
  BuildConfiguration: 'Debug'
  BuildNumber: ''
  SolutionName: ''
  WorkingDirectory: ''
  ProjectPrefix: ''
  MinhaVariavel: ''

steps:
  - task: UseDotNet@2                                      ## Utilizada para definir a versão do SDK 
    displayName: 'Use .NET sdk'
    version: '6.x'                                         

  - task: DotNetCoreCLI@2                                   ## Comado restore
    displayName: 'dotnet restore'
    inputs:
      command: 'restore'
      projects: '${{ parameters.WorkingDirectory }}/**/*.csproj'
      feedsToUse: 'config'
      nugetConfigPath: '${{ parameters.WorkingDirectory }}/NuGet.Config'

  - task: DotNetCoreCLI@2                                   ## Utilizado para executar o comando de build
    displayName: 'dotnet build'
    inputs:
      command: 'build'
      projects: '${{ parameters.WorkingDirectory }}/**/*.csproj'
      arguments: '--configuration ${{ parameters.BuildConfiguration }} --no-restore'

  - task: DockerCompose@0                                   ## Utilizado para executar o container do nosso postgres na pipeline
    displayName: 'Build and run PostgreSql container'
    inputs:
      containerregistrytype: 'Container Registry'
      dockerComposeFile: '${{ parameters.WorkingDirectory }}/devops/docker/docker-compose.yaml'
      dockerComposeCommand: 'up --detach'

  - script: |                                               ## comandos para a execução dos testes de integração e criação do xml de coverage (necessário para o Sonar)
      dotnet test ${{ parameters.WorkingDirectory }}/${{ parameters.SolutionName }}.sln --configuration ${{ parameters.BuildConfiguration }} --no-build --no-restore --logger trx /p:CollectCoverage=true /p:CoverletOutputFormat=opencover --collect:"XPlat Code Coverage"

      mkdir ${{ parameters.WorkingDirectory }}/results

      cp -f ${{ parameters.WorkingDirectory }}/tests/${{ parameters.ProjectPrefix }}.Tests/coverage.opencover.xml ${{ parameters.WorkingDirectory }}/results/coverage-integration-tests-opencover.xml
      
      dotnet tool install dotnet-reportgenerator-globaltool --tool-path .

      ./reportgenerator "-reports:${{ parameters.WorkingDirectory }}/results/coverage-integration-tests-opencover.xml;${{ parameters.WorkingDirectory }}/results/coverage-unit-tests-opencover.xml" "-targetdir:${{ parameters.WorkingDirectory }}/results" "-reporttypes:Cobertura"
    displayName: 'dotnet test'
    failOnStderr: true
{{< /codeblock >}}

Logo após a implementação dos yamls de teste será possivel executar os testes na pipeline azure, primeiramente será necessário criar uma nova pipeline na Azure.

![AzureCreate](/img/AzureCreatePipeline1.jpg)

Adicionar a Origem de seu repositório.

![AzureCreate](/img/AzureCreatePipeline2.jpg)

E configurar sua pipeline com o path da localização do seu yaml

![AzureCreate](/img/AzureCreatePipeline3.jpg)

Path.
![AzureCreate](/img/AzureCreatePipeline4.jpg)

E finalmente executar a sua pipeline com os testes de integração
![AzureTests](/img/AzureTests.jpeg)