+++
title = "Managing access to multiple AWS Accounts with OpenID"
description = "This blog post walks through a design and implementation for managing access to multiple AWS accounts using Keycloak"
keywords = ["keycloak", "okta", "auth0", "jwt", "jwks", "lambda", "serverless", "openid", "oidc", "federated", "aws", "active directory", "identity", "access", "multiple"]
categories = ["Cloud", "Security", "AWS", "Serverless"]
date = "2018-12-15T14:18:32Z"
+++

Many organisations look towards a multiple account strategy with Amazon Web Services (AWS) to provide administrative isolation between workloads, limited visibility and discoverability of workloads, isolation to minimize blast radius, management of AWS limits and cost categorisation. However, this comes at a large complexity cost, specifically around Identity Access Management (IAM).

Starting off with a single AWS account, and using a handful of IAM users and groups for access management, is usually the norm. As an organisation grows they start to see a need for separate staging, production, and developer tooling accounts. Managing access to these can quickly become a mess. Do you create a unique IAM user in each account and provide your employees with the unique sign-on URL? Do you create a single IAM user for each employee and use [`AssumeRole`](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) to generate credentials and to enable jumping between accounts? How do employees use the AWS Application Programming Interface (API) or the Command Line Interface (CLI); are long-lived access keys generated? How is an employee's access revoked should they leave the organisation?

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_0.jpg)
User per account approach

</center>

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_1.jpg)
All users in a single account
using STS AssumeRole to access other accounts

</center>

# Design

## Reuse employees existing identities

In most organisations, an employee will already have an identity, normally used for accessing e-mail. These identities are normally stored in Active Directory (AD), Google Suite (GSuite) or Office 365. In an ideal world, these identities could be reused and would grant access to AWS. This means employees would only need to remember one set of credentials and their access could be revoked from a single place. 

## Expose an OpenID compatible interface for authentication

OpenID provides applications with a way to verify a users identity using JSON Web Tokens (JWT). Additionally, it provides profile information about the end user such as first name, last name, email address, group membership, etc. This profile information can be used to store the AWS accounts and AWS roles the user has access to.

By placing an OpenID compatible interface on top an organisation's identity store users can easily generate JWTs which can be later used by services to authenticate them.

## Trading JWTs for AWS API Credentials

In order to trade JWTs for AWS API Credentials, a service can be created that runs on AWS with a role that has access to AssumeRole.This service would be responsible for validating a users JWT, ensuring the JWT contains the requested role and executing STS AssumeRole to generate the AWS API Credentials.

Additionally, the service would also generate an [Amazon Federated Sign-On URL](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html) which would enable users to access the AWS Web Console using their JWT.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_2.png)

</center>

# Example Implementation

Provided below is an example implementation of the above design. One user with username "demo" and password “demo” exists. Please do not use this demo in a production environment without https.

To follow along, clone or [download](https://github.com/imduffy15/aws-credentials-issuer/archive/master.zip) the code at [https://github.com/imduffy15/aws-credentials-issuer](https://github.com/imduffy15/aws-credentials-issuer).

 

## OpenID Provider

[Keycloak](https://www.keycloak.org/) provides an IAM along with OpenID Connect (OIDC) and Security Assertion Markup Language (SAML) interfaces. Additionally, it supports [federation](https://www.keycloak.org/docs/3.0/server_admin/topics/user-federation.html) to Active Directory (AD) and Lightweight Directory Access Protocol (LDAP) servers. Keycloak enables organisations to centrally manage employees access to many different services.

The[ provided code](https://github.com/imduffy15/aws-credentials-issuer) contains a [docker-compose file](https://github.com/imduffy15/aws-credentials-issuer/blob/master/docker-compose.yaml) that on executing docker-compose up will bring up a keycloak server with its administrative interface accessible at [http://localhost:8080/auth/admin/](http://localhost:8080/auth/admin/) using username "admin" and password “password”.

On the "clients" screen, a client named “aws-credentials-issuer” is present, users will use this client to generate their JWT tokens. This client is pre-configured to work with the [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1) for command line interfaces to generate tokens and the [Implicit Grant](https://tools.ietf.org/html/rfc6749#section-4.2) for a frontend application.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_3.png)

</center>

Under the "aws-credentials-issuer" additional roles can be added, these roles must exist on AWS and they must have a trust relationship to the account that will be running the “aws-credentials-issuer”.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_4.png)

</center>

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_5.png)

</center>

Additionally, these roles must be placed into the users JWT tokens, this is pre-configured under "mappers".

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_6.png)

</center>

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_7.png)

