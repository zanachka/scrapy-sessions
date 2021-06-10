# scrapy-sessions
A session-management extension for Scrapy.

[![PyPI Version](https://img.shields.io/pypi/v/scrapy-sessions.svg?color=blue)](https://pypi.org/project/scrapy-sessions)

## Overview
This library resolves at least three long-standing issues in Scrapy's session-management system that people have raised concerns about for years:
1. Scrapy's sessions are effectively a black box. They are difficult to expose and alter within a scrape.
2. Scrapy makes it very difficult to replace/refresh a session (and/or general 'profile') unilaterally across all requests that are scheduled or enqueued. This is important for engaging with websites that have a session-expiry system based on profile (IP/user-agent) or use short-lived sessions that require a custom renewal logic. Scrapy's cookie system fails to handle such websites.
3. Scrapy provides no native capability for maintaining distinct profiles (client identities) within a single scrape.

This library contains a `CookiesMiddleware` that exposes the Scrapy cookie jars in the spider attribute `sessions`. This is an instance of the new `Sessions` class (`objects.Sessions`) that allows one to examine the content of the current sessions and to clear and/or renew a session that is failing. The renewal procedure short-circuits the Scrapy request scheduling process, inducing an immediate download of the request specified, ahead of all others. This does not cause any adverse consequences (for example, scrape statistics are maintained perfectly).

This library also provides a tool for maintaining and rotating "profiles", making it easy to give the appearance that your scrape's requests are being generated by multiple, entirely distinct clients.

Another use case is for handling session cookies collected outside of Scrapy and fed into your spider. Whenever this external collection is necessary (for websites that require some kind of demonstration of Javascript rendering before they serve a session to an unknown client), this library provides a handy solution for cycling from one session to the next at each point of failure.

--- 
## Relation to the Default Scrapy CookiesMiddleware
The **scrapy-sessions** `CookiesMiddleware` is designed to override the default Scrapy `CookiesMiddleware`. It is an *extension* of the default middleware, so there shouldn't be adverse consequences from adopting it. 

The `"COOKIES_ENABLED"` and `"COOKIES_DEBUG"` settings work exactly as with the default middleware: if `"COOKIES_ENABLED"` is disabled, this middleware is disabled, and if `"COOKIES_DEBUG"` is enabled, you will get the same debug messages about cookies sent and received.

With this said, there are some important differences to note. With the default Scrapy middleware, the value of the `"cookiejar"` key in your `request.meta` names the session (cookie jar) that the request will use. If the session does not exist, a new session is created. The exact same applies in this library, except that you can now also use the `"session_id"` key for this purpose. The default value for this is now `0`, rather than `None`. So, if you don't use either of these keywords in any of your requests, each request will by default send the cookies associated with session `0`, and add any cookies it receives to session `0`. 

--- 
## Set up
### Basic
Override the default middleware:

````
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.cookies.CookiesMiddleware': None,
    'scrapy_sessions.CookiesMiddleware': 700,
}
````

This will allow you to interact with the `spider.sessions` attribute, in order to inspect, clear and renew sessions (see [*usage*](#usage)). It will also give you access to the response cookies via `response.meta["cookies"]`. 

### [Profiles](#profiles)
This is a separate add-on that hooks onto the sessions.

After changing `settings.py` as above, add the following:
`SESSIONS_PROFILES_SYNC: True`.

Then create a `profiles.py` file at the head of your project similar to the following:
````
from w3lib.http import basic_auth_header
PROFILES = [
    {"proxy":['proxy_url', basic_auth_header('username', 'password')], "user-agent": "MY USER AGENT"},
    {"proxy":['proxy_url', basic_auth_header('username', 'password')], "user-agent": "MY USER AGENT"}
]
````
(Either the "proxy" key or the "user-agent" key can be omitted for each profile (but not both).)

Finally, after importing the `load_profiles` function (`from scrapy_sessions.utils import load_profiles`), add the following to your spider settings:
````
custom_settings = {
  "SESSIONS_PROFILES":load_profiles('profiles.py')
}
````
Currently, this `load_profiles` function fails when trying to deploy on Zyte. I will try to solve this issue when I have time.

--- 
## [Usage](#usage)
### Accessing the cookies received in the last response
`response.meta["cookies"]`

### Accessing the current session id
`response.meta["session_id"]`
<br/><br/><br/>
In the below `self` is referring to a `Scrapy.spider` class.

### Viewing a session
The default session (session 0):
`self.sessions.get()`

A specified session:
`self.sessions.get(response.meta["session_id"])`

In dictionary format:
`self.sessions.get(session_id, mode=dict)`

### Clearing a session
The default session:
`self.sessions.clear()`

Specifying a session works the same as in `get`.

### Clearing and immediately renewing a session (instantly downloaded out of sync)
The default session:
`self.sessions.clear(renewal_request=Request(url='renewal_url',callback=self.cb))`

The callback is optional; if no callback is specified, the session is renewed just the same.

### Viewing a profile
The profile for the default session:
`self.sessions.get_profile()`

Specifying a session works the same as before.

This method will only work if `SESSIONS_PROFILES_SYNC` is enabled in the spider settings.

---
## Session Refresh Motivation
There are two use cases for this:
1. For handling websites that track session usage by some aspect of client identity, such as IP. This is not a common web-security feature but it does exist, and Scrapy can't handle it. By default, Scrapy will send all your requests with the one session, so if you send all your requests with the same identity signatures also, then you will be able to navigate such sites until your session expires due to reaching a time or usage limit. When this session expires, though, you need to refresh it and initiate a new one with a new identity. This library provides two ways of doing this, with and without using the `Profiles` add-on.
2. For handling the rotation of sessions collected by some process external to Scrapy. You might make use of such an approach whenever you are unable to collect a valid session without being able to render Javascript on a site, as in the case of sites that validate clients based on fingerprinting techniques. However you collect these sessions, it is vital to be able to seamlessly switch from session A to session B as soon as session A starts failing; the clear-and-renew amenity provided by this package is the appropriate solution.

### Session Refresh Implementation
#### With Profiles
Set up your profiles, then within some part of an errback function or middleware that only gets activated when a session expires (you may need some custom logic here), clear and renew your session using `sessions.clear`. Because you are using `profiles`, then any `renewal_request` you specify within the `clear` method will automatically get visited by a fresh profile.
#### Without Profiles
Within some part of an errback function or middleware that only gets activated when a session expires, clear and renew your session using `sessions.clear` by specifying a `renewal_request` that uses a fresh proxy and/or user-agent.

---
## Session Refresh Logic
Since this is the most complicated part of the library it's worth describing the underlying process. The following is what happens when `clear` is called with a `renewal_request` argument:
1. The specified session is cleared and the request specified is immediately downloaded, without entering the standard request queue. The way I have achieved this, the logs and statistics are updated as normal and everything seems to go smoothly.
2. The first response to make it to the `process_response` function in the middlewares will then re-fill the session. This should be the response derived from the `renewal_request`. (I think this is 100% guaranteed but I am not 100% sure.)
3. The comparison of the variable `_times_jar_renewed` in the request.meta (fed in during `process_request`) with the attribute `times_jar_renewed` on the `DynamicJar` object is used to determine in the `process_response` function whether a response has been downloaded using the old session. If this is the case for a given response, the request that led to that response is sent off to be retried using the new session.   

--- 
## Profiles
The idea of this tool is to manage distinct client identities within a scrape. The identity consists of two or more of the following attributes: session + user agent + proxy.

The profiles are input via a special `profiles.py` file (see [*setting up profiles*](#profiles)). Once you have these set up (and have tweaked the settings as required), one of these profiles is automatically associated with every new session created in your scrape. If there are more sessions than profiles, the profiles will be automatically recycled from the beginning. When a session is cleared, the profile is also removed.

### How it works
Index 0 of any "proxy" value is fed into the `request.meta["proxy"]` field in the `process_request` function of the middleware. Index 1 is fed into `request.headers['Proxy-Authorization']`.

Similarly, the "user-agent" value is fed into `request.headers["user-agent"]`.

--- 
## Future Directions
I am planning to add tests, and then I may at some point submit a pull request on the Scrapy repository proposing this as a replacement for the default Scrapy `CookiesMiddleware`.
