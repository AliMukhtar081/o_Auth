# Welcome to Our Auth2 Project Guide

Hello and welcome to our guide on how our project implements authorization and authentication using OAuth2.0. This guide is designed to give you a high-level understanding of how our system works and ensures that your data remains secure.

## What is OAuth2.0?

OAuth2.0 is a standard that app developers use to allow their users to securely access data without having to give their password. You have probably used OAuth2.0 without realizing it whenever you've clicked on a "Sign in with Google" or "Sign in with Facebook" button on a website. This is OAuth2.0 in action!

## How Our Project Uses OAuth2.0

We use OAuth2.0 to allow you, our client, to access our resources securely. Here's how it works in simple terms:

- **Authorization**: First, your application (Client) contacts our server (Server) and requests access. We verify that your application is authorized and send you an "authorization code".
- **Token Generation**: Next, your application uses this authorization code to request a "token". This token is like a temporary key that allows your application to access our resources.
- **Authentication**: Finally, your application uses this token to request resources from our server. We verify the token and, if it's valid, we send you the resources you requested.

## Refreshing the Token

Sometimes, a token can expire, which means it can no longer be used to access resources. To address this, we provide a "refresh token" which your application can use to get a new, valid token.

## How We Store Data

We use a database to store crucial information for the OAuth2.0 process. This includes details about the client (your application), the user, authorization codes, and tokens. We ensure this data is stored securely and is only accessible when necessary for the authorization process.

## Making Requests

Our system provides specific ways for your application to request authorization, tokens, and access to resources. These involve sending specific information to our server via URLs (web addresses). We've designed this process to be as straightforward as possible, and we're always here to help if you have any questions.

# Table of Contents

