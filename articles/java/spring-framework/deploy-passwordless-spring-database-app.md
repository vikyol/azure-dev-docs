---
title: 'Tutorial: Deploy to Azure Spring Apps with passwordless connection to Azure database'
description: Create a Spring Boot application with passwordless connection to an Azure database and deploy to Azure Spring Apps.
ms.author: xiada
ms.service: spring-apps
ms.topic: tutorial
ms.date: 09/27/2022
ms.custom: passwordless-java
---

# Tutorial: Deploy a Spring application to Azure Spring Apps with a passwordless connection to an Azure database

This article shows you how to use passwordless connections to Azure databases in Spring Boot applications deployed to Azure Spring Apps.

In this tutorial, you'll complete the following tasks using the Azure portal or the Azure CLI. Both methods are explained in the following procedures.

> [!div class="checklist"]
> - Provision an instance of Azure Spring Apps.
> - Build and deploy apps to Azure Spring Apps.
> - Run apps connected to Azure databases using managed identity.

> [!NOTE]
> This tutorial doesn't work for R2DBC.

## Prerequisites

- [JDK 8 or JDK 11](../fundamentals/java-jdk-install.md).
- An Azure subscription. If you don't already have one, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
- [Azure CLI](/cli/azure/install-azure-cli) 2.41.0 or above required.
- The Azure Spring Apps extension. You can install the extension by using the command: `az extension add --name spring`.
- A [Git](https://git-scm.com/downloads) client.
- [cURL](https://curl.haxx.se) or a similar HTTP utility to test functionality.
- MySQL command line client if you choose to run Azure Database for MySQL. You can connect to your server with Azure Cloud Shell using a popular client tool, the [mysql.exe](https://dev.mysql.com/downloads/) command-line tool. Alternatively, you can use the `mysql` command line in your local environment.
- [ODBC Driver 18 for SQL Server](/sql/connect/odbc/download-odbc-driver-for-sql-server) if you choose to run Azure SQL Database.

## Prepare the working environment

First, set up some environment variables by using the following commands:

```bash
export AZ_RESOURCE_GROUP=passwordless-tutorial-rg
export AZ_DATABASE_SERVER_NAME=<YOUR_DATABASE_SERVER_NAME>
export AZ_DATABASE_NAME=demodb
export AZ_LOCATION=<YOUR_AZURE_REGION>
export AZ_SPRING_APPS_SERVICE_NAME=<YOUR_AZURE_SPRING_APPS_SERVICE_NAME>
export AZ_SPRING_APPS_APP_NAME=hellospring
export AZ_DB_ADMIN_USERNAME=<YOUR_DB_ADMIN_USERNAME>
export AZ_DB_ADMIN_PASSWORD=<YOUR_DB_ADMIN_PASSWORD>
export AZ_USER_IDENTITY_NAME=<YOUR_USER_ASSIGNED_MANAGEMED_IDENTITY_NAME>
```

Replace the placeholders with the following values, which are used throughout this article:

- `<YOUR_DATABASE_SERVER_NAME>`: The name of your Azure Database server, which should be unique across Azure.
- `<YOUR_AZURE_REGION>`: The Azure region you'll use. You can use `eastus` by default, but we recommend that you configure a region closer to where you live. You can see the full list of available regions by using the command `az account list-locations`.
- `<YOUR_AZURE_SPRING_APPS_SERVICE_NAME>`: The name of your Azure Spring Apps instance. The name must be between 4 and 32 characters long and can contain only lowercase letters, numbers, and hyphens. The first character of the service name must be a letter and the last character must be either a letter or a number.
- `<AZ_DB_ADMIN_USERNAME>`: The admin username of your Azure database server.
- `<AZ_DB_ADMIN_PASSWORD>`: The admin password of your Azure database server.
- `<YOUR_USER_ASSIGNED_MANAGEMED_IDENTITY_NAME>`: The name of your user assigned managed identity server, which should be unique across Azure.

## Provision an instance of Azure Spring Apps

Use the following steps to provision an instance of Azure Spring Apps.

1. Update Azure CLI with the Azure Spring Apps extension by using the following command:

   ```azurecli
   az extension update --name spring
   ```

1. Sign in to the Azure CLI and choose your active subscription by using the following commands:

   ```azurecli
   az login
   az account list --output table
   az account set --subscription <name-or-ID-of-subscription>
   ```

1. Use the following commands to create a resource group to contain your Azure Spring Apps service and an instance of the Azure Spring Apps service:

   ```azurecli
   az group create \
       --name $AZ_RESOURCE_GROUP \
       --location $AZ_LOCATION
   az spring create \
       --resource-group $AZ_RESOURCE_GROUP \
       --name $AZ_SPRING_APPS_SERVICE_NAME
   ```

## Create an Azure database instance

Use the following steps to provision an Azure Database instance.

### [Azure Database for MySQL](#tab/mysql)

1. Create an Azure Database for MySQL server by using the following command:

   ```azurecli
   az mysql flexible-server create \
       --resource-group $AZ_RESOURCE_GROUP \
       --name $AZ_DATABASE_SERVER_NAME \
       --location $AZ_LOCATION \
       --admin-user $AZ_DB_ADMIN_USERNAME \
       --admin-password $AZ_DB_ADMIN_PASSWORD \
       --yes
   ```

1. Create a new database by using the following command:

   ```azurecli
   az mysql flexible-server db create \
       --resource-group $AZ_RESOURCE_GROUP \
       --database-name $AZ_DATABASE_NAME \
       --server-name $AZ_DATABASE_SERVER_NAME
   ```

### [Azure Database for PostgreSQL](#tab/postgresql)

1. Create an Azure Database for PostgreSQL server by using the following command:

   ```azurecli
   az postgres server create \
       --resource-group $AZ_RESOURCE_GROUP \
       --name $AZ_DATABASE_SERVER_NAME \
       --location $AZ_LOCATION \
       --sku-name B_Gen5_1 \
       --storage-size 5120 \
       --admin-user $AZ_DB_ADMIN_USERNAME \
       --admin-password $AZ_DB_ADMIN_PASSWORD
   ```

1. The PostgreSQL server is empty, so create a new database by using the following command:

   ```azurecli
   az postgres db create \
       --resource-group $AZ_RESOURCE_GROUP \
       --server-name $AZ_DATABASE_SERVER_NAME \
       --name $AZ_DATABASE_NAME
   ```

### [Azure SQL Database](#tab/sqlserver)

1. Create an Azure SQL Database server by using the following command:

   ```azurecli
   az sql server create \
       --location $AZ_LOCATION \
       --resource-group $AZ_RESOURCE_GROUP \
       --name $AZ_DATABASE_SERVER_NAME \
       --admin-user $AZ_DB_ADMIN_USERNAME \
       --admin-password $AZ_DB_ADMIN_PASSWORD
   ```

1. The SQL server is empty, so create a new database by using the following command:

   ```azurecli
   az sql db create \
       --resource-group $AZ_RESOURCE_GROUP \
       --server $AZ_DATABASE_SERVER_NAME \
       --name $AZ_DATABASE_NAME
   ```

---

## Create an app with a public endpoint assigned

Use the following command to create the app. If you selected Java version 11 when generating the Spring project, include the argument `--runtime-version=Java_11`.

   ```azurecli
   az spring app create \
       --resource-group $AZ_RESOURCE_GROUP \
       --service $AZ_SPRING_APPS_SERVICE_NAME \
       --name $AZ_SPRING_APPS_APP_NAME \
       --assign-endpoint true
   ```

## Connect Azure Spring Apps to the Azure database

### [Azure Database for MySQL](#tab/mysql)

First, use the following command to create a user-assigned managed identity for Azure Active Directory authentication. For more information, see [Set up Azure Active Directory authentication for Azure Database for MySQL - Flexible Server](/azure/mysql/flexible-server/how-to-azure-ad).

```azurecli
AZ_IDENTITY_RESOURCE_ID=$(az identity create \
    --name $AZ_USER_IDENTITY_NAME \
    --resource-group $AZ_RESOURCE_GROUP \
    --query id \
    --output tsv)
```

> [!IMPORTANT]
> After creating the user-assigned identity, ask your *Global Administrator* or *Privileged Role Administrator* to grant the following permissions for this identity: `User.Read.All`, `GroupMember.Read.All`, and `Application.Read.ALL`. For more information, see the [Permissions](/azure/mysql/flexible-server/concepts-azure-ad-authentication#permissions) section of [Active Directory authentication](/azure/mysql/flexible-server/concepts-azure-ad-authentication).

Next, use the following command to create a passwordless connection to the database.

```azurecli
az spring connection create mysql-flexible \
    --resource-group $AZ_RESOURCE_GROUP \
    --service $AZ_SPRING_APPS_SERVICE_NAME \
    --app $AZ_SPRING_APPS_APP_NAME \
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity mysql-identity-id=$AZ_IDENTITY_RESOURCE_ID
```

This Service Connector command will do the following tasks in the background:

- Enable system-assigned managed identity for the app `$AZ_SPRING_APPS_APP_NAME` hosted by Azure Spring Apps.
- Set the Azure Active Directory admin to the current signed-in user.
- Add a database user named `$AZ_SPRING_APPS_SERVICE_NAME/apps/$AZ_SPRING_APPS_APP_NAME` for the managed identity created in step 1 and grant all privileges of the database `$AZ_DATABASE_NAME` to this user.
- Add two configurations to the app `$AZ_SPRING_APPS_APP_NAME`: `spring.datasource.url` and `spring.datasource.username`.

  > [!NOTE]
  > If you see the error message `The subscription is not registered to use Microsoft.ServiceLinker`, run the command `az provider register --namespace Microsoft.ServiceLinker` to register the Service Connector resource provider, then run the connection command again.

### [Azure Database for PostgreSQL](#tab/postgresql)

Use the following command to create a passwordless connection to the database.

```azurecli
az spring connection create postgres \
    --resource-group $AZ_RESOURCE_GROUP \
    --service $AZ_SPRING_APPS_SERVICE_NAME \
    --app $AZ_SPRING_APPS_APP_NAME \
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity
```

This Service Connector command will do the following tasks in the background:

- Enable system-assigned managed identity for the app `$AZ_SPRING_APPS_APP_NAME` hosted by Azure Spring Apps.
- Set the Azure Active Directory admin to current sign-in user.
- Add a database user named `$AZ_SPRING_APPS_SERVICE_NAME/apps/$AZ_SPRING_APPS_APP_NAME` for the managed identity created in step 1 and grant all privileges of the database `$AZ_DATABASE_NAME` to this user.
- Add two configurations to the app `$AZ_SPRING_APPS_APP_NAME`: `spring.datasource.url` and `spring.datasource.username`.

  > [!NOTE]
  > If you see the error message `The subscription is not registered to use Microsoft.ServiceLinker`, run the command `az provider register --namespace Microsoft.ServiceLinker` to register the Service Connector resource provider, then run the connection command again.

### [Azure SQL Database](#tab/sqlserver)

> [!NOTE]
> Please make sure Azure CLI use the 64-bit Python, 32-bit Python has compatibility issue with the command's dependency [pyodbc](https://pypi.org/project/pyodbc/). 
> The Python information of Azure CLI can be got with command `az --version`. If it shows `[MSC v.1929 32 bit (Intel)]`, then it means it use 32-bit Python.
> The solution is to install 64-bit Python and install Azure CLI from [PyPI](https://pypi.org/project/azure-cli/).

Use the following command to create a passwordless connection to the database.

```azurecli
az spring connection create sql \
    --resource-group $AZ_RESOURCE_GROUP \
    --service $AZ_SPRING_APPS_SERVICE_NAME \
    --app $AZ_SPRING_APPS_APP_NAME \
    --target-resource-group $AZ_RESOURCE_GROUP \
    --server $AZ_DATABASE_SERVER_NAME \
    --database $AZ_DATABASE_NAME \
    --system-identity
```

This Service Connector command will do the following tasks in the background:

- Enable system-assigned managed identity for the app `$AZ_SPRING_APPS_APP_NAME` hosted by Azure Spring Apps.
- Set the Azure Active Directory admin to current sign-in user.
- Add a database user named `$AZ_SPRING_APPS_SERVICE_NAME/apps/$AZ_SPRING_APPS_APP_NAME` for the managed identity created in step 1 and grant all privileges of the database `$AZ_DATABASE_NAME` to this user.
- Add one configuration to the app `$AZ_SPRING_APPS_APP_NAME`: `spring.datasource.url`.

  > [!NOTE]
  > If you see the error message `The subscription is not registered to use Microsoft.ServiceLinker`, run the command `az provider register --namespace Microsoft.ServiceLinker` to register the Service Connector resource provider, then run the connection command again.

---

## Build and deploy the app

The following steps describe how to download, configure, build, and deploy the sample application.

1. Use the following command to clone the sample code repository:

   ### [Azure Database for MySQL](#tab/mysql)

   ```bash
   git clone https://github.com/Azure-Samples/quickstart-spring-data-jdbc-mysql passwordless-sample
   ```

   ### [Azure Database for PostgreSQL](#tab/postgresql)

   ```bash
   git clone https://github.com/Azure-Samples/quickstart-spring-data-jdbc-postgresql passwordless-sample
   ```

   ### [Azure SQL Database](#tab/sqlserver)

   ```bash
   git clone https://github.com/Azure-Samples/quickstart-spring-data-jdbc-sql-server passwordless-sample
   ```

1. Add the following dependency to your *pom.xml* file:

   ### [Azure Database for MySQL](#tab/mysql)

   ```xml
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-cloud-azure-starter-jdbc-mysql</artifactId>
       <version>4.5.0-beta.1</version>
   </dependency>
   ```

   This dependency adds support for the Spring Cloud Azure starter.

   ### [Azure Database for PostgreSQL](#tab/postgresql)

   ```xml
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-cloud-azure-starter-jdbc-postgresql</artifactId>
       <version>4.5.0-beta.1</version>
   </dependency>
   ```

   This dependency adds support for the Spring Cloud Azure starter.

   ### [Azure SQL Database](#tab/sqlserver)

   ```xml
   <dependency>
       <groupId>com.azure</groupId>
       <artifactId>azure-identity</artifactId>
       <version>1.5.4</version>
   </dependency>
   ```

   There's currently no Spring Cloud Azure starter for Azure SQL Database, but the `azure-identity` dependency is required.

1. Use the following command to update the *application.properties* file:

   ### [Azure Database for MySQL](#tab/mysql)

   ```bash
   cat << EOF > passwordless-sample/src/main/resources/application.properties

   logging.level.org.springframework.jdbc.core=DEBUG
   spring.datasource.azure.passwordless-enabled=true
   spring.sql.init.mode=always

   EOF
   ```

   ### [Azure Database for PostgreSQL](#tab/postgresql)

   ```bash
   cat << EOF > passwordless-sample/src/main/resources/application.properties

   logging.level.org.springframework.jdbc.core=DEBUG
   spring.datasource.azure.passwordless-enabled=true
   spring.sql.init.mode=always

   EOF
   ```

   ### [Azure SQL Database](#tab/sqlserver)

   ```bash
   cat << EOF > passwordless-sample/src/main/resources/application.properties

   logging.level.org.springframework.jdbc.core=DEBUG
   spring.sql.init.mode=always

   EOF
   ```

1. Use the following commands to build the project using Maven:

   ```bash
   cd passwordless-sample
   ./mvnw clean package -DskipTests
   ```

1. Use the following command to deploy the *target/demo-0.0.1-SNAPSHOT.jar* file for the app:

   ```azurecli
   az spring app deploy \
       --name $AZ_SPRING_APPS_APP_NAME \
       --service $AZ_SPRING_APPS_SERVICE_NAME \
       --resource-group $AZ_RESOURCE_GROUP \
       --artifact-path target/demo-0.0.1-SNAPSHOT.jar
   ```

1. Query the app status after deployment by using the following command:

   ```azurecli
   az spring app list \
       --service $AZ_SPRING_APPS_SERVICE_NAME \
       --resource-group $AZ_RESOURCE_GROUP \
       --output table
   ```

   You should see output similar to the following example.

   ```
   Name               Location    ResourceGroup    Production Deployment    Public Url                                           Provisioning Status    CPU    Memory    Running Instance    Registered Instance    Persistent Storage
   -----------------  ----------  ---------------  -----------------------  ---------------------------------------------------  ---------------------  -----  --------  ------------------  ---------------------  --------------------
   <app name>         eastus      <resource group> default                                                                       Succeeded              1      2         1/1                 0/1                    -
   ```

## Clean up resources

To clean up all resources used during this tutorial, delete the resource group by using the following command:

```azurecli
az group delete \
    --name $AZ_RESOURCE_GROUP \
    --yes
```

## Next steps

- [Spring Cloud Azure documentation](./index.yml)
