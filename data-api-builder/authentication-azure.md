---
title: Azure authentication
description: Configure authentication in Azure for Data API builder using Microsoft Entra ID and various authentication methods/providers.
author: seesharprun
ms.author: sidandrews
ms.reviewer: jerrynixon
ms.service: data-api-builder
ms.topic: concept-article
ms.date: 04/01/2024
# Customer Intent: As a developer, I want to configure Azure authentication, so that I can authenticate in the Azure environment.
---

# Azure Authentication in Data API builder

Data API builder allows developers to define the authentication mechanism (identity provider) they want Data API builder to use to authenticate who is making requests.

Authentication is delegated to a supported identity provider where access token can be issued. An acquired access token must be included with incoming requests to Data API builder. Data API builder then validates any presented access tokens, ensuring that Data API builder was the intended audience of the token.

The supported identity provider configuration options are:

- StaticWebApps
- JSON Web Tokens (JWT)

## Azure Static Web Apps authentication (EasyAuth)

Data API builder expects Azure Static Web Apps authentication (EasyAuth) to authenticate the request, and to provide metadata about the authenticated user in the `X-MS-CLIENT-PRINCIPAL` HTTP header when using the option `StaticWebApps`. The authenticated user metadata provided by Static Web Apps can be referenced in the following documentation: [Accessing User Information](/azure/static-web-apps/user-information?tabs=csharp).

To use the `StaticWebApps` provider, you need to specify the following configuration in the `runtime.host` section of the configuration file:

```json
"authentication": {
    "provider": "StaticWebApps"
}
```

Using the `StaticWebApps` provider is useful when you plan to run Data API builder in Azure, hosting it using App Service and running it in a container: [Run a custom container in Azure App Service](/azure/app-service/quickstart-custom-container?tabs=dotnet&pivots=container-linux-vscode).

## JWT

To use the JWT provider, you need to configure the `runtime.host.authentication` section by providing the needed information to verify the received JWT token:

```json
"authentication": {
    "provider": "AzureAD",
    "jwt": {
        "audience": "<APP_ID>",
        "issuer": "https://login.microsoftonline.com/<AZURE_AD_TENANT_ID>/v2.0"
    }
}
```

## Roles selection

Once a request is authenticated via any of the available options, the roles defined in the token are used to help determine how permission rules are evaluated to [authorize](authorization.md) the request. Any authenticated request is automatically assigned to the `authenticated` system role, unless a user role is requested for use. For more information, see [authorization](authorization.md).

## Anonymous requests

Requests can also be made without being authenticated. In such cases, the request is automatically assigned to the `anonymous` system role so that it can be properly [authorized](authorization.md).

## Related content

- [Local authentication](authentication-local.md)
- [Authorization](authorization.md)
