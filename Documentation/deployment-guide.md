# Prerequisites
To begin, we will need:
* A "Teams tenant", ie. a tenant with Microsoft Teams turned on
* Access to create a new Team within the "Teams tenant"
* Access to create new App Registrations the "Teams tenant"
* Access grant admin consent for either Delegated or Application API permissions
* A QBot Service Account (if using [Delegated permissions](#graph-api-access-app-registration) when calling Graph API)

* An Azure subscription where you can create the following types of resources
  * App Service
  * App Service Plan
  * SQL Database
  * Bot Channels Registration
  * QnA Service
  * QnA Knowledge Base
* A copy of the QBot GitHub repo
* Some familiarity with building a .NET Web API project
* Some familiarity with building an Angular 7 application

Here is a summary of all Azure resources that need to be created:

|#|Resource Type|Description|
|:-|:-|:-|
|1|Resource Group|Logical container to place all QBot related Azure resources|
|2|Bot Channels Registration|QBot Bot Channels Registration|
|3|QnA Service|Cognitive Services to host QnA KBs|
|4|Azure Search Service|Provisioned as part of QnA Service, if required|
|5|QnA Knowledge Base|The backing QnA KB where QBot will get answers from.<br>You are required to have one QnA KB per Course|
|6|SQL Database|Stores QBot related processing data|
|8|App Service|Hosts the [QBot API Web Service](#qbot-api-web-app)|
|9|App Service|Hosts the [Dashboard Tab App](#dashboard-tab-web-app) Angular site|
|10|App Service|Hosts the [Questions Tab App](#questions-tab-web-app) Angular site|
|11|App Registration|To support the Bot Channels Registration|
|11|App Registration|To support the QBot API authentication|
|12|App Registration|To support Graph API access|


---

### QBot Resource Group
Create new Azure Resource Group as a logical container to place all resources provisioned
This can help manage resources and monitor costs more easily.

### QBot API Web App
Create a new **Web App** with the following values

|Setting|Value|
|:-|:-|
|Name|Something easily identifiable as QBot API<br>Example: `qbot-api.azurewebsites.net`|
|Runtime Stack|ASP.NET V4.7|

> Please note the following settings for later: <br>
> The resulting QBot API Web App URL<br>
> eg: `https://qbot-api.azurewebsites.net`

### Dashboard Tab Web App
Create a new **Web App** with the following values

|Setting|Value|
|:-|:-|
|Name|Something easily identifiable as Dashboard Tab<br>Example: `qbot-dashboard-tab.azurewebsites.net`|
|Runtime Stack|.NET Core 2.2|

> Please note the following settings for later: <br>
> The Dashboard Tab Web App URL<br>
> eg: `https://qbot-dashboard-tab.azurewebsites.net`

### Questions Tab Web App
Create a new **Web App** with the following values

|Setting|Value|
|:-|:-|
|Name|Something easily identifiable as Question Tab<br>Example: `qbot-questions-tab.azurewebsites.net`|
|Runtime Stack|.NET Core 2.2|

> Please note the following settings for later: <br>
> The Question Tab Web App URL<br>
> eg: `https://qbot-questions-tab.azurewebsites.net`

### SQL Server
Create a new **SQL Database** with the following values

|Setting|Value|
|:-|:-|
|Collation|SQL_Latin1_General_CP1_CI_AS|

Go to Firewall settings, and ensure the "Allow Azure services and resources to access this server" switch is turned ON.

![](images/sql-firewall.png)

> Please note the following settings for later: <br>
> The Azure SQL Server connection string<br>
> eg: `data source=qbot-azure-sql-server;initial catalog=qbot-db;user id=sql-user;password=*****;MultipleActiveResultSets=True;App=EntityFramework&quot;`


#### Bot Channels App Registration
Create a new Azure *App Registration* for the purpose of Bot Channels Registration.

This App Registration does not have to be on the same tenant as your Teams instance.
You will need to be an Application Administrator on your tenant

|Setting|Value|
|:-|:-|
|Account Type|*Accounts in any organizational directory (Any Azure AD directory - Multitenant)*|

> Please note the following settings for later:<br>**Application (client) ID**

#### QBot API Auth App Registration
The custom Teams tabs are Angular apps that call the QBot API service (.NET Web API). This App Registration is used to authenticate these API calls.

This App Registration does not have to be on the same tenant as your Teams instance.

Create a new **App Registration** for the purpose of QBot API Authentication.

This App Registration MUST be on the same tenant as your Teams instance.
You will need to be an Application Administrator on your tenant


|Setting|Value|
|:-|:-|
|Account Type|*Accounts in any organizational directory (Any Azure AD directory - Multitenant)*|
|Authentication<br>Redirect URIs|https://qbot-dashboard-tab.azurewebsites.net/app-silent-end<br>https://qbot-questions-tab.azurewebsites.net/app-silent-end<br><br>Note: These are the same URLs where you deployed the [Dashboard Tab](#dashboard-tab-web-app) and the [Questions Tab](#questions-tab-web-app)

> Please note the following settings for later:<br>
> **Application (client) ID**<br>
> **Directory (tenant) ID**

#### Graph API Access App Registration
The QBot API service in turn calls Graph API to retrieve information like the questions asked, and conversations within a channel. This App Registration is used to authenticate these Graph API calls.

Create a new **App Registration** to allow Graph API access. You will need to be an Application Administrator on the "Teams tenant" to do this.

Note that this App Registration MUST be on the same tenant as your Teams instance.

##### Application vs Delegated Permissions
You will need to grant Graph API permissions to this App Registration (the full list of permissions is below). QBot supports both Application and Delegated permission types, so choose one that is most appropriate for your organisation.

One thing to be aware of is that QBot does use the **Teams Protected APIs** to retrieve things like channel messages. If you decide to use [Application Permissions](https://docs.microsoft.com/en-us/graph/auth/auth-concepts#microsoft-graph-permissions), then you must submit a request form so that this App Registration can use these protected APIs. More information can be found here: https://docs.microsoft.com/en-us/graph/teams-protected-apis

|Setting|Value|
|:-|:-|
|Account Type|*Accounts in any organizational directory (Any Azure AD directory - Multitenant)*|
|API Permissions|Add the following API permissions:<br><br>Name: **User.Read.All**<br>Type: **Application** or **Delegate**<br>Consent: **Required**<br><br>Name: **Group.ReadWrite.All**<br>Type: **Application** or **Delegate**<br>Consent: **Required**<br><br>Name: **Sites.ReadWrite.All**<br>Type: **Application** or **Delegate**<br>Consent: **Required**

When granting admin consent for these, you will need to be a Global Administrator on the respective tenant.

Go to Certificates and Secrets, add a new client secret with a suitable expiration date. Remember to copy the generated client secret.

If using Delegated permissions, go to the Authentication tab, and set to treat application as public client by default
![](images/app-reg-default-client-type.png)

> Please note the following settings for later:<br>**Application (Client) ID**
> <br>**Client Secret**

### Bot Channels Registration
Create a new **Bot Channels Registration** resource with the following values:

|Setting|Value|
|:-|:-|
|Bot handle|Any unique identifier, you can change the bot display name later|
|Pricing Tier|Pick an appropriate pricing tier for your needs|
|Messaging EndPoint|(QBot API URL)`/api/messages`<br>Example<br>If QBot API will be deployed to `https://qbot-api.azurewebsites.net`<br>then enter in `https://qbot-api.azurewebsites.net/api/messages`|
|Microsoft App ID and password|Auto create App ID and password|

When finished, go into the newly created Bot Channels Registration, under **Settings** tab:

> Please note the following settings for later:
> * Note down the **Microsoft App ID**
> * Click **Manage**
> * Note down a secret

Click on Channels setting, and add *Microsoft Teams* as a featured channel

![](images/bot-reg-teams-channel.png)


### QnA Maker Service
QBot uses QnA maker as it's knowlege base of questions and answers. Each course in QBot will require a back-end QnA KB provisioned, and this relationship is 1-1, ie. One QnA KB required per QBot Course.

https://www.qnamaker.ai/Create

1. Create a new QnA Service. Select an appropriate pricing tier to store all KBs for all required courses
2. Create the KB, and take note of the KB Name
3. (Optional) Populate your KB with initial data or add a chit-chat persona.
4. Publish the KB


> Please take note of the following settings for later:<br><br>
> ![](images/qna-deploy.png)
> 1. **QnA Service Host** - The full Host header
> 2. **QnA Knowledge Base ID** - The GUID part of the POST URL
> 3. **QnA Endpoint Key** - The GUID part of the Authorization header<br><br>
> ![](images/qna-configure.png)
> 4. **QnA HTTP Endpoint**
> 5. **QnA HTTP Key**



## QBot Application Build & Deployment
After your Azure resources are setup, next step is to Build QBot from source control. Grab the latest copy of all source code from the Git repo.


### QBot API
QBot API project is called `Microsoft.Teams.Apps.QBot.Bot` and is a .NET 4.5 Web API

#### Updates to web.config
The following values need to be updated, depending on the environment and installation details. `Web.config` separates it's secrets in 2 external files: `appSettings.secret.config` and `connectionStrings.secret.config`

``` xml
 <connectionStrings configSource="connectionStrings.secret.config"></connectionStrings>
 <appSettings file="appSettings.secret.config"></appSettings>
```

``` xml
<!-- appSettings.secret.config -->
<appSettings>
    <add key="BotId" value="" />
    <add key="BaseUrl" value="" />
    <add key="MicrosoftAppId" value="" />
    <add key="MicrosoftAppPassword" value="" />

    <add key="AADAuthority" value="" />
    <add key="AADServiceName" value="" />
    <add key="AADServicePassword" value="" />
    <add key="AADClientId" value="" />
    <add key="AADClientSecret" value="" />
    <add key="AADPermissionType" value="" />

    <add key="ida:ClientId" value="" />
    <add key="ida:TenantId" value="" />
    <add key="ida:Audience" value="" />
</appSettings>
```
|Key|Description|Value|
|:-|:-|:-|
|BotId|The Bot ID|`Question`|
|BaseUrl|The root URL where the [QBot API Web App](#qbot-api-web-app) was deployed|`https://qbot-api.au.ngrok.io`|
|MicrosoftAppId|Bot Channels Registration Microsoft App ID|Refer to steps in [Bot Registration](#bot-channels-registration)|
|MicrosoftAppPassword|Bot Channels Registration Secret|Refer to steps in [Bot Registration](#bot-channels-registration)|
|AADAuthority|Graph API target resource identifier|`https://login.microsoftonline.com/(tenantId)`|
|AADServiceName|QBot service account. Must be a valid Teams account. Only applicable when using **Delegate** permissions (see AADPermissionType below)|svc_qbot@unsw.edu.au|
|AADServicePassword|Encrypted password of the above QBot service account.  Only applicable when using **Delegate** permissions (see AADPermissionType below)|Use the **StringEncryption** project to encrypt the password of the QBot service account|
|AADClientId|Client ID of the Azure App Registration that uses Graph API to access Teams  conversations and replies|Refer to steps in [Graph API App Registration](#graph-api-access-app-registration)|
|AADClientSecret|Client secret of the Azure App Registration that uses Graph API to access Teams  conversations and replies|Refer to steps in [Graph API App Registration](#graph-api-access-app-registration)|
|AADPermissionType|Either **Application** or **Delegate**, depending on how API access permissions were set up|Refer to steps in [Graph API App Registration](#graph-api-access-app-registration)|
|ida:ClientId|Client ID of the Azure App Registration reuired to authenticate the QBot API|Refer to steps in [QBot API Authentication App Registration](#qbot-api-auth-app-registration)|
|ida:TenantId|QBot API Auth Azure App Registration - Tenant ID|Refer to steps in [QBot API Authentication App Registration](#qbot-api-auth-app-registration)|
|ida:Audience|QBot API Auth Azure App Registration - Application ID|api://5e55f7ba-8453-41b2-a274-75b6a71e4473|

``` xml
<!-- connectionStrings.secret.config -->

<!-- Replace ***** with your SQL Server, database, username & password -->
<connectionStrings>
    <add name="QBotEntities" connectionString="metadata=res://*/QuestionBotModel.csdl|res://*/QuestionBotModel.ssdl|res://*/QuestionBotModel.msl;provider=System.Data.SqlClient;provider connection string=&quot;data source=*****;initial catalog=*****;user id=*****;password=*****;MultipleActiveResultSets=True;App=EntityFramework&quot;" providerName="System.Data.EntityClient" />
</connectionStrings>
```

Finally, right-click on the "Microsoft.Teams.Apps.QBot.Bot" project, and choose "Publish" to your [QBot API](#qbot) web site 

### Angular apps - Dashboard Tab & Questions Tab
There are 2 Teams tabs, developed as Angular applications. They are in the projects: `DashboardTabApp` and `QuestionsTabApp`. Both are built and deployed in the same way.


Open up `Source\DashboardTabApp\src\environments\environment.ts` and make the following changes
``` typescript
export const environment = {
    production: false,
    apiBaseUrl: "https://qbot-api.azurewebsites.net/api/Request/",
    
    authConfig: {
        instance: "https://login.microsoftonline.com/",
        tenantId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
        clientId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
        redirectUri: "/app-silent-end",
        cacheLocation: "localStorage",
        navigateToLoginRequestUrl: false,
        extraQueryParameters: "",
        popUp: true,
        popUpUri: "/app-silent-start",
        popUpWidth: 600,
        popUpHeight: 535
    },

    // do not populate the following:
    upn: "",
    tid: "",
};
```
Key|Value
:-|:-
apiBaseUrl|The URL where the [QBot API Web App](#qbot-api-web-app) is deployed, with `/api/Request/` appended
tenantId|The Directory (tenant) ID of the [QBot API Auth App Registration](#qbot-api-auth-app-registration)
clientId|The Application (client) ID of the [QBot API Auth App Registration](#qbot-api-auth-app-registration)


Open up `Source\QuestionTabApp\src\environments\environment.ts` and make the following changes
``` typescript
export const environment = {
    production: false,
    apiBaseUrl: "https://qbot-api.azurewebsites.net/api/Request/",
    selfUrl: "",

    authConfig: {
        instance: "https://login.microsoftonline.com/",
        tenantId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
        clientId: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
        redirectUri: "/app-silent-end",
        cacheLocation: "localStorage",
        navigateToLoginRequestUrl: false,
        extraQueryParameters: "",
        popUp: true,
        popUpUri: "/app-silent-start",
        popUpWidth: 600,
        popUpHeight: 535
    },

    // do not populate the following:
    upn: "",
    tid: "",
    gid: "",
    cname: ""
};
```
Key|Value
:-|:-
apiBaseUrl|The URL where the [QBot API Web App](#qbot-api-web-app) is deployed, with `/api/Request/` appended
tenantId|The Directory (tenant) ID of the [QBot API Auth App Registration](#qbot-api-auth-app-registration)
clientId|The Application (client) ID of the [QBot API Auth App Registration](#qbot-api-auth-app-registration)
selfUrl|The base URL where this [Questions Tab App](#questions-tab-web-app) (Angular app is deployed), eg: `https://qbot-questions-tab.azurewebsites.net`|

Run the following commands to restore packages and build
```
npm install
ng-build
```

The output dist files will be generated in their respective `wwwroot` folder.

For the Angular `DashboardTabApp` application, copy these files to the [Dashboard Tab Web App]() application.<br>
For the Angular `QuestionsTabApp` application, copy these files to the [Questions Tab Web App]() application.<br>


### SQL Database configuration
Run the included SSDT package to create the initial SQL database schema and seed data.
To do this within Visual Studio, right click on the "Microsoft.Teams.Apps.QBot.Database" project, and choose "Publish".
Fill in the target database connection based on the [provisioned SQL Server](#sql-server) settings

![](images/publish-database.png)


### Deploy the Bot to Teams
#### Prepare the manifest file
Edit the `manifest.json` file, and replace the following values:

``` json
{
  "$schema": "https://statics.teams.microsoft.com/sdk/v1.2/manifest/MicrosoftTeams.schema.json",
  "manifestVersion": "1.2",
  "version": "1.0.0",
  "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
  "packageName": "qbot",
  "developer": {
    "name": "UNSW Sydney",
    "websiteUrl": "https://www.unsw.edu.au/",
    "privacyUrl": "https://www.unsw.edu.au/",
    "termsOfUseUrl": "https://www.unsw.edu.au/"
  },
  "icons": {
    "outline": "qbot.png",
    "color": "qbot.png"
  },
  "name": {
    "short": "Question",
    "full": "Question"
  },
  "description": {
    "short": "QBot",
    "full": "QBot keeps track of answered and unanswered questions, sends notifications to tutors and teachers in charge, and dynamically constructs its own knowledge database on the subject to help suggest answers in future. Just tag @question in the conversations tab of your class Team."
  },
  "accentColor": "#3F487F",
  "composeExtensions": [
    {
      "botId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "canUpdateConfiguration": true,
      "commands": [
        {
          "id": "SelectAnswer",
          "description": "Select an answer",
          "title": "Select answer",
          "type": "action",
          "context": [ "message" ],
          "fetchTask": true,
          "parameters": [ { "name": "SelectAnswer", "title": "Select Answer" } ]
        }
      ]
    }
  ],
    "configurableTabs": [
      {
        "configurationUrl": "https://qbot-questions-tab.azurewebsites.net/config?upn={upn}&tid={tid}&gid={gid}&cname={channelName}",
        "canUpdateConfiguration": true,
        "scopes": [
          "team"
        ]
      }
    ],
  "bots": [
    {
      "botId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "scopes": [
        "team",
        "personal"
      ]
    }
  ],
  "staticTabs": [
    {
      "entityId": "DashboardTab",
      "name": "Dashboard",
      "contentUrl": "https://qbot-dashboard-tab.azurewebsites.net/home?upn={upn}&tid={tid}&gid={groupId}&uid={userObjectId}",
      "scopes": [ "personal" ]
    }
  ],
  "permissions": [
    "identity",
    "messageTeamMembers"
  ],
  "validDomains": [ "qbot-dashboard-tab.azurewebsites.net", "qbot-questions-tab.azurewebsites.net", "qbot-api.azurewebsites.net" ]
}
```

|Key|Value|
|:-|:-|
id|Microsoft AppID (GUID) from the [Bot Channel Registration](#bot-channels-registration)
botId|Microsoft AppID (GUID) from the [Bot Channel Registration](#bot-channels-registration).<br>Remember to replace **both instances** in the `manifest.json`
configurationUrl|URL of the deployed [Question Tab Angular web application](#questions-tab-web-app) with `/#/config?upn={upn}&tid={tid}&gid={gid}&cname={channelName}` appended.
contentUrl|URL of the deployed [Dashboard Tab Angular web application](#dashboard-tab-web-app) with `/#/home?upn={upn}&tid={tid}&gid={groupId}&uid={userObjectId}` appended.
validDomains|Array of three strings representing the domains of the [Bot API Web App](#qbot-api-web-app), [Question Tab](#questions-tab-web-app) and [Dashboard Tab](#dashboard-tab-web-app)


So now, within Manifest` folder there will be 2 files
* `manifest.json` - Manifest file, which we just updated in steps above
* `bot_blue.png` - Starter bot icon

Zip up into a new package file (eg. `qbot-manifest.zip`) ready for upload into Teams

### Add QBot app into Teams
1. You must be a team owner
2. In Teams, go to the Team the Bot will be deployed to and click on the ‘+’ symbol under the Team title at the top of the window.
3. In the popup click on *Manage apps* at the bottom right corner which will navigate to the Apps page for that team.
4. At the bottom right corner click on *Upload a custom app* and select the .zip file created earlier which will add the bot to the Team.


## QBot Setup
Congratulations, you have succesfully built the QBot solution, and added the App into Teams. Final step is to set up the different courses

1. Go to the dashboard tab (initiate a personal coversation with the Bot)
2. Create a new course
3. Fill in the following values:

|Setting| Value|
|:-|:-|
|CourseName|Dropdown will show all Teams that you are a owner of. Select the Team to use as your course|
|PredictiveQnAServiceHost|**QnA Service Host** value from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAKnowledgeBaseId|**QnA Knowledge Base ID** value from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAEndpointKey|**QnA Endpoint Key** value from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAHttpKey|**QnA HTTP Key** value from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAHttpEndpoint|**QnA HTTP Endpoint** value from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAKnowledgeBaseName|Name of the QnA Knowledge Base from the [QnA Maker Setup](#qna-maker) step|
|PredictiveQnAConfidenceThreshold|Integer that should be from 0-100 that reflects the confidence percentage an answer from QnA Maker must be if it is to be supplied as an answer to a question|

4. Create the required Tutorial Groups
5. Assign and Map Users. Click the *Synchronise from Teams* button and assign their roles and any necessary tutorial groups
