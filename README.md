# Los pasos para replicar este ejemplo
## Craer Function App con Azure CLI

### Es necesario instalar lo siguiente 

[Net 6.0 SDK](https://dotnet.microsoft.com/en-us/download)

[Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cmacos%2Ccsharp%2Cportal%2Cbash#v2)

[Azure CLI version 2.4 o superior](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

### Validar lo siguiente 
 - Validar la version de azure functions core tools
    
        func --version  

 - Validar los SDKs de .NEt instalados

        dotnet --list-sdks 
 - Validar la version de Azure CLI

        az --version 
 - Hacer login en la cuenta de Azure

        az login 


1.- Ejecutar el comando az login para realizar el login en la cuenta de azure, 
    abrira el navegador para iniciar sesion en al cuenta de azure, 
    proporcione las credenciales e inicie sesión

        az login

2.- Para crear la funcion se va ejecutar el comando func init, antes que nada posicionarse en el directorio que desea crear el ejmeplo de Azure Functions
    ejecutar el comando siguiente nombreFuncion es el nombre que le quiere dar a su funcion  --dotnet indica que el runtime a utilizar es dotnet con lenguaje C# 
        
        func init nombreFuncion --dotnet 

3.- Posicionarse dentro del directorio creado con el nombreFuncion del paso anterior 

        cd nombreFuncion

4.- Ejecutar el comando func new como se muestra a continucación 
        
        func new --name HttpExampleFunction --template "HTTP trigger" --authlevel "anonymous"

 Esto crea un archivo .cs con el código de la funcion, es un ejemplo básico que espera un parametro name.
    Despues de validar el parametro ya sea en el query o en body muestra un mensaje.

5.- Ejecutar la función con el comando 
        func start
    si todo esta ok mostrara la URL en loscalhost y puerto donde se esta ejecutando la funcion
    Http Functions:

         HttpExample: [GET,POST] http://localhost:{PORT}/api/HttpExampleFunction

6.-  Prueba la funcion pasando un valor al parametro name
        
        http://localhost:{PORT}/api/HttpExampleFunction?name=valor

7.- PAra probar en POSTMAN u otro cliente puede haverlo por metodos GET y POST
    
    GET http://localhost:{PORT}/api/HttpExampleFunction?name=valor
    POST 
        {
            "name"="valor del parametro"
        }

8.- CTRL+C para detener la ejecucion de la funcion.

***
***
***
# Deploy Function App in Azure con Azure CLI

Para publicar nuestra Function App debemos crear el grupo de recursos donde crearemos nuestro servicio de Azure Function

1.- Login en la cuenta de Azure, ejecutando el comando 
        
        az login

2.- Crear el grupo de recursos contenedor con el siguiente comando
        
        az group create --name rgMyLocalFunctions --location "eastus" or westus
3.- Create storage account en mi grupo de recursos
        
        az storage account create --name samylocalfunctions --location eastus --resource-group rgMyLocalFunctions --sku Standard_LRS 

 Standard_LRS  especifica una cuenta de proposito general
    nombre de storage acoount debe ser en minusculas y numeros no cepta otros caracteres o mayusculas

4.- Crear Function App en AZURE
    
    az functionapp create --resource-group rgMyLocalFunctions --consumption-plan-location eastus --runtime dotnet --functions-version 4 --name MyLocalFunction --storage-account samylocalfunctions

5.- Publicar mi Function App en azure 
    
    func azure functionapp publish MyLocalFunction

6.- Examinar el log en tiempo real 
    
    func azure functionapp logstream MyLocalFunction

7.- Limpar o liberar recursos creados, como buena practica debemos liberar los recursos que ya no vamos a utilizar, esto ayudara que no nos genere costos los servicos que creamos y que ya no utilizaremos
    
    az group delete --name rgMyLocalFunctions

---
>## Conectar Azure Functions a Azure Storage usando linea de comandos

1.- Obtenemos las configuraciones de nuestra Function App en azure, 
    no mostrar en el video estos datos ya que son datos sensibles y deben guardarlos y no compartilos
    En el archivo local.settings.json se sobreescriben los valores de configuracion y cadenas de conexión 
    utilizaremos el valor de  AzureWebJobsStorage
        
        func azure functionapp fetch-app-settings MyLocalFunction

2.- Agregamos el siguiente paquete nuget para trabajar ocn storage 
    
    dotnet add package Microsoft.Azure.WebJobs.Extensions.Storage

3.- Agregamos un enlace a nuestra cuenta de alamcenamiento y agregamos al atributo de la funcion el qiguiente parametro
        
        [Queue("outqueue"),StorageAccount("AzureWebJobsStorage")] ICollector<string> msg,

            [FunctionName("HttpExample")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req, 
            [Queue("outqueue"),StorageAccount("AzureWebJobsStorage")] ICollector<string> msg, 
            ILogger log)

        if (!string.IsNullOrEmpty(name))
        {
            // Add a message to the output collection.
            msg.Add(name);
        }


        [FunctionName("HttpExampleFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "get", "post", Route = null)] HttpRequest req,
            [Queue("outqueue"),StorageAccount("AzureWebJobsStorage")] ICollector<string> msg,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            string name = req.Query["name"];

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            name = name ?? data?.name;

            if (!string.IsNullOrEmpty(name))
            {
                // Add a message to the output collection.
                msg.Add(name);
            }
            string responseMessage = string.IsNullOrEmpty(name)
                ? "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response."
                : $"Hello, {name}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }

4.- Ejecute la funcion app
        
        func start 


___
## Visualizar los mensajes in azure storage queue

1.- 
        
        export AZURE_STORAGE_CONNECTION_STRING="<MY_CONNECTION_STRING>"

2.- 
        
        az storage queue list --output tsv

3.- 
        
        echo `echo $(az storage message get --queue-name outqueue -o tsv --query '[].{Message:content}') | base64 --decode`

4.- 

        func azure functionapp publish <APP_NAME>


## Clean resources 

Ejecutar el comando para borrar el grupo de recursos

        az group delete --name NombreDelGrupoDeRecursos