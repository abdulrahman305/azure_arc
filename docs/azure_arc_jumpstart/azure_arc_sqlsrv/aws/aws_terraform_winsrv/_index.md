---
title: "Onboard a AWS EC2 instance with Windows Server & Microsoft SQL Server to Azure Arc"
linkTitle: "Onboard a AWS EC2 instance with Windows Server & Microsoft SQL Server to Azure Arc"
weight: 1
---

## Onboard a AWS EC2 instance with Windows Server & Microsoft SQL Server to Azure Arc

The following README will guide you on how to use the provided [Terraform](https://www.terraform.io/) plan to deploy a Windows Server installed with Microsoft SQL Server 2019 (Developer edition) in a Amazon Web Services (AWS) EC2 instance and connect it as an Azure Arc enabled SQL server resource.

By the end of the guide, you will have an AWS EC2 instance installed with Windows Server 2019 with SQL Server 2019, projected as an Azure Arc enabled SQL server and a running SQL assessment with data injected to Azure Log Analytics workspace.

## Prerequisites

* Clone this repo

    ```console
    git clone https://github.com/microsoft/azure_arc.git
    ```

* [Install or update Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest). **Azure CLI should be running version 2.7** or later. Use ```az --version``` to check your current installed version.

* [Create a free Amazon Web Services account](https://aws.amazon.com/free/) if you don't already have one.

* [Install Terraform >=0.12](https://learn.hashicorp.com/terraform/getting-started/install.html)

* Create Azure Service Principal (SP)

    To connect the EC2 instance to Azure Arc, an Azure Service Principal assigned with the "Contributor" role is required. To create it, login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/)).

    ```console
    az login
    az ad sp create-for-rbac -n "<Unique SP Name>" --role contributor
    ```

    For example:

    ```console
    az ad sp create-for-rbac -n "http://AzureArcServers" --role contributor
    ```

    Output should look like this:

    ```console
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "AzureArcServers",
    "name": "http://AzureArcServers",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **Note: It is optional but highly recommended to scope the SP to a specific [Azure subscription and resource group](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest)**

## Create a new AWS IAM Role & Key

Create AWS User IAM Key. An access key grants programmatic access to your resources which we will be using later on in this guide.

* Navigate to the [IAM Access page](https://console.aws.amazon.com/iam/home#/home).

    ![Create AWS IAM Role & Key](./01.png)

* Select the **Users** from the side menu.

    ![Create AWS IAM Role & Key](./02.png)

* Select the **User** you want to create the access key for.

    ![Create AWS IAM Role & Key](./03.png)

* Select ***Security credentials** of the **User** selected.

    ![Create AWS IAM Role & Key](./04.png)

* Under **Access Keys** select **Create Access Keys**.

    ![Create AWS IAM Role & Key](./05.png)

* In the popup window it will show you the ***Access key ID*** and ***Secret access key***. Save both of these values to configure the **Terraform plan** variables later.

    ![Create AWS IAM Role & Key](./06.png)

## Automation Flow

For you to get familiar with the automation and deployment flow, below is an explanation.

1. User is exporting the Terraform environment variables (1-time export) which are being used throughout the deployment.

2. User is executing the Terraform plan which will deploy the EC2 instance as well as:

    1. Create an Administrator Windows user account, enabling WinRM on the VM and change the Windows Computer Name.

    2. Generate and execute the [*sql.ps1*](https://github.com/microsoft/azure_arc/blob/master/azure_arc_sqlsrv_jumpstart/aws/winsrv/terraform/scripts/sql.ps1.tmpl) script. This script will:

        1. Install Azure CLI, Azure PowerShell module and SQL Server Management Studio (SSMS) [Chocolaty packages](https://chocolatey.org/).

        2. Create a runtime logon script (*LogonScript.ps1*) which will run upon the user first logon to Windows. Runtime script will:

            * Install SQL Server Developer Edition
            * Enable SQL TCP protocol on the default instance
            * Create SQL Server Management Studio Desktop shortcut
            * Restore [*AdventureWorksLT2019*](https://docs.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver15&tabs=ssms) Sample Database
            * Onboard both the server and SQL to Azure Arc
            * Deploy Azure Log Analytics and a workspace
            * Install the [Microsoft Monitoring Agent (MMA) agent](https://docs.microsoft.com/en-us/services-hub/health/mma-setup)
            * Enable Log Analytics Solutions
            * Deploy MMA Azure Extension ARM Template from within the VM
            * Configure SQL Azure Assessment

        3. Disable and prevent Windows Server Manager from running on startup

3. Once Terraform plan deployment has completed and upon the user initial RDP login to Windows, *LogonScript.ps1* script will run automatically and execute all the above.

## Deployment

Before executing the Terraform plan, you must set the environment variables which will be used by the plan. These variables are based on the Azure Service Principal you've just created, your Azure subscription and tenant, and your AWS account.

* Retrieve your Azure Subscription ID and tenant ID using the `az account list` command.

* The Terraform plan creates resources in both Microsoft Azure and AWS. It then executes a script on the virtual machine to install all the necessary artifacts.

    Both the script and the Terraform plan itself requires certain information about your AWS and Azure environments. Edit variables according to your environment and export it using the below commands

    ```console
    export TF_VAR_subId='Your Azure Subscription ID'
    export TF_VAR_servicePrincipalAppId='Your Azure Service Principal App ID'
    export TF_VAR_servicePrincipalSecret='Your Azure Service Principal App Password'
    export TF_VAR_servicePrincipalTenantId='Your Azure tenant ID'
    export TF_VAR_location='Azure Region'
    export TF_VAR_resourceGroup='Azure resource group Name'
    export TF_VAR_AWS_ACCESS_KEY_ID='Your AWS Access Key ID'
    export TF_VAR_AWS_SECRET_ACCESS_KEY='Your AWS Secret Key'
    export TF_VAR_key_name='Your AWS Key Pair name'
    export TF_VAR_aws_region='AWS region'
    export TF_VAR_aws_availabilityzone='AWS Availability Zone region'
    export TF_VAR_instance_type='EC2 instance type'
    export TF_VAR_hostname='EC2 instance Windows Computer Name'
    export TF_VAR_admin_user='Guest OS Admin Username'
    export TF_VAR_admin_password='Guest OS Admin Password'
    ```

    ![Export terraform variables](./07.png)

* From the folder within your cloned repo where the Terraform binaries are, the below commands to download the needed TF providers and to run the plan.

    ```console
    terraform init
    terraform apply --auto-approve
    ```

    Once the Terraform plan deployment has completed, a new Windows Server VM will be up & running as well as an empty Azure resource group will be created.

    ![terraform apply completed](./08.png)

    ![New AWS EC2 instance](./09.png)

    ![An empty Azure resource group](./10.png)

* Download the RDP file and log in to the VM (**using the data from the *TF_VAR_admin_user* and *TF_VAR_admin_password* environment variables**) which will initiate the *LogonScript* run. Let the script to run it's course and which will also close the PowerShell session when completed.

    ![Connect to AWS EC2 instance](./11.png)

    ![Connect to AWS EC2 instance](./12.png)

    > **Note: The script runtime will take ~10-15min to complete**

    ![PowerShell LogonScript run](./13.png)

    ![PowerShell LogonScript run](./14.png)

    ![PowerShell LogonScript run](./15.png)

    ![PowerShell LogonScript run](./16.png)

    ![PowerShell LogonScript run](./17.png)

    ![PowerShell LogonScript run](./18.png)

    ![PowerShell LogonScript run](./19.png)

    ![PowerShell LogonScript run](./20.png)

    ![PowerShell LogonScript run](./21.png)

* Open Microsoft SQL Server Management Studio (a Windows shortcut will be created for you) and validate the *AdventureWorksLT2019* sample database is deployed as well.

    ![Microsoft SQL Server Management Studio](./22.png)

    ![AdventureWorksLT2019 sample database ](./23.png)

* In the Azure Portal, notice you now have an Azure Arc enabled server resource (with the MMA agent installed via an Extension), Azure Arc enabled SQL server resource and Azure Log Analytics deployed.

    ![An Azure resource group with deployed resources](./24.png)

    ![Azure Arc enabled server resource](./25.png)

    ![MMA agent installed via an Extension](./26.png)

    ![Azure Arc enabled SQL server resources](./27.png)

## Azure SQL Assessment

Now that you have both the server and SQL projected as Azure Arc resources, the last step is complete the initiation of the SQL Assessment run.

* On the SQL Azure Arc resource, click on "Environment Health" followed by clicking the "Download configuration script".

    Since the *LogonScript* run in the deployment step took care of deploying and installing the required binaries, you can safely and delete the downloaded *AddSqlAssessment.ps1* file.

    Clicking the "Download configuration script" will simply send a REST API call to the Azure portal which will make "Step3" available and will result with a grayed-out "View SQL Assessment Results" button.

    ![SQL Assessment Environment Health](./28.png)

    ![SQL Assessment Environment Health](./29.png)

    ![View SQL Assessment Results](./30.png)

* After few minutes you will notice how the "View SQL Assessment Results" button is available for you to click on. At this point, the SQL assessment data and logs are getting injected to Azure Log Analytics.

    Initially, the amount of data will be limited as it take a while for the assessment to complete a full cycle but after few hours you should be able to see much more data coming in.  

    ![SQL Assessment results](./31.png)

    ![SQL Assessment results](./32.png)

## Cleanup

To delete the environment, use the *`terraform destroy --auto-approve`* command which will delete the AWS and the Azure resources.

![terraform destroy completed](./33.png)