</center>

Finally, the role must be assigned to a user. This can be done by navigating to users -> demo -> role mappings and moving the wanted role from "available roles" to “assigned roles” for the client “aws-credentials-issuer”

![](/images/managing-access-to-multiple-aws-accounts/image_8.png)

## AWS Credentials Issuer Service

### Backend

The [provided code](https://github.com/imduffy15/aws-credentials-issuer) supplies a [lambda function](https://github.com/imduffy15/aws-credentials-issuer/blob/master/lambda_handler.py) which will take care of validating the users JWT token and exchanging it using `AssumeRole` for AWS Credentials.

This code can be deployed to an AWS account by using the [serverless framework](https://serverless.com/) and the [supplied definition](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml). The [definition](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml) will create the following:

* An [IAM role](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml#L36) granting AssumeRole rights to the function

* A [function](https://github.com/imduffy15/aws-credentials-issuer/blob/master/lambda_handler.py#L67) for generating AWS API Credentials

* A [function](https://github.com/imduffy15/aws-credentials-issuer/blob/master/lambda_handler.py#L36) for generating [Amazon Federated Sign-On URL](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html)s 

With [AWSCLI](https://aws.amazon.com/cli/) configured with credentials for the account that the service will run in execute `sls deploy`, this will deploy the lambda functions and return URLs for executing them.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_9.png)

</center>

### Frontend

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_10.png)

</center>

The [provided code](https://github.com/imduffy15/aws-credentials-issuer) supplies a [frontend](https://github.com/imduffy15/aws-credentials-issuer/tree/master/ui) which will provide users with a graphical experience for accessing the AWS Web Console or generating AWS API Credentials.

The frontend can be deployed to an S3 bucket using the [serverless framework](https://serverless.com/). Before deploying it some variables must be modified. In the [serverless definition (](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml#L75)[`serverless.yml`](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml#L75)[)](https://github.com/imduffy15/aws-credentials-issuer/blob/master/serverless.yml#L75), replace "ianduffy-aws-credentials-issuer" with a desired S3 bucket name and modify [`ui/.env`](https://github.com/imduffy15/aws-credentials-issuer/blob/master/ui/.env) to contain your Keycloak and Backend URL as highlighted above. The deployment can be executed with `sls client deploy`. 

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_11.png)

</center>

On completion, a URL in the format of `http://<bucket-name>.s3-website.<region>.amazonaws.com` will be returned. This needs to be supplied to keycloak a redirect URI for the "aws-credentials-issuer" client.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_12.png)

</center>

## Usage

### Browser

By navigating to the URL of the S3 bucket a user can get access to the AWS Web Console or get API credentials which they can use to manually configure an application.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_13.gif)

</center>

### Command line

To interact with the "aws-credentials-issuer" the user must have a valid JWT. This can be done by executing the [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1) against keycloak.

token-cli can be used to execute the  [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1) and generate a JWT token, this can be downloaded from the projects [releases page](https://github.com/imduffy15/token-cli/releases/tag/v0.0.3); alternatively, on OSX it can be installed with homebrew brew install imduffy15/tap/token-cli.

Once token-cli is installed it must be configured to run against keycloak, this can be done as follows:

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_14.png)

</center>

Finally, a token can be generated with token-cli token get aws-credentials-issuer -p 9000. On first run the users browser will be opened and they will be required to login, on subsequent runs the token will be cached or refreshed automatically.

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_15.gif)

</center>

This token can be used against the "aws-credentials-issuer" to get AWS API credentials:

```
curl https://<API-GATEWAY>/dev/api/credentials?role=<ROLE-ARN> \
-H "Authorization: bearer $(token-cli token get aws-credentials-issuer)"
```

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_16.png)

</center>

Alternatively, a AWS [Amazon Federated Sign-On URL](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html) can also be generated:

```
curl https://<API-GATEWAY>/dev/api/login?role=<ROLE-ARN> \
-H "Authorization: bearer $(token-cli token get aws-credentials-issuer)"
```

<center>

![](/images/managing-access-to-multiple-aws-accounts/image_17.png)

</center>
