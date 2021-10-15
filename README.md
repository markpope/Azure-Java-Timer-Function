# Azure Java Timer Function that accesses a Key Vault

Functions are great for exposing microservice APIs, but they can be used to schedule business logic as well by using @TimerTrigger. This annotation accepts a cron formatted schedule that defines when it should run. In this project  several REST APIs are made to the same host in a class referenced from an maven included jar. The API token should not be hard coded in my jar or the Azure Function. It should be stored in a Key Vault and accessed programatically. Azure Functions written in .NET/C# can access Key Vaults without extra code.

## Create a Maven project from an Azure archetype

### Consider changing groupId and package below  before executing to create the maven project

```
mvn -DarchetypeGroupId=com.microsoft.azure \
    -DarchetypeArtifactId=azure-functions-archetype \
    -DarchetypeVersion=1.40 \
    -DarchetypeRepository=https://repo.maven.apache.org/maven2/ \
    -DgroupId=com.pope \
    -DartifactId=JavaFunction \
    -Dversion=1.0 \
    -Dpackage=com.pope.javafunction  \
    -DresourceGroup=java-functions-group \
    -DappName=$(artifactId) \
    -DjavaVersion=8 \
    -DappServicePlanName=java-functions-app-service-plan \
    -Dtrigger=TimerTrigger \
    -DappRegion=<b>eastus</b> \
    -Ddocker=false \
    -Darchetype.interactive=false  \
    --batch-mode org.apache.maven.plugins:maven-archetype-plugin:3.1.2:generate

```

### Create a Azure KeyVault and add a Secret

1. Create a Resource Group i.e. rg-dev
2. Create a Key Vault in rg-dev i.e. keyvault-dev
3. Create a Function App i.e. function-dev specify Java, version 8
4. Give function app function-dev a managed identity. Copy Object ID.
5. Grant function-dev access to key vault by adding a policy to the key vault for the function-dev Object ID for Secret Management role.
6. Add a secret to the keyvault-dev
7. Edit function-dev Configuration Properties and add <b>FUNCTIONS_WORKER_JAVA_LOAD_APP_LIBS=true</b>
8. On the left side of the Key Vault, select Secrets and add a <b>Name</b> and <b>Secret</b>.

### Add code to the generated Function class to access the Key Vault, and call your Java code.

1. Add the KeyVault Id and secret name in the Function method below.
2. Set the timer scheduler, its cron based.
3. Add you java code or classes from included jars in the try{-}catch{} block.

```
package com.pope;

import java.time.*;
import com.microsoft.azure.functions.annotation.*;
import com.microsoft.azure.functions.*;
import java.util.logging.Level;
import java.util.logging.Logger;

import com.azure.identity.DefaultAzureCredentialBuilder;
import com.azure.security.keyvault.secrets.SecretClient;
import com.azure.security.keyvault.secrets.SecretClientBuilder;
import com.pope.RestClient;

/**
 * Azure Functions with Timer trigger.
 */
public class Function {

	/**
	 * This function will be invoked periodically according to the specified schedule.
	 */
	@FunctionName("Function")
	public void run(
		@TimerTrigger(name = "timerInfo", schedule = "0 * * * * *") String timerInfo,
		final ExecutionContext context
	) {

		SecretClient secretClient = new SecretClientBuilder()
			.vaultUrl("https:// <Your KeyVault> .vault.azure.net/")
			.credential(new DefaultAzureCredentialBuilder().build())
			.buildClient();
		String token = secretClient.getSecret("<Secret Name>").getValue();

		context.getLogger().info("" + token);

		context.getLogger().info("Client Timer trigger function executed at: " + LocalDateTime.now());
		try {
			RestClient restClient = new RestClient();
			restClient.setToken(token);
			restClient.processInitiatives();
		} catch (Throwable ex) {
			Logger.getLogger(Function.class.getName()).log(Level.SEVERE, null, ex);
		}
	}
}
```

### Deploy to Azure

At the time of this writing the Azure portal or CLI does not support uploading Java jars. Launch VS Code, log into Azure, open this project and deploy the function.

New to VSCode? Follow these steps to get up and running:

1. Install .NET Core SDK https://dotnet.microsoft.com/download
2. Install Visual Studio Code https://code.visualstudio.com/Download
3. Install Azure CLI https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest
4. Install Python 3.9 https://www.python.org/downloads/
    * During installation add Python 3.9 to PATH
5. Install Docker https://docs.docker.com/docker-for-windows/install/
   * You need to have Hyper-V or Virtual Box installed first.
6. Install Node.js LTS https://nodejs.org/en/#home-downloadhead
   * Verify with node --version
7. Install VS Code extensions from the IDE Extensions Menu:
    * vsciot-vscode.azure-iot-tools
    * ms-vscode.csharp
    * ms-vscode.vscode-node-azure-pack
    * vsciot-vscode.vscode-dtdl
8. Verify .NET Core 3x SDK is configured correctly.
    * Dotnet --version
9. Sign into Azure for from VS Code
10. Install VS Code Extensions Menu:
    * Azure Functions
    * Maven for Java
    * Extensions for Java
    * Project Manager for Java
    * DTDL
    * Language sup