# Exercise 3: Explore GitHub advance security features 

Duration 60 minutes

Once the Fabrikamk Medical Conferences developer workflow has been deployed, we can apply the github advance security features.

## Task 1:
**Help references**



## Task 2: Enabling Github Dependabot 

## Task 3: Enabling Codescanning and CodeQL alerts 

What is codescanning? 
Code scanning is a feature that you use to analyze the code in a GitHub repository to find security vulnerabilities and coding errors. Any problems identified by the analysis are shown in GitHub.

1. Make sure your repository is public
**Note** If the repository visibility is private, go to the settings of the repository and change the visibility to public.
1. Go to seetings tab of the repository, then under security tab select code security and analysis
     
   ![](media/imgcs.png)

    ![Azure Resource Group containing cloud resources to which GitHub will deploy containers via the workflows defined in previous steps.](media/hol-ex2-task1-step5-1.png "Azure Resource Group")

2. In the LabVM, open the `seed-cosmosdb.ps1` PowerShell script in the `C:\Workspaces\lab\mcw-continuous-delivery-lab-files\infrastructure` folder of your lab files GitHub repository and replace `$studentprefix` variable value with **<inject key="Deploymentid" />**, update `$githubAccount = "Your github account name here"` variable with your GitHub username.

    ```pwsh
    $studentprefix = "Your 3 letter abbreviation here"
    $githubAccount = "hatboyzero"
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $cosmosDBName = "fabmedical-cdb-" + $studentprefix
    ```
   
   ![](media/seedcosmos.png)

3. Observe the call to fetch the MongoDB connection string for the CosmosDB database.

    ```pwsh
    # Fetch CosmosDB Mongo connection string
    $mongodbConnectionString = `
        $(az cosmosdb keys list `
            --name $cosmosDBName `
            --resource-group $resourcegroupName `
            --type connection-strings `
            --query 'connectionStrings[0].connectionString')
    ```

4. Note the call to seed the CosmosDB database using the MongoDB connection string passed as an environment variable (`MONGODB_CONNECTION`) to the `fabrikam-init` docker image we built in the previous exercise using `docker-compose`.

    ```pwsh
    # Seed CosmosDB database
    docker run -ti `
        -e MONGODB_CONNECTION="$mongodbConnectionString" `
        ghcr.io/$githubAccount/fabrikam-init:main
    ```
    
5.  Before you pull this image, you may need to authenticate with the GitHub Docker registry. To do this, run the following command before you execute the script. Fill the placeholders appropriately. 

    >**Note**: **Username is case sensitive make sure you enter the exact username and personal access token.**

    ```pwsh
    docker login ghcr.io -u USERNAME -p PERSONAL ACCESS TOKEN 
    ```

6. In your Powershell Terminal log in to Azure by running the following command. this will open edge browser, you need to enter the login details as below:
   
    
     * Azure Usename/Email: <inject key="AzureAdUserEmail"></inject> 
 
     * Azure Password: <inject key="AzureAdUserPassword"></inject> 
 

    ```pwsh
    az login
    ```
    
7. Run the `seed-cosmosdb.ps1` PowerShell script. Browse to the Azure Portal and navigate to **fabmedical-cdb-<inject key="DeploymentID" enableCopy="false" />** Cosmos DB resource and  and verify that the CosmosDB instance has been seeded.

     ```pwsh
      cd C:\Workspaces\lab\mcw-continuous-delivery-lab-files\infrastructure
      ./seed-cosmosdb.ps1
     ```
       
8. Once the script execution is completed, Browse to the Azure Portal and navigate to **fabmedical-cdb-<inject key="DeploymentID" enableCopy="false" />** Cosmos DB resource and select **Data Explorer** from the left menu  and verify that the CosmosDB instance has been seeded.

    ![Azure CosmosDB contents displayed via the CosmosDB explorer in the Azure CosmosDB resource detail.](media/hol-ex2-task1-step9-1.png "Azure CosmosDB Seeded Contents")

9. Below the `sessions` collection, select **Scale & Settings (1)** and **Indexing Policy (2)**.

    ![Opening indexing policy for the sessions collection.](./media/sessions-collection-indexing-policy.png "Indexing policy configuration")

10. Create a Single Field indexing policy for the `startTime` field (1). Then, select **Save** (2).

    ![Creating an indexing policy for the startTime field.](./media/start-time-indexing-mongo.png "startTine field indexing")

11. Open the `configure-webapp.ps1` PowerShell script in the `C:\Workspaces\lab\mcw-continuous-delivery-lab-files\infrastructure` folder of your lab files GitHub repository and replace `$studentprefix` variable value with **<inject key="Deploymentid" />** on the first line. Once the changes is done, make sure to save the file.

    ```pswh
    $studentprefix = "hbs"                                  # <-- Modify this value
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $cosmosDBName = "fabmedical-cdb-" + $studentprefix
    ```

12. Observe the call to configure the Azure Web App using the MongoDB connection string passed as an environment variable (`MONGODB_CONNECTION`) to the web application.

    ```pwsh
    # Configure Web App
    az webapp config appsettings set `
        --name $webappName `
        --resource-group $resourcegroupName `
        --settings MONGODB_CONNECTION=$mongodbConnectionString
    ```

