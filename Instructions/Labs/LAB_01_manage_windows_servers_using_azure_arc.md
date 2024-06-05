---
lab:
    title: 'Lab: Manage on-premises Windows servers by using Azure Arc'
    module: 'Module: Manage on-premises Windows servers by using Azure Arc'
---

# Lab: Manage on-premises Windows servers by using Azure Arc
# Student lab manual

## Lab scenario

Your organization has a hybrid environment consisting of servers hosted in its own datacenters and multiple Azure subscription. The organization currently struggles to maintain consistent management, maintenance, and monitoring approach across the on-premises and cloud based resources. To streamline the operational model and minimize administrative overhead, you plan to implement Azure Arc. By using Azure Arc, you plan to enforce organizational standards with Azure Policy, protect your servers with Microsoft Defender for Cloud, monitor performance and health using Azure Monitor, and manage updates through Azure Update Manager. Additionally, you intend to leverage Azure VM extensions and the azcmagent CLI to ensure consistent configuration across all servers.

## Objectives

After completing this lab, you will be able to:

- [Onboard on-premises Windows servers to Azure Arc](https://github.com/MicrosoftLearning/Deploy-and-manage-Azure-Arc-enabled-Servers/blob/master/Instructions/Labs/LAB_01_manage_windows_servers_using_azure_arc.md#exercise-1-onboard-windows-servers-to-azure-arc) 
- Manage on-premises Windows servers by using Azure Arc and Azure Policy
- Protect on-premises Windows servers by using Microsoft Defender for Cloud
- Monitor on-premises Windows servers by using Azure Monitor
- Update on-premises Windows servers by using Azure Update Manager
- Configure on-premises Windows servers by using Azure VM extensions and azcmagent CLI

## Lab duration

Total estimated time: 230 minutes

## Requirements

- A Microsoft Azure subscription with the sufficient number of available of vCPUs to accommodate deployment of two Azure virtual machines (VMs) in the Azure region you intend to use for this lab.
- A work/school account or Microsoft account that has the owner role in the Azure subscription you will be using for this lab. This account should also be a Global Administrator in the Microsoft Entra tenant associated with this subscription. 
- A lab computer with an Azure Cloud Shell-compatible web browser and access to Microsoft cloud.

## Exercises

The lab consists of the following exercises:

- Onboard Windows servers to Azure Arc
- Manage Azure Arc-enabled Windows servers by using Azure Policy
- Enhance security of Azure Arc-enabled Windows servers by using Microsoft Defender for Cloud
- Monitor Azure Arc-enabled Windows servers by using Azure Monitor
- Manage updates of Azure Arc-enabled Windows servers by using Azure Update Manager

### Exercise 1: Onboard Windows servers to Azure Arc

Estimated Time: 60 minutes

In this exercise, you will:

- Deploy Azure resources by using an Azure Resource Manager template
- Implement the operating system prerequisites for connecting Azure VMs to Azure Arc
- Prepare for connecting on-premises servers to Azure Arc
- Connect a Windows Server to Azure Arc by using Windows Admin Center 
- Connect Windows servers to Azure Arc non-interactively at scale

> **Note:** There are other methods of connecting Windows Server resources to Azure Arc in addition to these presented here. For their comprehensive listing, refer to [Azure Connected Machine agent deployment options](https://learn.microsoft.com/en-us/azure/azure-arc/servers/deployment-options).

> **Note:** For practical reasons, throughout all of the labs, you will use Azure VMs to emulate on-premises Windows servers. In general, you wouldn't connect Azure VMs to Azure Arc because natively they already include the same capabilities (such as support of Azure Resource Manager, VM extensions, managed identities, and Azure Policy). As a matter of fact, an attempt to install Azure Arc on an Azure VM will result in an error message indicating that such configuration is unsupported. However, it is possible to install Azure Arc on Azure VMs for evaluation and testing purposes. For more details regarding this topic, refer to the Microsoft Learn article [Evaluate Azure Arc-enabled servers on an Azure virtual machine](https://learn.microsoft.com/en-us/azure/azure-arc/servers/plan-evaluate-on-azure-virtual-machine).

> **Note:** It's worth noting that, while our focus is on Windows Server, the scope of Azure Arc extends to Linux servers, Kubernetes clusters, Azure data services, SQL Server, and virtualized platforms such as Azure Stack HCI and VMware vSphere.

#### Task 1: Deploy Azure resources by using an Azure Resource Manager template

In this task, you will deploy Azure compute resources that will be used to emulate Windows servers in an on-premises environment. The deployment will provision two Azure VMs named **arclab-winvm0** and **arclab-winvm1** running Windows Server 2022.

1. From the lab computer, start a Web browser, and navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com).
1. If prompted, sign in with either a work/school account or Microsoft account that has the owner role in the Azure subscription you will be using for this lab.
1. In the Azure Portal, start a PowerShell session in Cloud Shell. 

    > **Note**: If prompted, select **PowerShell**, then, in the **Getting started** pane, select **No storage account required**, in the **Subscription** drop-down list, select the Azure subscription you are using for this lab, and select **Apply**. 

1. In the PowerShell session in the Cloud Shell pane, run the following commands to create a shallow clone of the repository hosting the Azure Resource Manager (ARM) template you will use for the deployment and set the current directory to the location of that template and its parameter file:

    ```
    Set-Location -Path $HOME
    Remove-Item -Path ./Deploy-and-manage-Azure-Arc-enabled-Servers -Recurse -Force
    git clone --depth 1 https://github.com/MicrosoftLearning/Deploy-and-manage-Azure-Arc-enabled-Servers.git
    Set-Location -Path ./Deploy-and-manage-Azure-Arc-enabled-Servers/Allfiles/Templates
    ```

1. In the Cloud Shell pane, run the following command to set the value of the variable `$rgName` designating the target resource group name to `arclab-infra-RG`:

    ```powershell
    $rgName = 'arclab-infra-RG'
    ```

1. Run the following command to set the value of the variable `$location` to the name of the Azure regions to which you intend to deploy the lab resources (replace the `<Azure_region>` placeholder with the name of that region):

    ```powershell
    $location = '<Azure_region>'
    ```

1. Run the following command to create a resource group named **arclab-infra-RG** in the Azure region you chose:

    ```powershell
    New-AzResourceGroup -Name $rgName -Location $location -Force
    ```

1. Run the following command to set a unique value of the variable `$deploymentName`:

    ```powershell
    $deploymentName = 'arclabinfra' + $(Get-Date -Format 'yyyy-MM-dd-hh-mm')
    ```

1. Run the following command to set the name of the password of the local administrator (replace the `<Password>` placeholder with a password of your choice that you will use to sign in to the Azure VMs that are included in the deployment):

    > **Note**: Ensure that the password has at least 12 characters, includes a mix of lower case characters, upper case characters, one or more digits, and at least one special character

    ```powershell
    $adminPassword = ConvertTo-SecureString '<Password>' -AsPlainText -Force
    ```

1. Run the following command to set the value of the variable `$deployBastion` to either numeric 1 (`true`) or 0 (`false`), depending on whether you intend to use it or not to connect to Azure VMs via Remote Desktop:

    > **Note**: Using Azure Bastion is recommended, since this increases security by not exposing Azure VMs directly via public IP addresses assigned to their interfaces. However, note that Azure Bastion introduces extra [cost](https://azure.microsoft.com/en-us/pricing/details/azure-bastion/) and increases deployment time. 

    ```powershell
    [bool] $deployBastion = <1/0>
    ```

1. Run the following command to initiate the deployment:

    > **Note**: Replace the `<Password>` placeholder with a password of your choice that you will use to sign in to the Azure VMs that are included in the deployment. Ensure that the password has at least 12 characters, includes a mix of lower case characters, upper case characters, one or more digits, and at least one special character. Set the value of the `deployBastion` parameter to either `true` or `false`, depending on whether you want to use Azure Bastion to connect to Azure VMs via Remote Desktop):

    ```powershell
    New-AzResourceGroupDeployment -Name $deploymentName `
                                  -ResourceGroupName $rgName `
                                  -TemplateFile ./azuredeploylab.json `
                                  -TemplateParameterFile ./azuredeploy.parameters.json `
                                  -adminPassword $adminPassword -deployBastion $deployBastion
    ```

    > **Note**: The deployment should take about 1 minute without Azure Bastion and about 10 minutes if you decide to include it. Wait for the deployment to complete before you proceed to the next task.

#### Task 2: Implement the operating system prerequisites for connecting Azure VMs to Azure Arc

In this task, you will implement the operating system prerequisites for onboarding Azure VMs to Azure Arc are satisfied.

> **Note**: It is important to keep in mind that these prerequisites are applicable strictly to this particular lab scenario, in which Azure VMs are used to simulate on-premises servers. You do NOT need to implement such prerequisites in production scenarios that involve onboarding non-Azure VMs.

> **Note**: In order for Azure VMs to support installation of the Connected Machine agent used by Azure Arc, it is necessary to implement four tasks. The first one comprises creating a system environment variable named **MSFT_ARC_TEST** with the value of **true**. The second one consists of removing all Azure VM extensions. In the third one, you need to stop and disable the Azure VM Guest Agent service. Finally, it is also necessary to block access to the Azure Instance Metadata Service (IMDS) endpoint, which can be accomplished by using Windows Defender Firewall with Advanced Security. 

1. From the lab computer, in the web browser window displaying the Azure Portal, in the **Search** text box at the top of the page, search for and select **Virtual machines**.
1. On the **Virtual machines** page, select the **arclab-winvm0** entry. 
1. On the **arclab-winvm0** page, in the vertical menu on the left side, in the **Settings** section, select **Extensions + applications**. 
1. If any extensions are listed on the **Extensions** tab, select each of them and, in the extension's detail pane, select **Uninstall**.

    > **Note**: Wait for the uninstallation of each extension to complete before proceeding to the next one.

1. On the **arclab-winvm0** page, use the **Connect** menu option to connect to the Windows Server.

    > **Note**: Use the **Connect** or **Connect via Bastion** option, depending on whether you included Azure Bastion in your deployment in the previous task.

1. When prompted to authenticate, provide the **arclabadmin** username and the password you specified during the deployment.
1. Within the Remote Desktop session to **arclab-winvm0**, from the Start menu, start **Windows PowerShell ISE**. 
1. From the console pane of the **Administrator: Windows PowerShell ISE** window, run the following command to create the system environment variable **MSFT_ARC_TEST** and set its value to **true**:

    ```powershell
    [System.Environment]::SetEnvironmentVariable("MSFT_ARC_TEST",'true', [System.EnvironmentVariableTarget]::Machine)
    ```

1. From the console pane of the **Administrator: Windows PowerShell ISE** window, run the following command to stop and disable the status of the Azure VM Guest Agent:

    ```powershell
    Set-Service WindowsAzureGuestAgent -StartupType Disabled -Verbose
    Stop-Service WindowsAzureGuestAgent -Force -Verbose
    ```

1. From the console pane of the **Administrator: Windows PowerShell ISE** window, run the following command to create a Windows Defender Firewall with Advanced Security rule that blocks outbound access to the Azure IMDS endpoint:

    ```powershell
    New-NetFirewallRule -Name BlockAzureIMDS -DisplayName "Block access to Azure IMDS" -Enabled True -Profile Any -Direction Outbound -Action Block -RemoteAddress 169.254.169.254 
    ```

> **Note**: Repeat all steps in this task to connect to and implement the Azure Arc prerequisites in **arclab-winvm1**.

#### Task 3: Prepare for connecting on-premises servers to Azure Arc

In this task, you will prepare your lab environment for connecting on-premises resources to Azure Arc. This involves registering the following resource providers in the target Azure subscription:

- Microsoft.HybridCompute
- Microsoft.GuestConfiguration
- Microsoft.HybridConnectivity
- Microsoft.Compute.

1. If needed, switch to the Remote Desktop session to **arclab-winvm0**.
1. Within the Remote Desktop session to **arclab-winvm0**, start a Web browser, and navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com).
1. When prompted, sign in with either a work/school account or Microsoft account that has the owner or contributor role in the Azure subscription you are using for this lab.
1. In the Azure Portal, start a PowerShell session in Cloud Shell. 

    > **Note**: When prompted, select **PowerShell**, then, in the **Getting started** pane, select **No storage account required**, in the **Subscription** drop-down list, select the Azure subscription you are using for this lab, and select **Apply**. 

