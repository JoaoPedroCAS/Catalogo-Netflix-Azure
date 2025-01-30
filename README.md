# API DioFLIX

Este projeto demonstra o uso de microserviços utilizando os recursos do Microsoft Azure.

## Recursos Utilizados
- **Grupo de Recursos**
- **API Management**
- **Cosmos DB**
- **Storage Account**
- **Azure Function Local**

## Pré-requisitos
Para executar este projeto, certifique-se de ter as seguintes ferramentas instaladas:

- Visual Studio (ou IDE de sua preferência)
- .NET 8.0
- Postman (para testar as APIs)
- Conta no Microsoft Azure

## Criando a Infraestrutura no Azure

### 1. Criar um Grupo de Recursos
No portal do Azure, crie um novo Grupo de Recursos com o nome `flixdio`.

### 2. Criar o API Management
- Crie um novo **API Management Service** com o nome `apiflixdio01`.
- Defina o **Pricing Tier** como `Consumption`.

### 3. Criar o Azure Cosmos DB
- Crie um **Azure Cosmos DB** com o nome `cosmosdbflixdio01`.
- Selecione a opção **NoSQL** e utilize o modo **Serverless**.

### 4. Criar uma Conta de Armazenamento (Storage Account)
- Crie uma **Storage Account** com o nome `storageaccountdiod01`.
- Defina o **Primary Service** como `Azure Blob Storage` e ative o `Data Lake Storage Gen2`.
- Nas configurações avançadas, habilite o **Acesso Anônimo**.

### 5. Configurar Containers no Storage Account
- Acesse `storageaccountdiod01`.
- Navegue até `Navegador de Armazenamento > Contêineres de Blob`.
- Crie dois contêineres:
  - `video` (Permitir acesso para leitura)
  - `image` (Permitir acesso para leitura)

## Criando a Infraestrutura via Azure CLI

```bash
# Definição de variáveis
myRG=flixdio
myApiManagement=apiflixdio01
myCosmosDB=cosmosdbflixdio01
myLocation=eastus
myStorageAccount=storageaccountdiod0
myEmail=seu-email@dominio.com

# Criar o Grupo de Recursos
az group create --name $myRG --location $myLocation

# Criar o API Management
az apim create --name $myApiManagement -g $myRG -l $myLocation --publisher-email $myEmail --publisher-name FlixDio --sku-name Consumption

# Criar o Azure Cosmos DB
az cosmosdb create --name $myCosmosDB --resource-group $myRG --kind MongoDB --server-version 3.6 --locations regionName=WestUS failoverPriority=0 isZoneRedundant=False --enable-serverless true

# Criar a Storage Account
az storage account create --name $myStorageAccount --resource-group $myRG --location $myLocation --sku Standard_LRS --kind StorageV2 --allow-blob-public-access true

# Criar contêineres
az storage container create --name video --account-name $myStorageAccount
az storage container set-permission --name video --account-name $myStorageAccount --public-access blob
az storage container create --name image --account-name $myStorageAccount
az storage container set-permission --name image --account-name $myStorageAccount --public-access blob
```

## Criando a Azure Function para Upload no Storage Account

No Visual Studio, crie um novo projeto **Azure Function** chamado `fnPostDataStore` e utilize a função **Http Trigger**.

### Configurando Upload de Arquivos
Edite o `Program.cs` para permitir uploads de até 100MB:

```csharp
using Microsoft.AspNetCore.Server.Kestrel.Core;
using Microsoft.Azure.Functions.Worker.Builder;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = FunctionsApplication.CreateBuilder(args);
builder.ConfigureFunctionsWebApplication();

builder.Services.Configure<KestrelServerOptions>(options =>
{
    options.Limits.MaxRequestBodySize = 1024 * 1024 * 100; // 100MB
});

builder.Build().Run();
```

### Configurando Credenciais no `local.settings.json`
Acesse o portal do Azure e copie a chave de acesso da `Storage Account`. Depois, edite o arquivo `local.settings.json`:

```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "sua cadeia de conexão",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated"
    }
}
```

### Criando a Função de Upload
Edite `Functions1.cs` para validar e armazenar arquivos:

```csharp
public async Task<IActionResult> Run([HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req)
{
    _logger.LogInformation("Processando a imagem no storage ...");
    try
    {
        if (!req.Headers.TryGetValue("file-type", out var filetypeHeader))
        {
            return new BadRequestObjectResult("O cabeçalho file-type é obrigatório");
        }

        var fileType = filetypeHeader.ToString();
        var form = await req.ReadFormAsync();
        var file = form.Files["file"];

        if (file == null || file.Length == 0)
        {
            return new BadRequestObjectResult("Nenhum arquivo foi enviado");
        }

        string connectionString = Environment.GetEnvironmentVariable("AzureWebJobsStorage");
        string containerName = fileType;
        BlobContainerClient containerClient = new BlobContainerClient(connectionString, containerName);
        await containerClient.CreateIfNotExistsAsync();
        await containerClient.SetAccessPolicyAsync(PublicAccessType.BlobContainer);

        string blobName = file.FileName;
        var blob = containerClient.GetBlobClient(blobName);
        using (var stream = file.OpenReadStream())
        {
            await blob.UploadAsync(stream, true);
        }

        _logger.LogInformation($"Arquivo {file.FileName} armazenado com sucesso");
        return new OkObjectResult(new { Message = "Upload realizado com sucesso", BlobUri = blob.Uri });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Erro ao processar o upload");
        return new StatusCodeResult(StatusCodes.Status500InternalServerError);
    }
}
```

## Testando a API com Postman
- Configure um `POST` para o endpoint local da Azure Function.
- Adicione um **Header**: `file-type` com valor `image` ou `video`.
- No **Body**, envie um arquivo.
- Copie os retornos e valide o upload no Storage Account.

## Criando uma Azure Function para CosmosDB
Crie um novo projeto `fnPostDatabase` e adicione a função **Http Trigger**.

Edite `local.settings.json` com a cadeia de conexão do CosmosDB:

```json
{
    "IsEncrypted": false,
    "Values": {
        "AzureWebJobsStorage": "UseDevelopmentStorage=true",
        "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
        "CosmoDBConnection": "sua cadeia de conexão",
        "databaseName": "DioFlixDB",
        "ContainerName": "movies"
    }
}
```

---
Este README foi reformulado para maior clareza, organização e objetividade. Agora ele segue um fluxo mais intuitivo e fácil de acompanhar!

