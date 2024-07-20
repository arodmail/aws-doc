## Amazon Cognito User Pools for CIAM

### Abstract

This guide walks through a step-by-step process to create a basic Amazon Cognito User Pool. The guide covers how to configure a user pool as an OpenID Connect provider. This allows authentication through a username/password, social login, and MFA. The guide also provides instructions to configure a hosted UI that enables user sign-up and sign-in. The security focus of the guide is on issuing tokens using the OIDC Authorization Code flow with Proof Key for Code Exchange (PKCE, pronounced "pixy"), mainly for public clients that cannot securely store a client secret. Finally, the guide uses Postman to get JWTs from the OpenID provider's token endpoint. 

### 1. Intro

Amazon Cognito User Pools is a fully managed user directory for web and mobile applications. It offers features for user sign-up, sign-in, and user profile management. As a managed service, Cognito is backed by AWS cloud-based security and availability features, including MFA, Adaptive Authentication, Compromised Credentials checks, and 99.9% uptime. Cognito provides a customizable, hosted sign-up and sign-in UI, along with social sign-in integration with Facebook, Google, Amazon, and Apple.

#### CIAM 

Cognito is suitable for managing _customer_ identities in a secure and scalable way across digital products, applications, and services. Organizations can set up user pools to manage and secure the identity and access of their customers across digital channels. 

Support for the open standards to integrate with OAuth 2.0, SAML, and OpenID Connect (OIDC) providers for Single Sign-On (SSO) are features included, and Cognito integrates with other AWS services, for example, with Amazon API Gateway offering native support. As such, Cognito allows you to build on its features to implement CIAM (Customer Identity and Access Management) for your customer-facing  applications written in Node.js, Angular, React, and Vue frameworks, by interfacing with authentication and JSON Web Token (JWT) endpoints.

### 2. Overview

From a high level, 3 components are involved in an authorization request with a Cognito user pool's OpenID provider:

- Application - a customer-facing mobile or web application
- User Pool - can be thought of as an authorization server
- External IdP - an alternative, optional identity provider, for federated identities. 

![aws-congito-sequence-overview.png](img%2Faws-congito-sequence-overview.png)


High Level Steps

To obtain a JSON Web Token from a Cognito user pool that can be used to authorize access to an application's backend services:

1 - Request Authorization

- The application sends an authorization request to the user pool's `/oauth2/authorize` endpoint. 
- The user pool responds with a redirect its `/login` endpoint, causing the application to launch the hosted UI and prompt the user for their credentials. 
- The user submits their credentials through the hosted UI, and the user pool validates them against a local directory, or against an external IdP. 

2 - Authorize

- If the credentials are valid, the user pool issues an Authorization Code and sends it along with a redirect to the application's callback URL. 
- The application's callback URL exchanges the Authorization Code for a token by sending the code to the user pool's `/oauth2/token` endpoint. 
- The user pool's token endpoint responds with one or more JWT. 

