# Introduction #

These changes effect atom.service and gdata.service modules.

# Key Concepts #

In both the old and new versions, each HTTP request involved several components. There is a component which is responsible for opening a connection to the server, making the request, and receiving the response. There is also the idea of a authorization token which must be included in some requests as an HTTP `Authorization` header.

# High level overview #

In this new model, each `GDataService` instance contains:
  * an `http_client` object responsible for making HTTP requests.
  * a `token_store` which contains all of the auth tokens for the current user
    * the token store will find the appropriate token based on the requested URL
    * each token is an object which will perform the HTTP request using the `GDataService`'s `http_client`.

# Old vs. New #

## HTTP requests ##
In the old model, the HTTP requests were handled by a module level function called `HttpRequest`. The `gdata.service` module had a top level variable called `http_request_handler` which pointed to the desired module whose HttpRequest should be used. The default value was the `atom.service` module. This meant that a call to `gdata_service.Get('http://example.com')` made the HTTP request by using `atom.service.HttpRequest`.

If you wanted to change how the HTTP requests were made in this old model, you could do one of two things:
  1. Set `gdata.service.http_request_handler` to a different module before instantiating any `GDataService` objects.
  1. Set the `handler` member of a `GDataService` object to a different module.

In the new model, the HTTP requests are done using an instance of an `GenericHttpClient` class. (This class is defined in `atom.http_interface`.) Each instance of `GDataService` has an `http_client` member. This member has a `request` method which performs an HTTP request. When an instance of `GDataService` is created, the `http_client` defaults to a new instance of `atom.http.HttpClient`.

Under this new model, if you want to change how the HTTP requests were made you could do one of two things:
  1. Create an instance of a class which makes HTTP requests using a `request` method, and pass it in as the `http_client` argument when constructing a `GDataService` object.
  1. Create an instance of a class which makes HTTP requests using a `request` method and set the `http_client` member of an existing instance of `GDataService`.


## Token Management ##

In the old model, each instance of `GDataService` had a member called `auth_token` which stored the string which should be included as the HTTP `Authorization` header. This design had a couple of drawbacks. First, only one token could be stored in the `GDataService` and it was assumed that this token was valid for any requests that was being made by the client. Each token is actually valid for only a specific scope, a different token is required to access/modify data in different Google Data services. Second and more importantly, the token was stored as just a static string and was assumed to be constant. This is fine for ClientLogin tokens and non-secure mode AuthSub tokens, but secure AuthSub tokens and OAuth tokens require a digital signature to be included as part of the Authorization header. This digital signature must be calculated based on parameters in the HTTP request, so the Authorization header is unique to each request.

In the new model, the auth token is an object which knows which scopes it can be used with, and the `GDataService` can contain many token objects. The tokens are stored in a `token_store` member object which is able to add, find, and remove tokens.

When an HTTP request is made, the `GDataService` will ask it's `token_store` for the appropriate token based on the URL being requested. This token is responsible for constructing the Authorization header based on request parameters. Since the token would need to know the parameters of the HTTP request, I decided to allow the token to make the request using the `GDataService`'s `http_client`. The key method in the token is `perform_request` which looks like this (in pseudocode):
```
token.perform_request(http_client, operation, url, data, headers):
    # Create the Authorization header based on request parameters.
    auth_header = ...
    # Set the HTTP Authorization header.
    headers['Authorization'] = auth_header
    # Use the http_client object to make the HTTP request.
    return http_client.request(operation, url, data, headers)
```

One of the reasons that I created the `token_store` was to allow the `GDataService` to store and load a collection of tokens which could be reused over multiple sessions. I plan to write a subclass of the TokenStore which will run on Google App Engine and store all of the tokens for a specific user. These stored tokens would be loaded based on the current user and an appropriate token would be found in the store. Only when a suitable token could not be found would a user have to go through the AuthSub redirect process to obtain a new token.

## Pseudocode ##

Here is some pseudocode which provides a descriptions of how the logic has changed. (Sometimes code is easier to read in these situations.) In both examples, a `GDataService` object is performing an HTTP GET request.
### Old 'Get' method ###
```
# Set the Authorization header
# auth_token is a string
headers['Authorization'] = self.auth_token
# Make the HTTP request, the handler defaults to the atom.service module
self.handler.HttpRequest(self, 'GET', data=None, uri, headers)
```
### New 'Get' method ###
```
# Find a token to set the Authorization header as the request is being made
token = self.token_store.find_token(url)
# Tell the token to perform the request using the http_client object
# By default, the http_client is an instance of atom.http.HttpClient which uses httplib to make requests
token.perform_request(self.http_client, 'GET', url, data=None, headers)
```