Burp Suite
====================

Introduction
-------------

### On my Cybersecurity quest I've been given the advise to get familiar with Burp Suite, for the ability it gives to fiddle with a website and help to detect and exploit diverses vulnerabilities.

### I'll probably use a lot of the content of Portswigger Academy, the author of Burp Suite.

Path traversal
---------------

Path traversal enable an attacker to read aritrary files on the server running an application, like:

- App code, settings and data.
- Credentials for back-end and external systems.
- Sensitive OS files.

Sometimes, an offensive actor might even be able to write arbitrary files on the server, enabling application modification of data and behavior, and in the end taking full control of the system.



API testing
------------

### To start API testing, you need first to gather as much information as possible about the API, to discover its attack surface.

#### API recon

To begin, let's identify API endpoints.
Then, let's find out

- The input data the API processes, both compulsory and optional parameters.

- The types of requests accepted, like supported HTTP methods and media formats

- The rate limits and the authentication mechanisms.

#### API documentation

APIs are usually documented. Documentation can be found in both in human-readable and machine-readable forms. It's often publicly available. If you can acces it, start the recon by reviewing the documentation.

#### Discovering API documentation

You can look if some documentation is accessible by scanning it or manual search.

look for exemple in here:

- /api
- /swagger/index.html
- openapi.json

Once identified an endpoint, investigate the base path:

- /api/swagger/v1
- /api/swagger
- /api

You can also use a list of common paths to find documentation using Intruder.

#### Machine readable documentation

You can use a range of automated tools to analyze any machine-readable API documentation that you find.

You can use Burp Scanner to crawl and audit OpenAPI documentation, or any other documentation in JSON or YAML format. You can also parse OpenAPI documentation using the OpenAPI Parser BApp.

You may also be able to use a specialized tool to test the documented endpoints, such as Postman or SoapUI.

#### Identifying API endpoints

Think also to look at the JS files, the often use API calls. You can use the JS Link Finder BApp and review the JS files directly.

#### Interacting with API endpoints

Now that you've identified endpoints, interact with them using Burp Repeater and Burp Intruder.

Investigate by exemple the API response to changing the HTTP method and media type.

Review error messages and other response closely, They can include information that you can use to construct a valid HTTP request.

You can use the built-in HTTP verbs list in Burp Intruder to automatically cycle through a range of methods.

**When testing different HTTP methods, target low-priority objects. This helps make sure that you avoid unintended consequences, for example altering critical items or creating excessive records.**

###### Identifying supported content types
###### Using Intruder to find hidden endpoints

#### Finding hidden parameters

During API recon, you may find undocumented parameters that the API supports.

You can attempt to use these to change the application's behavior. Burp includes numerous tools that can help you identify hidden parameters:

- Burp Intruder enables you to automatically discover hidden parameters, using a wordlist of common parameter names to replace existing parameters or add new parameters. Make sure you also include names that are relevant to the application, based on your initial recon.

- The Param miner BApp enables you to automatically guess up to 65,536 param names per request. Param miner automatically guesses names that are relevant to the application, based on information taken from the scope.

- The Content discovery tool enables you to discover content that isn't linked from visible content that you can browse to, including parameters.

#### Mass assignment vulnerabilities

Mass assignment (also known as auto-binding) can inadvertently create hidden parameters. It occurs when software frameworks automatically bind request parameters to fields on an internal object. Mass assignment may therefore result in the application supporting parameters that were never intended to be processed by the developer.

##### Identifying hidden params

Since mass assignment creates parameters from object fields, you can often identify these hidden parameters by manually examining objects returned by the API.

For example, consider a `PATCH /api/users/` request, which enables users to update their username and email, and includes the following JSON:

```
{
    "username": "wiener",
    "email": "wiener@example.com",
}
```

A concurrent `GET /api/users/123` request returns the following JSON:

```
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "false"
}
```

This may indicate that the hidden `id` and `isAdmin` parameters are bound to the internal user object, alongside the updated username and email parameters.

##### Testing mass assignement vulnerabilies

#### Preventing vulnerabilities in APIs

When designing APIs, make sure that security is a consideration from the beginning. In particular, make sure that you:

- Secure your documentation if you don't intend your API to be publicly accessible.

- Ensure your documentation is kept up to date so that legitimate testers have full visibility of the API's attack surface.

- Apply an allowlist of permitted HTTP methods.

- Validate that the content type is expected for each request or response.

- Use generic error messages to avoid giving away information that may be useful for an attacker.

- Use protective measures on all versions of your API, not just the current production version.

To prevent mass assignment vulnerabilities, allowlist the properties that can be updated by the user, and blocklist sensitive properties that shouldn't be updated by the user.

#### Server-side parameter pollution

Some internal APIs aren't directly accessible from the internet.
Server side parameter pollution occurs when a website embeds user imput in a server-side request to an internal API without adequate encoding. This means that an attacker may be able to manipulate or inject parameters, which may enable them to, for exemple:

- Override existing parameters.
- Modify the application behavior.
- Access unauthorized data.

You can test all user input for any kind of parameter pollution.
For exemple, query parameters, form fields, headers, and URL path parameters may all be vulnerable.

> Sometimes called HTTP parameter pollution. However, this term is also used to refer to a web applicationfirewall (WAF) bypass technique. To avoid confusion, we'll refer to it only by server-side parameter pollution. Very different than server-side prototype pollution.

#### Testing for server-side parameter pollution in the query string

To test for server-side parameter pollution in the query string, place query syntax characters like #, &, and = in your input and observe how the application responds.

Consider a vulnerable application that enables you to search for other users based on their username. When you search for a user, your browser makes the following request:

```
GET /userSearch?name=peter&back=/home
```

To retrieve user information, the server queries an internal API with the following request:

```
GET /userSearch?name=peter&back=/home
```

##### Truncating query strings

You can use a URL-encoded # character to attempt to truncate the server-side request. To help you interpret the response, you could also add a string after the # character.

For example, you could modify the query string to the following:

```
GET /userSearch?name=peter%23foo&back=/home
```

The front-end will try to access the following URL:

```
GET /users/search?name=peter#foo&publicProfile=true
```


> It's essential that you URL-encode the # character. Otherwise the front-end application will interpret it as a fragment identifier and it won't be passed to the internal API.

Review the response for clues about whether the query has been truncated. For example, if the response returns the user peter, the server-side query may have been truncated. If an Invalid name error message is returned, the application may have treated foo as part of the username. This suggests that the server-side request may not have been truncated.

If you're able to truncate the server-side request, this removes the requirement for the publicProfile field to be set to true. You may be able to exploit this to return non-public user profiles.