This is a generalized overview. For a more detailed flow, see [Sequence Diagram (Flow)](#5-sequence-diagram-flow) below. 


### 3. Create User Pool Process

A user pool in Cognito is a user directory. It stores user profile attributes such as name, email, phone number, and custom attributes. The following steps collect the settings and configuration parameters to create a user pool:  

* Step 1 - Configure Sign-In experience
* Step 2 - Configure security requirements
* Step 3 - Configure Sign-Up experience
* Step 4 - Configure message delivery
* Step 5 - Integrate your app
* Step 6 - Review and create

#### Step 1 - Configure Sign-In experience

In the first step, select how users can sign-in to your pool. To simplify the process, skip the social sign-in options. Select `User name` and `email` as attributes that can be used to sign in. 

![aws-cognito-step-1.png](img%2Faws-cognito-step-1.png)

#### Step 2 - Configure security requirements

This step allows you to customize password rules, enable MFA, and to define  account recovery options. Use the Cognito defaults for password rules, and select _Optional MFA_ with _Authenticator Apps_ and _SMS messages_ options checked. 

![aws-cognito-step-2.png](img%2Faws-cognito-step-2.png)

#### Step 3 - Configure Sign-Up experience

This step enables self-registration if you want anyone on the internet to sign-up for your application's services. Here you can define how attribute verification should work, for example to send a verification code to a user's email address to verify their email. User's can only sign in with verified attributes. Select _Enable self-registration_, and _Cognito-assisted verification_. Leave other values as defaults. 

![aws-cognito-step-3.png](img%2Faws-cognito-step-3.png)

#### Step 4 - Configure message delivery

Setup email notification. Select _Send email with Cognito_. 

![aws-cognito-step-4.png](img%2Faws-cognito-step-4.png)

#### Step 4.1 - Create a new IAM role

Enter a role name for a new AWS IAM Role that will be granted permission to send SMS messages. Enter `cognito-sms-role`. 

![aws-cognito-step-4-1.png](img%2Faws-cognito-step-4-1.png)

#### Step 5 - Integrate your app

The application specific settings for the user pool are defined in this step, which creates an application client. 

#### Step 5.1 - User pool name

Provide a name for your new pool, for example, `cognito-user-pool`. 

![aws-cognito-step-5-1.png](img%2Faws-cognito-step-5-1.png)

#### Step 5.2 - Hosted authentication pages

We're going to enable the Hosted UI for our tests later, so check the option to _Use the Cognito Hosted UI_, and _Use a Cognito domain_. This sets up OAuth and hosted UI endpoints through an `amazoncognito.com` domain. For the domain prefix enter `auth-tester`

![aws-cognito-step-5-2.png](img%2Faws-cognito-step-5-2.png)

#### Step 5.3 - Initial app client

Select _Confidential client_ in this step, and make sure the _Generate a client secret_ option under _Client Secret_ is selected. Specify a client name, for example, `cognito-confidential-client`. And  specify a callback URL under _Allowed callback URLs_. In this guide this value is required but stubbed out for testing purposes only. It does not need to be a valid URL that accepts and processes callback requests. Enter URL: `https://localhost`.

![aws-cognito-step-5-3.png](img%2Faws-cognito-step-5-3.png)

#### Step 6 - Review and create

Confirm you selections, and click "Create user pool" at the bottom of the page. 

![aws-cognito-step-6.png](img%2Faws-cognito-step-6.png)

#### Step 6.1 - User pool created successfully

With your new user pool created, let's verify the basic settings. 

![aws-cognito-step-6-1.png](img%2Faws-cognito-step-6-1.png)

### 4. Verification

View the user pool and go to the _App integration_ tab. Scroll to the bottom and select the `cognito-confidential-client` under _App clients and analytics_. Here you can see the _Client ID_ and _Client secret_ values that are used in the Postman demo later. 

![aws-cognito-verify-1.png](img%2Faws-cognito-verify-1.png)

#### Step 4.1 Verify Hosted UI

Scroll down to the _Hosted UI_ section and click _View Hosted UI_. This launches the login screen your users will be presented with to enter their credentials, to sign-up for your application, and/or to recover their account password through a "Forgot your password" link. 

![aws-cognito-verify-hosted-ui.png](img%2Faws-cognito-verify-hosted-ui.png)

#### Step 4.2 Verify OpenID Config

The meta-data for the OpenID provider exposed by your user pool is available through:

```
https://cognito-idp.us-west-1.amazonaws.com/us-west-1_LtLR26KRq/.well-known/openid-configuration
```

Or replace your AWS `region` and `user-pool-id` in the URL: 

```
https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/openid-configuration
```

Response

```json lines
{
  "authorization_endpoint": "https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/authorize",
  "end_session_endpoint": "https://auth-tester.auth.us-west-1.amazoncognito.com/logout",
  "id_token_signing_alg_values_supported": [
    "RS256"
  ],
  "issuer": "https://cognito-idp.us-west-1.amazonaws.com/us-west-1_LtLR26KRq",
  "jwks_uri": "https://cognito-idp.us-west-1.amazonaws.com/us-west-1_LtLR26KRq/.well-known/jwks.json",
  "response_types_supported": [
    "code",
    "token"
  ],
  "revocation_endpoint": "https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/revoke",
  "scopes_supported": [
    "openid",
    "email",
    "phone",
    "profile"
  ],
  "subject_types_supported": [
    "public"
  ],
  "token_endpoint": "https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/token",
  "token_endpoint_auth_methods_supported": [
    "client_secret_basic",
    "client_secret_post"
  ],
  "userinfo_endpoint": "https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/userInfo"
}
```

In the meta-data, the following endpoints are used later to setup Postman to interface with your OpenID provider:

| Endpoint               | URL                                                                   |
|------------------------|-----------------------------------------------------------------------|
| Authorization Endpoint | `https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/authorize` |
| Token Endpoint         | `https://auth-tester.auth.us-west-1.amazoncognito.com/oauth2/token`     |

### 5. Sequence Diagram (Flow)

Before moving on, let's explore in detail the step-by-step sign-in flow, the components involved, and their interactions. 

![aws-cognito-sequence-diagram.png](img%2Faws-cognito-sequence-diagram.png)

In this flow, there are 4 main components: the user's web browser on a mobile device or laptop, the application's backend, a Cognito user pool, and an External IdP for social sign-in, other federated identity store, or SAML providers. 

##### Steps

1 - A user through a web browser initiates the process by proceeding to login, or by selecting an IdP from a login screen. The application sends a request to the authorization endpoint of the OpenID provider exposed by the Cognito user pool.

2 - For sign-in as a user in the user pool, the response from the authorization endpoint is a redirect to the Self Hosted UI. The redirect is followed by the browser, and the Self Hosted UI is returned and displayed, where the user enters their username/password and clicks the Sign-In button to POST their credentials to the OpenID provider's login endpoint. 

OR,

3 - For social sign-in, federated identity, or SAML authentication, the user is redirected to the IdP login screen, where they enter their username/password and POST their credentials to the IdP sign-in endpoint. If the credentials are recognized by the IdP, then an authorization code or a SAML assertion is returned.  

4 - In this step, the authentication process is complete. For Single Sign-On, a session cookie is set at this point, valid for 1 hour. Next, given a PKCE Authorization Code, the application's callback URL exchanges the code for a set of JWTs by sending a POST request to the OpenID provider's token endpoint. The access token returned represents a user's session with an expiration, valid for 60 seconds or less and can be used to authorize access to the application's backend services. This token can be refreshed to extend the session, using a refresh token. 

5 - Finally, the user is authenticated and authorized into the application's content, which is sent to the user's browser. If the same user attempts to access another application in the same Cognito user pool (or same domain), then the SSO session cookie, while still valid, allows the user to skip having to sign-in again. 

### 6. Demo/Test - Postman

#### 6.1 Overview

This part of the guide uses Postman to interface with the OpenID provider exposed by your new Cognito user pool. 

#### 6.2 Configure New Token

First, create a new request, but leave the URL blank. Go directly to the Authorization tab. Select _Authorization Code (with PKCE)_ for the _Grant Type_, and enter values for Callback URL, Auth URL, Access Token URL, Client ID, and Client Secret. These values are available in Steps 4 and 4.2 above.  

![aws-cognito-postman-auth-1.png](img%2Faws-cognito-postman-auth-1.png)

#### 6.3 Launch Self-Hosted UI

Click "Get New Access Token". The Self-Hosted UI should appear in a new window. We don't have any users in our user pool yet, so let's create a new user by going through the user registration flow. Select _Sign up_.

![aws-cognito-postman-auth-2.png](img%2Faws-cognito-postman-auth-2.png)

#### 6.4 User Sign-Up

Provide a Username, a valid email address you have access to, and a password that satisfies the default password rules. Click Sign up. 

![aws-cognito-postman-auth-3.png](img%2Faws-cognito-postman-auth-3.png)

#### 6.5 Confirm Account

In this step, you are prompted for a Verification code which was sent to the email address provided in the previous step. Get the code from the email message, enter it here and click Confirm account. This code is valid for a set time period. If the time period passes, then you can  request a new code. 

![aws-cognito-postman-auth-4.png](img%2Faws-cognito-postman-auth-4.png)

#### 6.6 Authentication Complete

You should see a dialog confirming a new access token was received. Click _Proceed_. 

![aws-cognito-postman-auth-5.png](img%2Faws-cognito-postman-auth-5.png)

#### 6.7 Authentication Complete - Tokens

The tokens associated with the authorization request are displayed by Postman in a new window. Here you will find an ID Token, Access Token, Refresh Token, and the expiration time. 

![aws-cognito-postman-auth-6.png](img%2Faws-cognito-postman-auth-6.png)

#### 6.8 User Sign-Up and Authentication Sequence

Open the Console in Postman to review the sequence of HTTPS requests that comprise the new user sign-up flow including a token request. 

![aws-cognito-postman-auth-8.png](img%2Faws-cognito-postman-auth-8.png)

#### 6.9 User Added to Pool

To view the new user in the user pool, go back to the AWS Console > Amazon Cognito > view your pool, and the new user should be listed under the _User_ tab. Note that the user's _Confirmation status_ is "Confirmed" because of the email verification step during sign-up. 

![aws-cognito-postman-auth-7.png](img%2Faws-cognito-postman-auth-7.png)

### 7. Demo/Test - User Sign-In Sequence

#### 7.1 Overview

To try the sign-in flow for your new user through Postman:

- Clear cookies
- Clear console
- Get new Access Token
- Enter Username + Password in the Self Hosted UI.


- ![aws-cognito-postman-auth-9.png](img%2Faws-cognito-postman-auth-9.png)

Open the Postman Console to see the sequence of HTTPS requests that comprise the user sign-in flow:

![aws-cognito-postman-auth-10.png](img%2Faws-cognito-postman-auth-10.png)

For a diagrammatic representation of this flow see "Sequence Diagram (Flow) above". 

#### 7.2 Step 1 - Authorization Endpoint (GET)

The first step sends a GET request to the OpenID provider's `/oauth2/authorize` endpoint. Note the `location` Response header redirects the application to the `/login` endpoint which displays the Self Hosted UI. Also note that the Response Code is 302. 

![aws-cognito-postman-auth-11.png](img%2Faws-cognito-postman-auth-11.png)

The application initiates the flow by sending the following URL parameters:

| Parameter               | Description                                                                                                                                           |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| `response_type=code`    | Indicates that the client is requesting an authorization code.                                                                                        |
| `client_id`             | The client identifier issued by the OpenID Provider.                                                                                                  |
| `redirect_uri`          | The URI to which the authorization server will send the user after granting or denying access. This is the Callback URL we defined in Step 5.3 above. |
| `scope`                 | Optional. Specifies the scope of the access request, which includes `openid` to indicate OpenID Connect authentication.                               |
| `code_challenge`        | A URL-safe base64-encoded SHA256 hash of a randomly generated `code_verifier`.                                                                        |
| `code_challenge_method=S256` | Indicates that the SHA256 method is used to hash the `code_verifier`.                                                                                 |


#### 7.3 Step 2 - Redirect to Self-Hosted UI (GET)

This step simply follows the redirect from the previous response, to launch the Self Hosted UI and prompt for Username / Password values. 

![aws-cognito-postman-auth-12.png](img%2Faws-cognito-postman-auth-12.png)

#### 7.4 Step 3 - Submit User Credentials (POST)

When the user clicks the _Sign In_ button in the Self Hosted UI, a POST request is sent to the OpenID provider's `/login` endpoint to submit the credentials entered through the Self Hosted UI. The Response Code from this request, if the credentials are valid, is 302 with a redirect to the application's Callback URL, including the PKCE Authorization Code.

![aws-cognito-postman-auth-13.png](img%2Faws-cognito-postman-auth-13.png)

Response Header

```
location: https://localhost?code=d89066dd-cff4-423f-811a-1164d6161b4b
```

#### 7.5 Step 4 - Token Endpoint (POST)

In this final step, the last in the sign-in flow, the Authorization Code is exchanged for a JWT. A POST request is sent to the OpenID provider's `/oauth2/token` endpoint that includes the auth code and the URL to redirect to within the application. 

![aws-cognito-postman-auth-14.png](img%2Faws-cognito-postman-auth-14.png)

Request Body

```
grant_type: "authorization_code"
code: "d89066dd-cff4-423f-811a-1164d6161b4b"
redirect_uri: "https://localhost"
code_verifier: "7NUjQzeZ7_hmI9zGurufbT0XyjGoQWXe9p_6iQDTLnI"
```

Response Body

The response body is a JSON document with the following tokens:

```
{
  "id_token" : "eyJraWQiOiI3YkV5V3FjVE5EW ...",
  "access_token" : "eyJraWQiOiJVSkJUcEZVZjAwU ...",
  "refresh_token" : "eyJjdHkiOiJKV1QiLCJlbmMiO ..." 
  "expires_in" : 3600,
  "token_type" : "Bearer"
}
```

The application's Callback URL may be designed securely to accept these tokens to grant the authorized  user access to the application's backend services. JWT's may also include claims that tell the application how to make more granular authorization decisions about access. 


### 8. Conclusion

PKCE in Amazon Cognito User Pools mitigates the risk of authorization code interception attacks by ensuring that the code exchanged for tokens is verified with a unique proof key. This is particularly useful for public clients, such as mobile and single-page applications, that cannot securely store client secrets.

Amazon Cognito User Pools is a robust  Customer Identity and Access Management (CIAM) solution for managing customer identities across digital platforms. By stepping through the creation and configuration of a basic user pool and trying out the OpenID provider's endpoints through Postman, this guide is intended to help you get started quickly and to apply the concepts in real-world CIAM scenarios. 

### 9. Links

The OAuth 2.0 Authorization Framework

https://datatracker.ietf.org/doc/html/rfc6749

The OAuth 2.0 Authorization Framework: Bearer Token Usage

https://datatracker.ietf.org/doc/html/rfc6750

OpenID Authentication 2.0

https://openid.net/specs/openid-authentication-2_0.html

CIAM - Customer Identity and Access Management

https://aws.amazon.com/what-is/ciam/
