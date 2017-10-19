---
layout: post
title:  "IWSVA RCE Enhancement"
date:   2017-10-19 15:45:55
categories: Security
---

# Protection for malicious code injection on web console   

### Background

ZDI exposed some remote code execution vulnerabilities claim (VC) before, these VC have been trim by adding   
parameters inspection on every interface. But the left interfaces still suffers from similar vulnerabilities.  
On some conditions, attacker can utilize this vulnerability to perform remote code execution.

IWSVA web console has no a common parameters inspection module to defend the malicious code injection.  
Web console is based on tomcat server, tomcat provides filter mechiasm to inspection HTTP request before  
delivering to servlets.   

### ParamsFilter Framework

#### Filter chain
Currently, web console has a filter chain with two filers (for authentication and CSRF). To inspect  
all request parameters, ParamsFilter would be append on the filter chain.  

```
HttpRequest ------>  AuthFilter ------> CSRFGuardFilter ------> ParamsFilter
|                    |                   |                    |
|         request -->| authentication -->| token validate --->| parameters inspection
```

#### Parameter filter

- **White list for almost URI**  
The design for white list need collect all URL request data as possible, and analyze the parameters.

- **Black list for specific URI**  
some service interface should not be limit commands' customized parameters by design, i.e.(password, mount device)  

- **By pass for commonlog**  
https://IWSVA:8443/rest/commonlog/*  
Almost all request data to commonlog service are proxy to python WSGI, the WSGI is mainly to update,  
query database with specific SQL commands files.  
Other commonlog service calling uihelper already has security enhancement in ZDI VC fix.

### Steps for white & black list
- Collect request parameters from web UI: request URL, charactor scope.  
  Tool: [**paramscollect**](.//params_collect.tgz)  
  Manual: [**Readme**](.//RE Tomcat filter RE Summary about ManageSRouteSettings Exploit.msg)
- Design regex patterns (white & black) based on the request URL
- Tests and adjust patterns.