1. [Installation and Setup](#install)
1. [Important Links](#links)
1. [Oauth2.0 Things to Know](#thingsToKnow)
   - [State](#thingsToKnow-state)
   - [Scope](#thingsToKnow-scope)
1. [Flows](#flow)
   - Authorization Code Grant
     - [0. Overview](#flow-overview)
     - [1. Authorization](#flow-authorization)
     - [2. Token](#flow-token)
     - [3. Authentication](#flow-authentication)
   - Refresh Token
     - [0. Overview](#refresh-overview)
1. [Database](#database)
   - [Client](#database-authorization)
   - [User](#database-user)
   - [Authorization Code](#database-code)
   - [Token](#database-token)
1. [URL Queries](#url)
   - [Authorization Code](#url-code)
   - [Token](#url-token)
   - [Access Protected Resource](#url-resource)

<a id='install'></a>

# Installation and Setup

1. Clone this Repo
1. `cd` into the project root folder, and run `yarn`
   - If `yarn` is not installed, install it and then run `yarn`
1. Run `yarn authServer` to boot up the oauth 2.0 server
1. Run `yarn devAuth` to boot up the oauth 2.0 server in dev mode. This will enable hot reloading when your code changes.
1. Run `yarn test` to run unit tests that cover all implemented grants
   - For verbose output, modify `level` in `auth/tests/setup.js` to be `DebugControl.levels.ALL`

[back](#top)

<a id='links'></a>

# Important Links

Checkout
[Oauth-server-github](https://github.com/oauthjs/node-oauth2-server)
if you are running into any weird errors. Tutorials are seriously lacking
on this implementation of the Oauth2.0 protocol as most people use
an external service for it. Luckily, the errors are pretty specific,
so you can go through their code to figure out what is happening.

Also, if you want to see how the middleware is generated, checkout
[this](https://github.com/oauthjs/express-oauth-server)
to see the middleware stuff. Their examples are out of date, so
ignore them.

[back](#top)

<a id='thingsToKnow'></a>

# Things To Know

OAuth has a few parameters that are important to understand. Here is a list of good things to know:

<a id='thingsToKnow-state'></a>

### State

State is an optional string provided by the client. It helps the client to protect against Cross Forgery requests and should not be used to transmit private data as it may be openly exposed and changed.

<a id='thingsToKnow-scope'></a>

### Scope

<a id='flow'></a>

# Flow

First, some definitions and reminders:

- _Client_: The application wanting to have access to your resources
- _User_: The person wanting to use your resources on the Client
- _Authorization_: The process of determining whether something has access to protected resources
- _Authentication_: The process of determining a person's identity.
- _OAuth2.0_: A protocol outlining how authorization should happen. It is NOT an authentication library. You will need to provide that yourself.

Each of the grants provide a token which enables the user to access resources like the following diagram shows:

1. Token is passed up in the authorization header
2. Oauth Server validates the token
3. Protected Resources are sent back down

![Protected Resources](/resources/images/ProtectedResources.png)

[back](#top)

### Authorization Code Grant

<a id='flow-overview'></a>

##### 0. Overview

Aight, with those out of the way, we need to cover the basic flow with the authorization code grant.

1. Authorization
   - Client application contacts the Server and requests access
   - Client application provides a client_id (unique string identifier)
   - Client provides a redirect uri to send the user after the code is delivered
   - Client may provide user data for authentication purposes
   - Server validates information and sends back an authorization code
2. Token
   - Client uses received authorization code to request a token
   - Client sends client_id, client_secret (if applicable)
   - Server validates request, and sends a token.
3. Authentication
   - Client uses token to gain access to Server's protected resources

![Authorization Code Grant Flow](/resources/images/AuthorizationCode.png)

In the OAuth2.0 library, each of the above areas are handled within
dedicated urls. Specific details on how to handle the different things
are added to the `model` object within the OAuth server.

Now, we will explore each of the above 3 areas in depth.

[back](#top)

<a id='flow-authorization'></a>

##### 1. Authorization

After hitting the Authorization url, the following calls are made within
the model object in this order:

1. `getClient`: This will extract the `client_id` from the request's query or body parameter. You can then go through and verify that the client is indeed a good client, what redirects are permitted for the Client, and what grants they have access to (in this example, just 'authorization_code'). Upon verification, return a valid Client object.
   - After calling `getClient`, if you passed an `authenticateHandler` to the `authorize` method call, this will then be called. If you return some non-falsy object, that object will be the User and will assume the User was able to authenticate. If you return a falsy object, we will assume that the user failed to authenticate, and must be booted. So, in short, this is where you authenticate the user.
1. `saveAuthorizationCode`: This will give you an authorization code, the retrieved Client Object from `getClient`, and the user from the `authenticateHandler`. This information needs to be stored in your database. Once stored, return the information

After making the above calls, the server redirects you to the provided `redirect_uri` with the authorization code present as a url query parameter.

[back](#top)

<a id='flow-token'></a>

##### 2. Token

After hitting the token url, the following calls are made within the model object in this order:

1. `getClient`: Same as before, but will now allow you set the `client_secret` to ensure the client is a valid client.
1. `getAuthorizationCode`: using the `authorizationCode` the client provides, look up in the database for the client, user, and code information for that code, and return it. If none, return false to stop the request as it is invalid.
1. `revokeAuthorizationCode`: using the `authorizationCode` the client provides, delete from the database where the code exists. Return true if the code was deleted, false otherwise. Each authorization code can only be used once.
1. `generateAccessToken (optional)`: using the client and user provided, generate an access token. If not provided, the library will use its in-built library to generate a token. If you want to use JWTs, this is where that would happen.
1. `saveToken`: using the client, code, and token generated earlier, save this information in the database.

The token is then sent as a json response like this:

```js
{
  access_token: "38a72b9262f931a74377dc4f8c0d1d906a89af35",
  token_type: "Bearer",
  expires_in: 86399
}
```

[back](#top)

<a id='flow-authentication'></a>

##### 3. Authentication

Use the token type and token code to add an authorization header like this: `${token_type $token_code}`. This will allow the token to be transmitted securely.

After hitting an authenticate url, the following calls are made within the model object in this order:

1. `getAccessToken`: using the token code provided by the client, return the token, client, and user associated with the token code. If not valid, return false.

If you want to access this information in your routes, it is found in `res.locals.oauth.token`, so you immediately have access to the client and user information associated with the token.

[back](#top)

<a id='refresh-overview'></a>

### Refresh

##### Overview

The refresh token flow is one of the simplest of the grants. After any successful grant flow is completed and a token is generated, a refresh token is created along-side. If the refresh token is then returned with the other information, the client will be able to use the `refresh_token` with its `client_id`, `client_secret`, and `grant_type` of refresh_token in a post to the /token route to get access to a new valid token.

![Refresh Token Gran](/resources/images/RefreshToken.png)

<a id='database'></a>

# Database

There are four tables that should be stored in the database:

- Client
- User
- Authorization Code
- Token

The server will make use of the stored value in these for authorization and authentication purposes. Ideally, the database system used would be promise-based such that they can just be returned. If this is not available, you can make use of the callback parameter in each of the model functions.

[back](#top)

<a id='database-client'></a>

### Client

This stores information on the client.

- id: unsigned long primary key auto_increment. // For Relations
- client_id: String unique // Client knows
- client_secret: String // Client knows
- data_uris: [String] // Array of acceptable redirect locations
- grants: [String] // Array of grants client has access to (authorization_code being one)

[back](#top)

<a id='database-user'></a>

### User

This stores information about the user.

- id: unsigned long primary key auto_increment. // For Relations
- Anything else you want / need for your server

[back](#top)

<a id='database-code'></a>

### Authorization Code

This stores information related to the authorization code

- authorization_code: String primary key // string with a valid code
- expires_at: Date // When the code expires
- redirect_uri: String // String with a valid uri
- client_id: unsigned long references Client(id)
- user_id: unsigned long references User(id)

[back](#top)

<a id='database-token'></a>

### Token

This stores information related to your tokens

- access_token: String primary key // string with a valid access token
- access_token_expires_at: Date // When token expires
- client_id: unsigned long references Client(id)
- user_id: unsigned long references User(id)

[back](#top)

<a id='url'></a>

# URL Queries and Formatting

Once everything is set up the way you want it, you are ready to start making requests to the server. As a reminder, there are three categories of requests that are possible:

1. Get Authorization Code
1. Get Token
1. Get Access to Protected Resource

This section will outline how each of these requests ought to be formatted to successfully go through.

[back](#top)

<a id='url-code'></a>

### Authorization Code

The request for an authorization code requires the following information:

- client_id // The unique string identifying a client
- redirect_uri // The place to redirect after receiving the code
- response_type // what the client is expecting. Should be `code`

These parameters can be included within the body of a POST request, or be sent as URL Query Parameters like this: `/request/authorization?client_id=<ID>&redirect_uri=<URL>&response_type=code`

You can additionally send up other information to help validate the user within the authentication handler.

You know this has handled things successfully when you redirect to the uri you provided.

[back](#top)

<a id='url-token'></a>

### Token

The request for an access token requires the following information:

- client_id // Unique string of client
- client_secret (if applicable) // client secret key
- grant_type // authorization_code in this example

The request should additionally have the following header:

`'Content-Type': 'application/x-www-form-urlencoded'`

and the data should be provided within the body of a post request.

This will send back a json response as outlined earlier.

[back](#top)

<a id='url-resource'></a>

### Access Protected Resource

Requesting access to protected resources consists of making the request as usual, but adding the following header:

```js
{
  Authorization: `${tokenType} ${token}`,
}
```

with the tokenType and token coming from the json response in the token request.

[back](#top)
