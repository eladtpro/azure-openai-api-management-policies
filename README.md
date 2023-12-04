![Flow](assets/OpenAI.Restrict-Round-Robin.png)


# OpenAI usage management with Azure API Management (APIM)

This article aims to provide guidance for organizations that have concerns when using OpenAI services.  
These concerns may include addressing auditing prompts and responses, capacity planning and limitations, error handling, and retry capabilities.  
In addition, organizations can increase their usage by creating an OpenAI instance pool and sharing resources with other consumers.  
  
To achieve all of the above and more, we will be using Azure OpenAI services via Azure API Management (APIM). 

---

###### Preps
[Prerequisites](#prerequisites)  
[Workflow](#workflow)  
###### Setting up Azure API Management (APIM) for OpenAI 
[Setup: Azure API Management Key Vault integration](#keyvault)  
[Setup: Azure API Management Backend services](#policies)  
[Setup: Azure API Management API with policies](#policies)  
[Setup: Azure API Management logging settings](#logging)  
[Pricing and cost savings](#pricing)
###### Deployment & Testing
[Azure Data Explorer (ADX): Run KQL Queries to extract usage details](#kql)  
[Testing with Postman](#postman)  
[DevOps: API Management configuration deployment](#devops)  
[API Management configuration git repository](#github)  


## <a name="workflow"></a>Prerequisites
* If you don't have an [Azure subscription](https://learn.microsoft.com/en-us/azure/guides/developer/azure-developer-guide#understanding-accounts-subscriptions-and-billing), create an [Azure free account](https://azure.microsoft.com/free/?ref=microsoft.com&utm_source=microsoft.com&utm_medium=docs&utm_campaign=visualstudio) before you begin.  
* The Azure CLI version is 2.47.0 or later. Run az --version to find the version, and run az upgrade to upgrade the version. If you need to install or upgrade, see [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
* If you have multiple Azure subscriptions, select the appropriate subscription ID in which the resources should be billed using the [az account](https://learn.microsoft.com/en-us/cli/azure/account) command.
* Understand [Azure API Management terminology](https://learn.microsoft.com/en-us/azure/api-management/api-management-terminology).
* [Create an Azure API Management instance](https://learn.microsoft.com/en-us/azure/api-management/get-started-create-service-instance).


## <a name="workflow"></a> Workflow
1. Client applications access Azure OpenAI endpoints to perform text generation (completions) and model training (fine-tuning).
2. Azure Application Gateway provides a single point of entry to Azure OpenAI models and provides load balancing for APIs.

    > Note
    > Load balancing of stateful operations like model fine-tuning, deployments, and inference of fine-tuned models isn't supported.


3. Azure API Management enables security controls and auditing and monitoring of the Azure OpenAI models.
    a. In API Management, enhanced-security access is granted via Microsoft Entra groups with subscription-based access permissions.
    b. Auditing is enabled for all interactions with the models via Azure Monitor request logging.
    c. Monitoring provides detailed Azure OpenAI model usage KPIs and metrics, including prompt information and token statistics for usage traceability.

4. API Management connects to all Azure resources via Azure Private Link. This configuration provides enhanced security for all traffic via private endpoints and contains traffic in the private network.

5. Multiple Azure OpenAI instances enable scale-out of API usage to ensure high availability and disaster recovery for the service.

---

## Setting up Azure API Management (APIM) for OpenAI
>  [Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) is a hybrid, multicloud management platform for APIs across all environments. As a platform-as-a-service, API Management supports the complete API lifecycle.  

### <a name="keyvault"></a>Setup: Azure API Management Key Vault integration

###### ToDo:
* [ ] If no Key Vault exists, create a new API management integrated Key Vault:  
[Use managed identities in Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-managed-service-identity)
* [ ] Navigate to API Management service > APIs > Named values > "+Add"
* [ ] In "Type" select "Key Vault"
* [ ] Press "Select"
* [ ] Select the formally created Key Vault and the desired secret
* [ ] Click "Save"

![APIM Key Vault](assets/apim-key-vault-integration.png)

### <a name="backend"></a>Setup: Azure API Management Backend services
> *Managed Identities* are a feature of *Azure Active Directory* that allows Azure resources to authenticate themselves as service principal with other supported Azure resources.
> This way *Azure API Management* can authenticate itself with *Azure OpenAI* services. **In this method there is no need to store any credentials in the APIM configuration**, e.g no need for OpenAI api-key.

Backends are the API Management representation of the backend services that API Management proxies. this way we can configure the Azure OpenAI services as backends. Each backend configuration will contain refernce to api-key saved in Azure Key Vault

###### ToDo:  
* [ ] Navigate to API Management service > APIs > Backends > "+Add"
* [ ] Enter OpenAI service url with the "/openai" suffix, e.g "https://openaitwo.openai.azure.com/openai"
* [ ] Add OpenAI api-key header if needed, in case not using managed identities
* [ ] Click "Create".

![Backends](assets/backends.png) 

### <a name="policies"></a>Setup: Azure API Management policies  

> For the API to be flexable for any URL path route, we are setting the Frontend URL to be a `/{*path}` like so.
> ![APIM Frontend URL](assets/frontend-wildcard.png)

###### ToDo:  

**Option I**: Configure policies manually using Azure Portal
* [ ] Navigate to API Management service > APIs > "+Add API"
* [ ] Navigate to API Management service > APIs > Your API > "+Add operation"
* [ ] Navigate to API Management service > APIs > Your API > Operations > Your operation > "Add policy"
* [ ] Add the desired policies, an OpenAI example can be found [here](/.apim-sweden.scm.azure-api.net/api-management/policies/apis/Open_AI__1[Current]_open-166H7PW/operations/POST__{_path}_open-aipost-query.xml)
* [ ] Save the changes

**Option II**: Configure policies manually using Visual Studio Code
* [ ] Open Visual Studio Code
* [ ] Install the [Azure API Management extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-apimanagement)
* [ ] Navigate to API Management configuration repo > policies > apis > Your API
* [ ] Modify the desired policy, an OpenAI example can be found [here](/.apim-sweden.scm.azure-api.net/api-management/policies/apis/Open_AI__1[Current]_open-166H7PW/operations/POST__{_path}_open-aipost-query.xml)
* [ ] Save the changes

![VS Code](assets/policies-vscode.png)

**Option III**: Deploy service configuration from git to the API Management service instance  
> The full guide can be found in [How to save and configure your API Management service configuration using Git](https://learn.microsoft.com/en-us/azure/api-management/api-management-configuration-repository-git#deploy-service-configuration-changes-to-the-api-management-service-instance)
* [ ] Clone the ".apim-sweden.scm.azure-api.net" repository to your repository.
* [ ] Navigate to API Management service > Deployment + infrastructure > Repository
* [ ] Press "Deploy to API Management"
* [ ] On the Deploy repository configuration page, enter the name of the branch containing the desired configuration changes, and optionally select Remove subscriptions of deleted products. Select Save.  


###### Policies
Azure API Management uses policies to modify API behavior by running sequential statements on the request or response.  
Some of the relevant policies for OpenAI include:
* [*rate-limit*](https://learn.microsoft.com/en-us/azure/api-management/rate-limit-policy): The rate-limit policy prevents API usage spikes by limiting the call rate to a specified number per period.  
* [*validate-jwt*](https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy): The validate-jwt policy enforces the existence and validity of a supported JSON web token (JWT).  
* [*set-header*](https://learn.microsoft.com/en-us/azure/api-management/set-header-policy): The set-header policy sets the value of a header or adds a new header to the request or response, e.g OpenAI api-key header.
* [*retry*](https://learn.microsoft.com/en-us/azure/api-management/retry-policy): The retry policy executes its child policies once and then retries their execution until the retry condition is met.
* [*cache-lookup-value*](https://learn.microsoft.com/en-us/azure/api-management/cache-lookup-value-policy): The cache-lookup-value policy performs cache lookup by key and return a cached value.  
* [*set-backend-service*](https://learn.microsoft.com/en-us/azure/api-management/set-backend-service-policy): edirect an incoming request to a different backend than the one specified in the API settings for that operation.  
* [*return-response*](https://learn.microsoft.com/en-us/azure/api-management/return-response-policy): The return-response policy returns a specified response to the caller. for adding a custom response data to the request.

Other policies are available out of the box. Policies are applied inside the gateway between the API consumer and the managed API, allowing changes to both the inbound request and outbound response. For a complete list, see [API Management policy reference](https://learn.microsoft.com/en-us/azure/api-management/api-management-policies).  





### <a name="logging"></a>Setup: Azure API Management logging settings
For us to be able to collect usage data we need to enable logging for the API Management service.
###### ToDo:
* [ ] Navigate to API Management service > APIs > Your API > Settings
* [ ] Scroll down to "Diagnostics logs" and select "Azure Monitor"
* [ ] Check the Backend Request and the Frontend Request checkboxes 
* [ ] Set the payload threshold to the desired value, e.g 4096
* [ ] Click "Save"

![APIM Monitoring](assets/diagnostics-logs.png)

---

## Deployment & Testing

### <a name="kql"></a>Azure Data Explorer (ADX): Run KQL Queries to extract usage details



#### Query usage analytics



```
ApiManagementGatewayLogs
| where OperationId == 'post-query' //completions_create'
| extend model = tostring(parse_json(BackendResponseBody)['model'])
| extend prompt_tokens = parse_json(parse_json(BackendResponseBody)['usage'])['prompt_tokens']
| extend completion_tokens = parse_json(parse_json(BackendResponseBody)['usage'])['completion_tokens']
| extend total_tokens = parse_json(parse_json(BackendResponseBody)['usage'])['total_tokens']
| extend request_messages = substring(parse_json(parse_json(BackendRequestBody)['messages'])[0], 0, 100)
| extend response_choices = substring(parse_json(parse_json(BackendResponseBody)['choices'])[0], 0, 100)
| extend requestBody = parse_json(BackendRequestBody)
| extend responseBody = parse_json(BackendResponseBody)
```


#### Query prompt details

```
ApiManagementGatewayLogs
| where OperationId == 'completions_create'
| extend model = tostring(parse_json(BackendResponseBody)['model'])
| extend prompttokens = parse_json(parse_json(BackendResponseBody)['usage'])['prompt_tokens']
| extend prompttext = substring(parse_json(parse_json(BackendResponseBody)['choices'])[0], 0, 100)
```


### <a name="postman"></a>Testing with Postman

We have severel options for testing the API, one of them is using Postman.
We generated in Postman two identical requests, one will go through the API Management service and the other will go directly to the OpenAI service.

You can import the configuration from [/postman/azure-open-ai.postman_collection.json](/postman/azure-open-ai.postman_collection.json)


###### ToDo:
* [ ] Open the imported collection "azure-open-ai"
* [ ] Navigate to the variables tab
* [ ] Set the "openai-api-key" variable to the openAI service api-key
* [ ] Set the "apim-subscription-key" variable to the API Management service subscription key
* [ ] Go to the openai request and press "Send"
* [ ] Go to the apim request and press "Send"
* [ ] Compare the responses (they should be identical)

![Postman](assets/postman.png)

##### Azure API Management tracing headers



| Header Name   |      Value      | Aim |
|----------|-------------|------|
| Ocp-Apim-Subscription-Key |  [API-KEY] | By publishing APIs through API Management, you can easily secure API access using subscription keys. Users who need to consume the published APIs must include a valid subscription key in HTTP requests when calling those APIs. |
| Ocp-Apim-Trace |    true   |   The [Trace policy](https://learn.microsoft.com/en-us/azure/api-management/trace-policy) adds a custom trace to the request tracing output in the test console when tracing is triggered, that is, Ocp-Apim-Trace request header is present and set to true and Ocp-Apim-Subscription-Key request header is present and holds a valid key that allows tracing. |
| Content-Type | application/json |    OpenAI service expect this header to be presence and contain a json content |


### <a name="github"></a>API Management configuration git repository

In this repository you will find the following files:
[apim-sweden.scm.azure-api.net](apim-sweden.scm.azure-api.net) sub-module contains the API Management configuration files.


APIM configuration files are stored in the [apim-sweden.scm.azure-api.net](apim-sweden.scm.azure-api.net) sub-module.


in the image that describes the repository structure, we can see it corresponds to the apim entities:  
apis, backends, products, policies, etc.

![APIM Config](assets/APIM-repo-config.png)












### DevOps: API Management configuration deployment 

#### Automate configuration deployment your API Management service configuration using Git

[How to save and configure your API Management service configuration using Git](https://learn.microsoft.com/en-us/azure/api-management/api-management-configuration-repository-git)




![Azure Config](assets/api-management-enable-git.png)



#### Get access credentials

To clone a repository, in addition to the URL to your repository, your need a username and a password.

###### ToDo:  
* [ ] On the **Repository** page, select **Access credentials** near the top of the page.
* [ ] Note the username provided on the **Access credentials** page.
* [ ] To generate a password, first ensure that the **Expiry** is set to the desired expiration date and time, and then select **Generate**.
* [ ] Set the credentials in your Git client to the username and password provided on the **Access credentials** page.
* [ ] Run the following commands for setting the credentials in your Git client:

```
# windows
git config --global credential.helper wincred

# generic
git config --global credential.provider generic

# mac
git config --global credential.helper osxkeychain

git config credential.https://{name}.scm.azure-api.net.username apim
git config credential.https://{name}.scm.azure-api.net.password apim

# reset helper if needed: git config --global --unset credential.helper
```

> Make a note of this password. 
> Once you leave this page the password will not be displayed again.


### Clone the repository to your local machine


#### Clone / Add Submodule

```
# clone
git clone https://{name}.scm.azure-api.net/

# clone with credentials
git clone https://username:password@{name}.scm.azure-api.net/

# create submodule
git submodule add https://{name}.scm.azure-api.net   

# if using branch other than master
git checkout <branch_name>

```

 

![APIM Config](assets/api-management-git-configure.png)

[Terrafrom - Azure API Management](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/api_management)  