# Overview: Embedded Browser as Content Handler in saml2test2

## Test Operations in the Test Tool
(to be updated from Rainer's view of saml2test to actual saml2test2)

The test operation is a sequence of SAML request-response pairs called (SAML) flows.
In this case there is just a single flow, which  is AuthnRequest/Response. However,
a flow can consist of multiple HTTP request-response pairs.  This is example from
the perspective of the test tool, emulating the user browser in the WebSSO flow:
- 1a. HTTP GET or POST on IDP’s SSO service URL, contents: AuthnRequest + RelayState
- 1b. HTTP response 30x to IDP login page (assuming the user has no active IDP session)
- 2a. HTTP GET IDP login
- 2b. HTTP response 200 (assuming UID/PW login)
- 3a. HTTP POST on IDP to login submit URL, content: form data with uid/pw
- 3b. HTTP response 200 (assuming IDP requires consent)
- 4a. HTTP POST on IDP to consent submit URL, content: form data with attribute release consent
- 4b. HTTP response 200, contents: Page wit Javascript that will POST SamlResponse + Relaystate)
- (Ends here - test tool analyzes form contents to check if test was passed.)

There is a function in the test tool (let us call it flow handler) that will
execute the operation outlined above. It is governed be following rules:
- A. HTTP Code 30x will be handled internally by HTTP client
- B. HTTP Code >= 400 will terminate the operation
- C. HTTP Code < 300 will be handled by the flow handler if it is the final response
(although this requires the flow handler to analyze the POST data - possibly
difficult if Javascript is involved. This would be better be done by a content handler)
- D. HTTP Code < 300 before the final response will be handled by a submodule using
robobrowser to match (and scape?) a particular page and construct the appropriate
contents for the subsequent HTTP request.

## Combining Different Content Handlers in the Test Tool
This section is taking the example above to extend the flow manager with the
embedded browser. Now the flow handler has a processing pipeline that allows
different content handlers to register, in our case robobrowser (RB) first and
embeddedBrowser (EB) second. The proposed interface to each content handler is:

    class ContentHandler:
    	 def __init__(self, interactions, conv=None):
    	 	:param interactions: This content handler does not have interactions,
    	 	so a NotImplemented is raised if interactions is not None.
    	 	:param conv: Is ignored

    	 def handle_response(self, urllib_request, urllib_response,
    	 		events, auto_close_urls, verify_ssl=True, cookie_jar=None):

			:param urllib_request: The (urllib2) HTTP request. The request is needed to
			resolve some data with relative paths.
			:param urllib_response: The (urllib2) HTTP response, which will be displayed
			in the Browser as a starting point for the test.
           	:param auto_close_urls: A list of AutoCloseUrl instances, which return
           	control to the caller on expected URLs, which are either passing or
           	failing the test.
           	:param events: An aatest.events.Events instance. This instance is updated by
           	the handler.
           	:param verify_ssl: (True/False) whether the test fails on ssl errors
           	:param cookie_jar: A http.cookiejar.CookieJar instance. Cookies for the embedded
           	browser are generated from the response/request pair. This instance is only
           	updated by the browser.
			:return: 'OK' if the test was passed, 'NOK' if it was not passed, 'aborted' if
			aborted by the user.



### Process:
Let us assume that there is a RB config matching the login form, but not the
consent page. Response 2b in the example above would be matched by the RB content
handler, returning the data to satisfy the login form. Response 3b would not be
matched by the RB content handler, so the flow handler would pass it on to the
EB content handler, which would create the window and load the HTML content. The
user would then leave the user test status as „OK“ and confirm the consent form
action. On pressing the submit button, where the URL will match an AutoCloseURL,
the EB content handler will close the window and return the ch_response.  The
flow handler will take the http_request parameters from the ch_response and
submit it to the test target.

# Embedded Browser Implementation
The browser will never send the request it is called with. It just needs the
request to know hostname and path. This might help to understand the limitations
of the design and actual implementaiton, and serve as a guide through the code:

1. The browser view is started and given a special fake URI to create a request.
2. Whenever the browser requests a page, this is fumbled through the Qt-lib and comes to light again in an interceptor module, that overloads the network handling of requests.
3. This module recognizes the special fake URI and, instead of reaching out to the network, recreates the response buffer from the given html and sets cookies
4. That response buffer is then bubbling up until it ends up in the browser view, which acts as a normal webkit browser all the time.
5. The same interceptor module checks, if any "known good/bad" URI comes along in a response and then terminates the browser automatically