13. Run the `configure-webapp.ps1` PowerShell script.

    ```pwsh
    cd C:\Workspaces\lab\mcw-continuous-delivery-lab-files\infrastructure
    ./configure-webapp.ps1
    ```

14. Once the script execution is completed, Browse to the Azure Portal and search for **fabmedical-web-<inject key="DeploymentID" enableCopy="false" />** App service and select **Configuration** from left side menu and verify that the environment variable `MONGODB_CONNECTION` has been added to the Azure Web Application settings.

    ![Azure Web Application settings reflecting the `MONGODB_CONNECTION` environment variable configured via PowerShell.](media/hol-ex2-task1-step12-1.png "Azure Web Application settings")

15. Take the GitHub Personal Access Token you obtained in the Before the Hands-On Lab guided instruction and assign it to the `GITHUB_TOKEN` environment variable in PowerShell. We will need this environment variable for the `deploy-webapp.ps1` PowerShell script, but we do not want to add it to any files that may get committed to the repository since it is a secret value.

    ```pwsh
    $env:CR_PAT="<GitHub Personal Access Token>"
    ```


### Task 2: Deployment Automation to Azure Web App

 1. Open the `deploy-webapp.ps1` PowerShell script in the `infrastructure` folder of your lab files GitHub repository and add your GitHub account username for the `$githubAccount` variable on the second line. Once the changes are done make sure to save the file. 

    >**Note:** We have already updated the $studentprefix in this file with the required value. 

    ```pwsh
    $studentprefix = "deploymentID"                                 
    $githubAccount = "GitHub account username"                           # <-- Modify this value
    $resourcegroupName = "fabmedical-rg-" + $studentprefix
    $webappName = "fabmedical-web-" + $studentprefix
    ```

 1. Note the call to deploy the Azure Web Application using the `docker-compose.yml` file we modified in the previous exercise.

    ```pwsh
    # Deploy Azure Web App
    az webapp config container set `
        --docker-registry-server-password $env:CR_PAT `
        --docker-registry-server-url https://docker.pkg.github.com `
        --docker-registry-server-user $githubAccount `
        --multicontainer-config-file ./../docker-compose.yml `
        --multicontainer-config-type COMPOSE `
        --name $webappName `
        --resource-group $resourcegroupName
    ```

 1. Run the `deploy-webapp.ps1` PowerShell script.

     ```pwsh
     ./deploy-webapp.ps1
     ```

    > **Note**: Make sure to run the `deploy-webapp.ps1` script from the `infrastructure` folder

 1. Browse to the Azure Portal and verify that the Azure Web Application is running by checking the `Log stream` blade of the Azure Web Application detail page.

    ![Azure Web Application Log Stream displaying the STDOUT and STDERR output of the running container.](media/hol-ex2-task2-step4-1.png "Azure Web Application Log Stream")

 1. Browse to the `Overview` blade of the Azure Web Application detail page and find the web application URL. Browse to that URL to verify the deployment of the web application. It might take a few minutes for the web application to reflect new changes.

    ![The Azure Web Application Overview detail in Azure Portal.](media/hol-ex2-task2-step5-1.png "Azure Web Application Overview")
   
    >**Note:** If you see any nginx error while browsing the App URL, that's fine as it will take a few minutes to reflect the changes.
    
    ![The Contoso Conference website is hosted in Azure.](media/hol-ex2-task2-step5-2.png "Azure hosted Web Application")
    

### Task 3: Continuous Deployment with GitHub Actions

With the infrastructure in place, we can set up continuous deployment with GitHub Actions.