1. In the PowerShell session within the Cloud Shell pane, run the following commands to register the Azure resource providers required to implement Azure Arc-enabled servers:

    ```powershell
    Register-AzResourceProvider -ProviderNamespace Microsoft.HybridCompute
    Register-AzResourceProvider -ProviderNamespace Microsoft.GuestConfiguration
    Register-AzResourceProvider -ProviderNamespace Microsoft.HybridConnectivity
    Register-AzResourceProvider -ProviderNamespace Microsoft.AzureArcData
    ```

    > **Note:** Do not wait for the registration to complete but instead proceed to the next step. The registration should complete within the next few minutes. 

#### Task 4: Connect a Windows Server to Azure Arc by using Windows Admin Center

In this task, you will install Windows Admin Center, register it with Azure, and use it to connect **arclab-winvm0** to Azure Arc.

1. Within the Remote Desktop session to **arclab-winvm0**, switch to the **Administrator: Windows PowerShell ISE** window and open the script pane.
1. In the Windows PowerShell ISE script pane, enter the following script that installs Windows Admin Center and select the green arrow icon in the toolbar to execute it:

    ```powershell
    Invoke-WebRequest 'https://aka.ms/WACDownload' -OutFile "$pwd\WAC.msi"
    $msiArgs = @("/i", "$pwd\WAC.msi", "/qn", "/L*v", "log.txt", "SME_PORT=443", "SSL_CERTIFICATE_OPTION=generate")
    Start-Process msiexec.exe -Wait -ArgumentList $msiArgs
    ```

    > **Note:** Wait for the installation of Windows Admin Center to complete. The installation should take about 3 minutes. It provisions the Windows Admin Center gateway component accessible via [https://localhost](https://localhost), secured by a self-signed certificate valid for 60 days.

    > **Note:** Next, you need to register Windows Admin Center with Azure.

1. Within the Remote Desktop session to **arclab-winvm0**, start a web browser and navigate to the [https://localhost](https://localhost) page.

    > **Note:** Ensure that you use the **localhost** name, rather than the actual server name.

1. When presented with the warning **Your connection isn't private**, select **Advanced** and then select **Continue to localhost (unsafe)**.

    > **Note:** The warning is expected since the target site is using a self-signed certificate.

1. If prompted to authenticate, sign in as **arclabadmin**.
1. Close the pane confirming the successful installation, wait for the updates of the Windows Admin Center extensions to complete, and acknowledge their completion.
1. In Windows Admin Center, on the **All connections** page, select the cogwheel icon in the upper right corner of the page.
1. On the **Settings \| Account** page, in the **Azure Account** section, select **Register with Azure** and then, on the **Register with Azure** pane, select **Register**.
1. In the **Get started with Azure in Windows Admin Center** pane, in step 2 of the registration process, select **Copy** to copy the registration code into Clipboard.
1. Select the link next to the **Enter the code** text in step 3 of the registration process.
 
    > **Note:** This will open another tab in the Microsoft Edge window displaying the Enter code page.

1. In the **Enter code** pane, paste the code you copied into Clipboard and select **Next**.
1. If prompted to sign in, use the same credentials you used earlier to access the Azure subscription.
1. When prompted to confirm the question **Are you trying to sign in to Windows Admin Center?**, select **Continue**.
1. Verify that the sign in was successful and close the newly opened tab of the web browser window.
1. Back on the **Get started with Azure in Windows Admin Center** pane, in step 4 of the registration process, select **Create new** and then select **Connect**.

    > **Note:** This will generate a service principal in the Microsoft Entra tenant associated with the Azure subscription you are using in this lab.

1. In the **Get started with Azure in Windows Admin Center** pane, in step 5 of the registration process, select **Sign in**.
1. If prompted to sign in, provide the same Microsoft Entra ID credentials you have been using so far in the lab to authenticate access to the Azure subscription.
1. If prompted, in the **Permissions requested** pop-up window, review the permissions required by the application, select the checkbox **Consent on behalf of your organization** and select **Accept**.
1. On the **Register with Azure** pane of the Windows Admin Center page, verify that the registration was successful.

    > **Note:** Next, you will use Windows Admin Center to connect the Windows Server to Azure Arc.

1. In the web browser window displaying Windows Admin Center, select the **Settings** blue header, in the drop-down list, select **All connections**, and, in the list of connections, select the ****arclab-winvm0** (Gateway)** entry.
1. In the Windows Admin Center interface, on the **arclab-winvm0** page, in the navigation menu on the left side, select **Azure hybrid center**.
1. In the **Azure hybrid center** pane, on the **Available Services** tab, in the **Set up Azure Arc** section, select **Set up**.
1. In the **Set up Azure Arc** pane, perform the following actions:

    1. Ensure that the **Subscription** drop-down list displays the name of the Azure subscription you are using in this lab.
    1. In the **Resource group** section, ensure that the **Create new** option is selected and then enter **arclab-servers-RG**.
    1. In the **Azure region** drop-down list, select the name of the same Azure region you chose to deploy the Azure VMs earlier in this lab. 

    > **Note:** This Azure region will store metadata of your Azure Arc-enabled servers.

1. Select **Set up** to proceed with configuring **arclab-winvm0** as an Azure Arc-enabled server.

    > **Note:** The server will connect to Azure, download the Connected Machine agent, install it and register with Azure Arc. To track the progress, select the **Notifications** (a bell-shaped) icon in the upper right corner of the Windows Admin Center toolbar. The installation will trigger a display of a Command Prompt window, providing execution context for the **azcmagent.exe** installer of the Connected Machine agent. Wait for the installation to complete. This might take about 2 minutes. 

1. To validate the successful outcome, switch to the web browser window displaying the Azure portal, in the **Search** text box, search for and select **Azure Arc**.
1. On the **Azure Arc** page, in the navigation menu on the left side, in the **Azure Arc resources** section, select **Machines**.
1. On the **Azure Arc \| Machines ** page, verify that the entry **arclab-winvm0** appears in the list of Azure Arc-enabled machines with the Arc agent status of **Connected**.

#### Task 5: Connect Windows servers to Azure Arc non-interactively at scale

In this task, you will connect a Windows Server to Azure Arc by using a service principal, which illustrates one of the non-interactive methods for connecting servers to Azure Arc that facilitate at scale scenarios. This service principal can be used instead of your identity (which was used in the previous task) to authorize access to the target Azure subscription. You will implement this method by using the second Azure VM you deployed earlier in this lab named **arclab-winvm1**.

> **Note:**  The process of connecting to Azure Arc is automated by using a template script available from the Azure portal. You will generate it first.

1. If needed, switch to the Remote Desktop session to **arclab-winvm1**.
1. Within the Remote Desktop session to **arclab-winvm1**, start a Web browser, and navigate to the Azure portal at [https://portal.azure.com](https://portal.azure.com).
1. When prompted, sign in with either a work/school account or Microsoft account that has the owner role in the Azure subscription you are using for this lab.
1. In the Azure Portal, in the **Search** text box, search for and select **Azure Arc**.
1. On the **Azure Arc** page, in the navigation menu on the left side, in the **Azure Arc resources** section, select **Machines**.
1. On the **Azure Arc \| Machines** page, select **Add/Create** and then select **Add a machine** from the drop-down menu.
1. On the **Add servers with Azure Arc** page, in the **Add multiple servers** tile, select **Generate script**.
1. On the **Basics** tab of the **Add multiple servers with Azure Arc** page, specify the following settings:

    |Setting|Value|
    |---|---|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Resource group|**arclab-servers-RG**|
    |Region|The name of the same Azure region you selected in the previous task|
    |Operating system|**Windows**|
    |Connectivity method|**Public endpoint**|

1. In the **Authentication** section, under the **Service principal** drop-down list, select **Create new**. 
1. In the **New Azure Arc service principal** pane, specify the following settings and then select **Create**:

    |Setting|Value|
    |---|---|
    |Name|**arclab-servers-sp**|
    |Scope assignment level|**Resource group**|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Resource group|**arclab-servers-RG**|
    |Client secret description|**arclab-servers-sp-secret**|
    |Expires|**1 week**|
    |Role assignment roles|**Azure Connected Machine Onboarding**|

    > **Note:** Adjust the expiration interval based on your preferences.

    > **Note:** The Azure Connected Machine Onboarding role is available for at-scale onboarding. It is able to only read or create new Azure Arc-enabled servers in Azure. It cannot be used to delete servers already registered or manage extensions. As a best practice, you should consider assigning this role only to the Microsoft Entra service principal used to onboard machines at scale. The Azure Connected Machine Resource Administrator role grants permissions to read, modify, reonboard, and delete a machine. This role is designed to support management of Azure Arc-enabled servers, but not other resources in the resource group or subscription.

1. When presented with the **Download service principal ID and secret** dialog box, select **Download and close**.

   > **Important:** You will not be able to retrieve the client secret again after you close this dialog box.

1. In the browser window, in the notification area, select **Open file** to open **servicePrincipal.txt** file containing the id and secret values of the newly created service principal.
1. Back on the **Basics** tab of the **Add multiple servers with Azure Arc** page, select **Download and run script**.
1. On the **Download and run script** tab of the **Add multiple servers with Azure Arc** page, ensure that the **Deployment method** option is set to **Basic** and in the section labeled **2. Download the script and add service principal credentials**, select **Download**.
1. If prompted, in the upper right corner of the web browser window, select **Keep** to continue with the download.

    > **Note:** For Windows, the script is named **OnboardingScript.ps1**.

    > **Note:** The script establishes a connection to Azure Arc by using the **azcmagent** command and referencing the service principal you created. To successfully execute the command, you need to replace the `<ServicePrincipalId>` and `<ENTER SECRET HERE>` placeholders in line 3 and 4 (respectively) of the script with the values of the service principal id and secret, which you can find in the downloaded **servicePrincipal.txt** file.
1. Switch to the **Administrator: Windows PowerShell ISE** window and open the downloaded **OnboardingScript.ps1** file.
1. In the script pane of the **Administrator: Windows PowerShell ISE** window, replace the `<ServicePrincipalId>` and `<ENTER SECRET HERE>` placeholders in line 3 and 4 (respectively) of the script with the values of the service principal id and secret included in the downloaded **servicePrincipal.txt** file.
1. In **Administrator: Windows PowerShell ISE**, save the changes to the **OnboardingScript.ps1** file.
1. From the console pane of the **Administrator: Windows PowerShell ISE** window, run the following command to allow running the downloaded script in environments where the RemoteSigned PowerShell execution policy is in place:

    ```powershell
    Unblock-File -Path $env:USERPROFILE\DOWNLOADS\OnboardingScript.ps1
    ```

1. Open another tab in the script pane of the **Administrator: Windows PowerShell ISE** window and use it to run the following script to execute the script on the local server and, as the result, connect it to Azure Arc:

    ```powershell
    $scriptName = 'OnboardingScript.ps1'
    $scriptPath = "$env:USERPROFILE\Downloads\$scriptName"
    $remoteDirectoryName = 'Temp'
    $servers = 'arclab-winvm1'
 
    $servers | ForEach-Object {New-Item -ItemType Directory -Path (Join-Path "\\$_\c`$\" "$remoteDirectoryName") -Force}
    $servers | ForEach-Object {Copy-Item -Path $scriptPath -Destination (Join-Path "\\$_\c`$\" "$remoteDirectoryName")}
 
    Invoke-Command -ComputerName $servers -ScriptBlock {
       param($directory, $file)
       & $("c:\$directory\$file")
    } -ArgumentList ($remoteDirectoryName, $scriptName)
    ```

    > **Note:** The script creates the C:\Temp directory on the target server, copies to it the script, and uses PowerShell Remoting to launch the script execution.

    > **Note:** Obviously, in this case, you could just run the script locally on the server. The method you used is meant to illustrate the approach you could apply to any number of servers, simply by appending their names to the value of the `$servers` variable.

    > **Note:** Wait for the script to complete. This might take about 3 minutes.

1. As in the previous task, to validate the successful outcome, in the web browser window displaying the Azure portal, navigate to the **Azure Arc \| Machines** page and verify that the entry for **arclab-winvm1** appears in the list of Azure Arc-enabled machines with the Arc agent status of **Connected**.

### Exercise 2: Manage Azure Arc-enabled Windows servers by using Azure Policy

Estimated Time: 25 minutes

In this exercise, you will:

- Create a policy assignment targeting Azure Arc-enabled Windows servers
- Evaluate results of the policy assignment

> **Note:** Azure Policy supports a wide range of scenarios that deal with compliance assessment of Arc-enabled Windows Severs. For an example of such scenario, refer to [Tutorial: Create a policy assignment to identify non-compliant resources](https://learn.microsoft.com/en-us/azure/azure-arc/servers/learn/tutorial-assign-policy-portal).

#### Task 1: Create a policy assignment targeting Azure Arc-enabled Windows servers

In this task, you will assign a built-in policy to the resource group containing your Arc-enabled Windows servers you deployed in the previous exercise of this lab. The policy will result in automatic installation of the Azure Monitor agent, which you will facilitate implementing the monitoring functionality of Arc-enabled machines later in this lab.

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, in the **Search** text box, search for and select **Policy**.
1. On the **Policy** page, in the vertical navigation menu on the left side, in the **Authoring** section, select **Definitions**.
1. On the **Policy \| Definitions** page, in the **Search box, enter **Configure Windows Arc-enabled machines** and, in the list of results, select **Configure Windows Arc-enabled machines to run Azure Monitor Agent**.
1. On the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy definition page, review the policy definition and available effects and then select **Assign policy**.
1. On the **Basics** tab of the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy assignment page, on the right side of the **Scope** text box, select the ellipsis button, in the **Scope** pane, in the **Subscription** drop-down list, select the Azure subscription you are using in this lab, in the **Resource Group** drop-down list, select **arclab-servers-RG**, followed by the **Select** button.
1. Back on the **Basics** tab of the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy assignment page, note that **Policy enforcement** is enabled by default and select **Next**.
1. On the **Advanced** tab, review the available options without modifying them and then select **Next**.
1. On the **Parameters** tab, select **Next**, since this policy definition does not include any parameters that need input or review.
1. On the **Remediation** tab, select the **Create a remediation task** checkbox, ensure that **System assigned managed identity** option is selected, and then select **Next**.

    > **Note:** This is necessary in order to apply the assignment to the existing resources. In general, a policy settings will be enforced on resources created after the assignment is created. However, policies with the **deployIfNotExist** and **Modify** effects apply to existing resources if remediation task is configured.

1. On the **Non-compliance message** tab, select **Next**.
1. On the **Review + create** tab, select **Create**.

#### Task 2: Evaluate policy results

In this task, you will review the results of the policy assignment.

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, navigate back to the **Policy \| Definitions** page and, in the vertical menu on the left side, select **Compliance**.
1. On the **Policy \| Compliance** page, in the list of policy assignments, locate the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** entry. Most likely, at this point, the policy will likely list the **Compliance** status for both resources targeted by the policy assignment as *Not started** or **Non-compliant**.

    > **Note:** If that's the case, wait until the compliance status changes to **Non-compliant**. This could take about 3 minutes. You might need to refresh the page to display the newly created policy assignment and its updated status.

1. On the **Policy \| Compliance** page, select the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** entry.
1. On the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy compliance page, review the detailed listing of non-compliant resources and note the **Create remediation task** button in the toolbar.

    > **Note:** You could use this functionality to explicitly invoke the remediation task. In our case, this is not required since the remediation task should already be running. 

1. To verify whether the remediation task is running, navigate back to the **Policy \| Compliance** page, select **Remediation** and, on the **Policy \| Remediation** page, select the **Remediation tasks** tab.
1. In the list remediation tasks, note the existing entry representing the remediation task associated with the newly created policy assignment.

    > **Note:** The remediation task should transition from the **Evaluating** state through **In Progress** to **Complete**. This might take about 15 minutes. Do not wait for the task to complete but instead proceed to the next task. Consider revisiting this task later on.

    > **Note:** Alternatively, to verify the status of the remediation task, you can navigate back to the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy compliance page, select the **Activity Logs** button in the toolbar, and review the log entries. You can use the log to track progress of the remediation activities. At some point, the log should display the entries referencing invocation of the **deployIfNotExists** policy action and installation of the Azure Monitor Agent extension.

1. Once the remediation task completes, navigate back to the **Configure Windows Arc-enabled machines to run Azure Monitor Agent** policy compliance page and verify that all four resources are listed as compliant.

    > **Note:** You might need to refresh the page to display the updated compliance status.

1. To validate the results in another way, in the Azure portal, navigate back to the **Azure Arc \| Machines** page and select the **arclab-winvm1** entry.
1. On the **arclab-winvm1** page, in the vertical navigation menu on the left side, select **Extensions** and verify that the **AzureMonitorWindowsAgent** appears on the list of installed extensions.

### Exercise 3: Enhance security of Azure Arc-enabled Windows servers by using Microsoft Defender for Cloud

Estimated Time: 45 minutes

In this exercise, you will:

- Configure Microsoft Defender for Cloud-based protection of Azure Arc-enabled Windows servers 
- Review the Microsoft Defender for Cloud-based protection of Azure Arc-enabled Windows servers 

#### Task 1: Configure Microsoft Defender for Cloud-based protection of Azure Arc-enabled Windows servers 

In this task, you will enable protection of Arc-enabled Windows servers by Microsoft Defender for Cloud. 

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, in the **Search** text box, search for and select **Microsoft Defender for Cloud**.
1. On the **Microsoft Defender for Cloud \| Overview** page, in the navigation menu on the left side, select **Getting started**.
1. On the **Microsoft Defender for Cloud \| Getting started** page, on the **Upgrade** tab, in the **Enable Defender for Cloud on 1 subscription** section, ensure that the checkbox next to the Azure subscription entry is selected and then select **Upgrade**.

    > **Note:** This will initiate a 30-day trial of Microsoft Defender for Cloud.

1. If your session gets redirected to the **Install agents** tab, **DO NOT** select **Install agents**.

    > **Note:** Microsoft Defender for Cloud does not rely on Azure Monitor Agent or Log Analytics Agent (its retirement date was set for August 2024) for the Defender for Servers offering (with the exception of Defender for SQL Server on machines), which includes protection of Azure Arc-enabled servers. For more information, refer to [Azure Monitor Agent in Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/auto-deploy-azure-monitoring-agent).

1. On the **Microsoft Defender for Cloud \| Getting started** page, in the navigation menu on the left side, in the **Management** section, select **Environment settings**.
1. On the **Microsoft Defender for Cloud \| Environment settings** page, select the entry representing the Azure subscription you are using in this lab.
1. On the **Settings \| Defender plans** page, if prompted to try **Auto-provisioning updated experience**, select **Try it now**. This will redirect your browser session to the **Settings & monitoring** page.
1. On the **Settings & monitoring** page, review the default settings.

    > **Note:** **Log Analytics agent** component should be, by default, disabled. Similarly, **Guest Configuration agent (preview)** is by default disabled, however, in case of servers connected to Azure Arc, this component is already included in the Connected Machine agent.

1. Navigate back to the **Settings \| Defender plans** page and review both the **Cloud Security Posture Management (CSPM)** and **Cloud Workload Protection** plans.

#### Task 2: Review the Microsoft Defender for Cloud-based protection of Azure Arc-enabled Windows servers 

In this task, you will review the Microsoft Defender for Cloud-based protection capabilities available to Azure Arc-enabled Windows servers.

> **Note:** Considering that the lab relies on Azure VMs to simulate on-premises Windows servers, it might be somewhat challenging to analyze the functionality of Microsoft Defender for Cloud in the context of Azure Arc, since both the Azure VM (simulating the on-premises Windows server) and the corresponding Azure Arc objects (implemented as the same Azure VMs) appear as resources on several Microsoft Defender for Cloud pages. You can distinguish between the two resource types based on a number of clues, including different icons used to display them in the Azure portal. To identify these icons, compare resources listed on the **Virtual machines** page and those listed on the **Azure Arc \| Machines** page.

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, navigate back to the **Microsoft Defender for Cloud \| Overview** page.
1. On the **Microsoft Defender for Cloud \| Overview** page, in the navigation menu on the left side, in the **General** section, select **Inventory**.
1. On the **Microsoft Defender for Cloud \| Inventory** page, review the list of resources and note that it includes entries for **arclab-winvm0**  and **arclab-winvm1** virtual machines and matching entries with the value **Machines - Azure Arc** in the **Resource type** column. 

    > **Note:** Review the values in the **Defender for Cloud** column on the **Microsoft Defender for Cloud \| Inventory** page and verify that the Azure Arc servers are onboarded to Microsoft Defender for Cloud.

1. On the **Microsoft Defender for Cloud \| Overview** page, in the navigation menu on the left side, in the **General** section, select **Recommendations**.
1. On the **Microsoft Defender for Cloud \| Recommendations** page, try to locate any recommendations applicable to either **arclab-winvm0** or **arclab-winvm1** Azure Arc-enabled Windows server (not the Azure VM).

    > **Note:** You might be able to locate the recommendation titled **Windows Defender Exploit Guard should be enabled on machines**.

1. Assuming that you found a recommendation applicable to one of the two Azure Arc-enabled Windows servers, select it.
1. On the corresponding remediation page, review the **Take action** tab, including remediation steps. The steps in some cases include a button labeled **Fix**, which launches **Quick fix** remediation wizard.

    > **Note:** Applying remediation steps is optional. After the remediation process completes, it may take up to 24 hours before the outcome is reflected in the Azure portal.

### Exercise 4: Monitor Azure Arc-enabled Windows servers by using Azure Monitor

Estimated Time: 30 minutes

In this exercise, you will:

- Configure VM Insights for Azure Arc-enabled Windows servers
- Review the monitoring capabilities of Azure Arc-enabled Windows servers

#### Task 1: Configure VM Insights for Azure Arc-enabled Windows servers

In this task, you will configure VM Insights for one of the Windows servers you onboarded to Azure Arc earlier in this lab. Note that you could alternatively manage this configuration by using Azure Policy for multiple servers, similarly to how you deployed the Azure Monitor agent in one of the previous exercises.

> **Note:**  To start, you will create an Azure Log Analytics workspace dedicated to the use of telemetry generated by Arc-enabled Windows servers.

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, in the **Search** text box, search for and select **Log Analytics workspaces**.
1. On the **Log Analytics workspaces** page, select **+ Create**.
1. On the **Basics** tab of the **Create Log Analytics workspace** page, specify the following settings and then select **Review + Create**:

    |Setting|Value|
    |---|---|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Resource group|**arclab-servers-RG**|
    |Name|**arclab-lamonitoringworkspace**|
    |Region|The name of the same Azure region you used in the previous exercises of this lab|

1. On the **Review + Create** tab, wait for the validation to complete and then select **Create**.

    > **Note:** Wait for the provisioning process to complete. This should take about 1 minute. 

1. While signed to the server arclab-winvm1, in the web browser window displaying the Azure portal, navigate to the **Azure Arc \| Machines** page and select the **arclab-winvm1** entry.
1. On the **arclab-winvm1** page, in the vertical navigation menu on the left side, in the **Monitoring** section, select **Insights**.
1. On the **arclab-winvm1 \| Insights** page, select **Enable**. 
1. In the **Monitoring configuration** pane, below the **Data collection rule** drop-down list, select **Create New**.

    > **Note:** You will create a new data collection rule that includes processes and dependencies in order to enable the Map functionality.

1. In the **Create new rule** pane, specify the following settings:

    |Setting|Value|
    |---|---|
    |Data collection rule name|**arclab-datamap-DCR**|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Enable processes and dependencies (Map)|enabled|
    |Log Analytics workspaces|**arclab-lamonitoringworkspace**|

1. Select **Create**.
1. Back in the **Monitoring configuration** pane, select **Configure**.

    > **Note:** Do not wait for the completion of monitoring configuration (you can use the notification area accessible via the bell icon in the toolbar of the Azure portal page to track the progress) but instead proceed to the next task. The monitoring configuration process might take about 10 minutes to complete.

#### Task 2: Review monitoring capabilities of Azure Arc-enabled Windows servers

In this task, you will review the resulting monitoring capabilities of Azure Arc-enabled Windows servers provided by VM Insights.

> **Note:**  The completion of the monitoring configuration might take several minutes. While you won't likely be able to view map data during that time, you should be able to access performance data. To determine whether this is the case, refresh the web browser window displaying the **arclab-winvm1 \| Insights** page every minute or so until the **Enable** button no longer appears and then select the **Performance tab**.

> **Note:**  To minimize the idle time, consider switching back to the exercise which involved implementing Azure Policy remediation task and then returning back to this exercise.

1. While on the **arclab-winvm1 \| Insights** page, refresh the web browser page to display its updated interface that includes **Get started**, **Performance**, and **Map** tabs.
1. On the **arclab-winvm1 \| Insights** page, select the **Performance** tab and review the charts displaying CPU, memory, disk, and network telemetry.
1. On the **arclab-winvm1 \| Insights** page, select the **Map** tab. This interface provides data about processes running on the monitored server, along with their incoming and outgoing connections. 

    > **Note:** The **Map** functionality should become available within 10-15 minutes following enabling VM Insights. 

1. Expand the list of processes for the monitored server. Select one of the processes to review its details and dependencies.
1. Select **arclab-winvm1** to display the server properties and, in the properties pane, select **Log Events**. This will display a table summarizing event types and their corresponding counts.
1. To view actual logged events, select any of the event type entries. You will be redirected to the Log Analytics workspace where the events are collected. From this interface, you can examine individual log entries in detail. 

### Exercise 5: Manage updates of Azure Arc-enabled Windows servers by using Azure Update Manager

Estimated Time: 25 minutes

In this exercise, you will:

- Configure Update Manager for Azure Arc-enabled Windows servers
- Review Update Manager capabilities for Azure Arc-enabled Windows servers

#### Task 1: Configure Update Manager for Azure Arc-enabled Windows servers

In this task, you will configure Update Manager for one of the Windows servers you onboarded to Azure Arc earlier in this lab. 

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, navigate to the **Azure Arc \| Machines** page and select the **arclab-winvm1** entry.
1. On the **arclab-winvm1** page, in the vertical navigation menu on the left side, in the **Operations** section, select **Updates**.
1. On the **arclab-winvm1 \| Updates** page, in the **Guest OS updates** section, select **Go to Updates by using Azure Update Manager**.
1. On the **arclab-winvm1 \| Updates** page, select **Check for updates** and, in the **Trigger assess now**, select **OK**.

    > **Note:** Do not wait for the assessment to complete, but instead proceed to the next step. The assessment might take about 5 minutes to complete.

1. On the **arclab-winvm1 \| Updates** page, select **Update settings**, on the **Change update settings** page, in the **Periodic assessment** drop-down list for **arclab-winvm1**, select **Enable**, review the other settings, and then select **Save**.

    > **Note:** Hotpatching is available only for Azure VMs running Windows Server Datacenter Azure Edition. Patch orchestration is not appliable to Arc-enabled servers.

1. On the **arclab-winvm1 \| Updates** page, select **Schedule updates**. 
1. On the **Basics** tab of the **Create a maintenance configuration** page, specify the following settings:

    |Setting|Value|
    |---|---|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Resource group|**arclab-servers-RG**|
    |Configuration name|**arclab-maintenance-configuration**|
    |Region|The name of the same Azure region you used in the previous exercises of this lab|
    |Maintenance scope|**Guest (Azure VM, Arc-enabled VMs/servers**|
    |Reboot setting|**Reboot if required**|

1. Select **Add a schedule**, in the **Add/Modify schedule** pane, specify the following settings and then select **Save**:

    |Setting|Value|
    |---|---|
    |Start on|next Saturday's date at **9:00 PM** of your local time zone|
    |Maintenance window|**3** Hours and **0** Minutes|
    |Repeats|**1 Week** on **Saturday**|
    |Add end date|disabled|

1. Back on the **Basics** tab of the **Create a maintenance configuration** page, select **Next: Dynamic scopes >**:
1. On the **Dynamic scopes** tab, select **Next: Resources >**.

    > **Note:** Dynamic scopes allow you to narrow down the scope of configuration by using such criteria as resource groups, locations, operating system types, or tags.

1. On the **Resources** tab, verify that **arclab-winvm1** appears in the list of resources and select **Next: Updates >**.
1. On the **Updates** tab, review the existing settings without making any changes and select **Review + create**.

    > **Note:** You have the option of including update classifications as well as including and excluding individual KB ID/packages.

1. On the **Review + create** tab, wait for the validation to complete and then select **Create**.

    > **Note:** Do not wait for the maintenance configuration setup to complete but instead proceed to the next step. The setup should complete within 1 minute. 

#### Task 2: Review Update Manager capabilities for Azure Arc-enabled Windows servers

In this task, you will review some of the resulting update management capabilities of Azure Arc-enabled Windows servers provided by Update Manager.

1. Navigate back to the **arclab-winvm1 \| Updates** page and select **One-time update**.
1. On the **Machines** tab of the **Install one-time updates** page, select the checkbox next to the **arclab-winvm1** entry and then select **Next**.
1. On the **Updates** tab, review the selected updates to install and select **Next**. 

    > **Note:** You have the option of including and excluding individual KB ID/packages.

1. On the **Properties** tab, in the **Reboot option** drop-down list, select **Never reboot** and select **Next**. 
1. On the **Review + install** tab, review the resulting settings and select **Install**.

    > **Note:** Do not wait for the update installation to complete. You can review it later on by reviewing the **History** tab on the **arclab-winvm1 \| Updates** page in the Azure portal or by triggering another assessment against the **arclab-winvm1** from the same page. 

### Exercise 6: Configure on-premises Windows servers by using Azure VM extensions and azcmagent CLI

Estimated Time: 45 minutes

In this exercise, you will:

- Configure Azure Arc-enabled Windows servers by using an Azure VM extension
- Configure Azure Arc-enabled Windows servers by using azcmagent CLI
- Deprovision the lab environment

#### Task 1: Configure Azure Arc-enabled Windows servers by using an Azure VM extension

In this task, you will use the Azure VM Custom Script Extension for Windows to install and configure Internet Information Services web site on one of the Windows servers you onboarded to Azure Arc earlier in this lab. 

1. Within the Remote Desktop session to **arclab-winvm1**, in the web browser window displaying the Azure portal, switch to the **Administrator: Windows PowerShell ISE** window and, if needed, open a new tab in the script pane.
1. In the Windows PowerShell ISE script pane, enter the following script that installs the **Web-Server** server role, removes the default home page of the default web site hosted by the web server, and replaces it with a custom page that displays the local computer name:

    ```powershell
    Install-WindowsFeature -name Web-Server -IncludeManagementTools
    Remove-Item -Path 'C:\inetpub\wwwroot\iisstart.htm'
    Add-Content -Path 'C:\inetpub\wwwroot\iisstart.htm' -Value "$env:computername"
    ```

1. Save the content of the script pane as `C:\Temp\Install_IIS.ps1`.
1. Within the Remote Desktop session to **arclab-winvm1**, switch to the web browser window displaying the Azure portal, 
1. In the Azure portal, search for and select **Storage accounts** and, on the **Storage accounts** blade, select **+ Create**.
1. On the **Basics** tab of the **Create a storage account** page, specify the following settings (leave others with their default values) and select **Next**:

    |Setting|Value|
    |---|---|
    |Subscription|The name of the Azure subscription you are using in this lab|
    |Resource group|**arclab-servers-RG**|
    |Storage account name|any globally unique name between 3 and 24 characters in length, consisting of lower case letters and digits, starting with a lower case letter|
    |Region|The name of the same Azure region you used in the previous exercises of this lab|
    |Performance|**Standard**|
    |Redundancy|**Locally-redundant storage (LRS)**|

1. On the **Advanced** tab, review the available options, accept the defaults, and select **Next**.
1. On the **Networking** tab, review the available options, accept the default option **Public endpoint (all networks}** and select **Next**.
1. On the **Data protection** tab, review the available options, accept the defaults, and select **Next**.
1. On the **Encryption** tab, review the available options, accept the defaults, select **Review + Create**, wait for the validation process to complete and select **Create**.

    >**Note**: Wait for the Azure Storage account to be created. This should take about 2 minutes.

1. Once the provisioning of the storage account completes, select **Go to resource**.
1. On the storage account page, in the **Data Storage** section, select **Containers** and then select **+ Container**.
1. In the **New container** pane, in the **Name** text box, enter **scripts** and select **Create**.
1. Back on the storage account page displaying the list of containers, select **scripts**.
1. On the **scripts** page, select **Upload**.
1. In the **Upload blob** pane, select **Browse for files**, in the **Open** dialog box, navigate to the `C:\Temp` folder, select the `Install_IIS.ps1` file you created earlier in this task, select **Open**, and back on the **Upload blob** page, select **Upload**.
1. In the Azure portal, navigate to the **Azure Arc \| Machines** page and select the **arclab-winvm1** entry.
1. On the **arclab-winvm1** page, in the vertical navigation menu on the left side, in the **Settings** section, select **Extensions**.
1. On the **arclab-winvm1 \| Extensions** page, select **+ Add**.
1. On the **Install extension** page, select **Custom Script Extension for Windows - Azure Arc** and then select **Next**.
1. On the **Configure Custom Script Extension for Windows - Azure Arc Extension** page, next to the **Script file (Required)** text box, select **Browse**.
1. On the **Storage accounts** page, select the name of the storage account into which you uploaded the `Install_IIS.ps1` script, on the **Containers** page, select **scripts**, on the **scripts** page, select **Install_IIS.ps1**, and then click **Select**.
1. Back on the **Configure Custom Script Extension for Windows - Azure Arc Extension** page, select **Review + create** and then select **Create**.

    >**Note**: This will automatically run the script on the server arclab-winvm1. Wait for the script to complete. This should take about 3 minutes.

1. To validate that the script completed successfully, open another tab in the web browser window and navigate to [http:\\localhost](http:\\localhost). Ensure that the page displays the name of the web server (i.e. `arclab-winvm1` in this case). Alternatively, you can review the content of the `C:\inetpub\wwwroot\iisstart.htm`. 

#### Task 2: Configure Azure Arc-enabled Windows servers by using azcmagent CLI

In this task, you will use azcmagent CLI to block installation of the legacy Azure Log Analytics agent (also referred to as Microsoft Monitoring Agent) on one of the Windows servers you onboarded to Azure Arc earlier in this lab. 

>**Note**: The Azure Connected Machine agent command line tool, implemented as azcmagent.exe (also known as azcmagent CLI), allows you configure, manage, and troubleshoot Azure Arc-enabled servers. is installed with the Azure Connected Machine agent and controls actions specific to the server where it's running. For more information regarding this tool, refer to [azcmagent CLI reference](https://learn.microsoft.com/en-us/azure/azure-arc/servers/azcmagent).

1. Within the Remote Desktop session to **arclab-winvm1**, from the Start menu, open the **Windows System** folder and then select **Command Prompt**.
1. From the Command Prompt, run the following command to list the current configuration settings of the Azure Arc Connected Machine agent:

    ```cmd
    azcmagent config list
    ```

1. Review the output and verify that it includes the following content:

    ```cmd
    Local Configuration Settings
      incomingconnections.enabled (preview)             : true
      incomingconnections.ports (preview)                   : []
      connection.type (preview)                             : direct
      proxy.url                                             :
      proxy.bypass                                          : []
      extensions.allowlist                                  : []
      extensions.blocklist                                  : []
      guestconfiguration.enabled                            : true
      extensions.enabled                                    : true
      config.mode                                           : full
      guestconfiguration.agent.cpulimit                     : 5
      extensions.agent.cpulimit                             : 5
    ```

1. Run the following command to block installation of Microsoft Monitoring Agent:

    ```cmd
    azcmagent config set extensions.blocklist "Microsoft.EnterpriseCloud.Monitoring/MicrosoftMonitoringAgent"
    ```

1. Re-run the `azcmagent config list` command, review its output, and verify that it includes the following content:

    ```cmd
    Local Configuration Settings
      incomingconnections.enabled (preview)             : true
      incomingconnections.ports (preview)                   : []
      connection.type (preview)                             : direct
      proxy.url                                             :
      proxy.bypass                                          : []
      extensions.allowlist                                  : []
      extensions.blocklist                                  : [Microsoft.EnterpriseCloud.Monitoring/MicrosoftMonitoringAgent]
      guestconfiguration.enabled                            : true
      extensions.enabled                                    : true
      config.mode                                           : full
      guestconfiguration.agent.cpulimit                     : 5
      extensions.agent.cpulimit                             : 5
    ```

#### Task 3: Deprovision the lab environment

In this task, you will remove all Azure resources created in this lab to avoid any extra charges. 

1. Switch back to the lab computer and, in the Web browser displaying the Azure portal, start a Cloud Shell PowerShell session. 
1. In the PowerShell session in the Cloud Shell pane, run the following command to delete all resources in the resource groups created throughout the lab:

    ```powershell
    Get-AzResourceGroup -Name 'arclab-infra-RG' | Remove-AzResourceGroup -Force -AsJob
    Get-AzResourceGroup -Name 'arclab-servers-RG' | Remove-AzResourceGroup -Force -AsJob
    ```

    >**Note**: The commands execute asynchronously (as determined by the -AsJob parameter), so it will take a few minutes before the resource groups and resources they contain are actually removed.
