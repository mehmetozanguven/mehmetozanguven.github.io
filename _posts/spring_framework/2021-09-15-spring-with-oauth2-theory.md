---
layout: post
title: Spring Boot with OAuth2 - Theory
date: 2021-09-15 22:45:31 +0530
categories: "spring"
author: "mehmetozanguven"
---

In this post, we are going to learn **what the oAuth2 is, how to use OAuth2 in Spring boot**. We will start with why do we need a OAuth2, we will look at the possible implementations of OAuth2 and finally we will talk about possible security issues when using OAuth2.

<nav class="custom-table-of-contents">
<hr class="horizontal-line">
  <h4 class="table-of-contents-title">Contents</h4>
  * this unordered seed list will be replaced by toc as unordered list
  {:toc}
 <hr class="horizontal-line">
</nav>

## What is the OAuth2

OAuth 2.0 is the industry-standard protocol for authorization.

OAuth 2.0 focuses on client developer simplicity while providing specific authorization flows for web applications, desktop applications, mobile phones, and living room devices.

Overall **OAuth2 primary purpose is to allow a third-party website or app to access to a resource**

> Quick Notes:
>
> - **Authorization** is the process of giving the user permission to access a specific resource or function.
> - **Authentication** is the process of validating users whom they claim to be.

## Why do we need OAuth2

There must be a reason to use one tool instead of another. OAuth2 can be useful because:

- We don't want to store user's credential (I mean, user's password) in our system.
- If we have many small applications (let's say N applications) and if each application can be accessed with authenticated users, then we will have N separate tables to store each logged-in users/customers

If you look at the picture below, same client username and password is stored in the three different applications with independently. If you work in the organization, this will be hard to maintain.

<img src="/assets/spring/oauth2/independent_credentials.png" alt="independent_credentials.png" />

Therefore it would be better if we isolated the responsibility for credential management in one component. We can call this component **authorization server**:

> With this way we eliminate the duplication of user's credentials.

<img src="/assets/spring/oauth2/auth_example.png" alt="auth_example.png" />

## The components of OAuth2

It is essential to know components of OAuth2 before diving into implementation.

- The **resource server**: The application which is hosting resources owned by users. Resources can be users' data.
- The **user (resource owner)**: The individual who owns resources. A user represents the username and password to identify itself.
- The **client**: The application that accesses the resources owned by the user on user behalf.
  - The client uses ClientID and a secret key to identify itself. But these are not the same as the user credentials.
- The **authorization server**: The application that authorizes the client to access the user's resources. When the authorization server decides that a client is authorized to access, it creates(or issues) a token.
  - Then client uses this token to prove to the resource server that it was authorized by the authorization server.
  - The resource server allows the client to access the resource if it has a valid token.

This all processes can be easily understood by a picture:

<img src="/assets/spring/oauth2/auth_components.png" alt="auth_components.png" />

## Implementations with OAuth2

OAuth2 provides multiple possible authentication flows and we can choose the one which applies to our case. In general OAuth2 will provide us a token for authorization. Once we obtain the token, we can access specific resources. Multiple possible authentication provides multiple way to get token which is called **grants**.

Here are the most common grants OAuth2 provides:

- Authorization code
- Password
- Client credentials

Let's discuss these by one by.

### Grant Type - Authorization Code

All process can be described in the flow graph:

<img src="/assets/spring/oauth2/grant_auth_code.png" alt="grant_auth_code.png" />

> In detail, when client says "Via AuthServer, please allow me ..." what it is happening is that client will redirect the user(you) to the authorization server login page.
>
> When credentials, provided by you, was correct, authorization server will call the client's url called **redirect URI**

Overall there are 3 steps:

1. Make the authentication request
2. Obtain an access token
3. Call the protected resource

#### Make the Authentication Request - Step 1

The client redirects the user to an endpoint of the authorization server. For instance, this is happening when you click "Sign In with Google" on any websites

Be aware of that users will interact directly with the authorization server. Users won't send the credentials to the client.(web or mobile app)

Client calls the authorization endpoint with the following details in the request:

- **`response_type=code`**: indicates that client expects a code. The code will be used to obtain an access token.
- **`client_id=ID`** : This ID identifies client application.
- **`redirect_uri`**: tells the authorization server where to redirect the user after successful authentication.
- **`scope`**: related to the granted authorities
- **`state`**: defines a cross-site request forgery (CSRF) token used for the CSRF protection.

After successful authentication:

- The AuthServer calls back the client, specified by redirect URI, and provides a **code** and the **state** value.
- The client checks the state value is the same as the one it send. (To confirm that no one else attempting to call the redirect URI

#### Obtain an access token - Step 2

Step 1 is a proof that the user authenticated.

After the step1, the client will call the AuthSever with the **code(first token)** to get the token(second token).

Let's be clear:

- The first code is a proof that the user directly interacted with the AuthServer.The client receives this code and use it again to authenticate and to obtain an access token(second token)
- The client uses the second token to access resources on the resource server.

You may ask "why AuthServer didn't return the token(access token) from the step 1? If AuthServer was return the token for the first time, the client wouldn't need to call the second time":

Actually there is flow called **implicit grant type** where AuthServer returns the access token with the first call. But nowadays, almost all known authServers doesn't allow us to use implicit grant type because it has security problem:

- AuthServer, using the code(first token), is making sure that it was indeed the right client.
- By sending the code, the client proves who it is by using their credentials.

To get an access token, client makes a call and sends:

- The authorization code, which states that user authorized them
- Their credentials, which states that same client is sending request.

Overall, in the step 2, client makes a request to the AuthServer. And the request contains:

- **`code`**: it is the Authorization code from the step 1. (It is a prove that the user authenticated)
- **`client_id` and `client_secret` **, these are the client's credentials
- **`redirect_url`**: same as the step 1
- **`grant_type`**: this is the value of `authorization_code`. Defines the flow we want to follow.

As a response, the server sends back an **`access_token`**. The client will use this token to call resource server.

#### Call the protected resource - Step 3

Right now, the client can call for the protected resource. The client uses an access token in the authorization request header when calling an endpoint of the resource server.

---

### Grant Type - Password

Now let's discuss **password grant type**.

> This grant_type is less secure than the authorization code grant type. Because user will share his/her credentials with the client.

<img src="/assets/spring/oauth2/password_grant_type.png" alt="password_grant_type.png" />

In this flow, the client collects the user credentials and uses these to authenticate and obtain an access token from the AuthServer

This flow generally is used when the authServer is maintained by the same organization. In the same organization, if we use Authorization code grant type, then for each logged-in requests users will be re-directed to the login page of the organization and back again. Password grant type can be a good choice for these scenarios.

There are two tasks to perform:

- Request an access token
- Use the access token to call resources

#### Request an acces token

It is simple. The client collects the user credentials and calls the authorization server to obtain an access token. The client also sends the following :

- **`grant_type`**: this is the value of password grant type
- **`client_id` and `client_secret`**: these are the client's credentials
- **`scope`**: related to the granted authorities
- **`username` and `password`**: these are the user's credentials. (sent as plain text)

As a response, client receives an access token.

#### Use access token to call resources

Once the client has an access token, it can use the token to call the endpoints on the resource server.

---

### Grant Type - Client Credentials

Let's discuss **client credentials grant type**

<img src="/assets/spring/oauth2/client_credentials_grant_type.png" alt="client_credentials_grant_type.png"/>

As you can see there is no actor like user in the flow.

This flow can be useful when only machine-to-machine communication is required. There are two steps:

- Request an access token
- Use the access token to call resources

#### Request an access token

Client will send a request with the following ones:

- **`grant_type`**: this is the value of client credentials grant type
- **`client_id` and `client_secret`**: these are the client's credentials
- **`scope`**: related to the granted authorities

In response, client receives an access token.

#### Use the access token to call resources

The client will use the token to call endpoints on the resource server.

---

## Refresh token with OAuth2

As a final step, let's discuss the token itself.

Be aware of that, OAuth2 doesn't force us to implement specific token. Even token can have an infinite expiry time. But as general, tokens generated from AuthServer has short live (as possible).

<img src="/assets/spring/oauth2/refresh_token.png" alt="refresh_token.png" />

When the client has a refresh token, the client sends a request with the following:

- **`grant_type`**: this is the value of refresh token
- **`client_id` and `client_secret`**: these are the client's credentials
- **`refresh_token`**: this is the refresh token
- **`scope`**: related to the granted authorities

In response, the authorization server returns a new access token and a new refresh token.

## Security in OAuth2

Of course, OAuth2 is not bulletproof solution. There can be some security problem while using OAuth2:

- CSRF attacks: After logged-in, if there is no protection for CSRF attacks then we will be vulnerable to run action which was not started by the legitimate user(s)
- Stolen client credentials: If someone stoles the client credentials (assume that your google credentials, you are developing spring boot application with Sign On with Google, was stolen by someone else), we are basically in the dead situation :)
- Replaying token: If someone intercepts the request, then they can also intercepts the tokens

## Conclusion

In this article, we looked at the what is the OAuth2 and theory of the OAuth2 itself.

We looked at the different implementations of OAuth2:

- Authorization Grant Type: Most common used one. You will use this one when implementing Single Sign On with Gmal/Github/Facebook etc..
- Password grant type: Used when the authServer is maintained by the same organization. Also it is less secure than the Authorization Grant Type
- Client credentials grant type: Useful when only machine-to-machine communication is required.

We also looked at the refresh token and its purpose. Basically refresh token is used when the access token is expired and it is useful because we don't want to redirect users to login page every time when token is expired.

We also know that OAuth2 doesn't force us to implement specific token. Even token can have an infinite expiry time.

As every software, OAuth2 has also some security issues when it is not implemented in the right way. Some security concerns:

- CSRF: someone can do an action on behalf of the legitimate user(s)
- Stolen client credentials: whatever you do, protect your client's credentials, otherwise you are dead.

In the next article, I will do an OAuth2 example with Spring Boot.
