---
layout:      post
title:       Swagger UI and DropWizard - API Docs and REST Client
description: 
headline:    "Swagger UI and DropWizard - API Docs and REST Client"
categories:  [Swagger,DropWizard]
tags:        [Swagger,DropWizard,REST,Java,API]
image:       
comments:    true
mathjax:     
featured:    false
published:   false
---


Just wanted to share something that I did in one Add-On with DropWizard. This Add-On has some Quartz scheduled jobs and the customer wanted to monitor/trigger them but we could not do this from the Jive UI because the instance runs in HTTPS (no mixed-content limitation) and the customer did not have a valid SSL certificate running on the Add-On server (we would have disable global certificate validation in Jive to make this work). One option is of course to have them install some kind of REST client and invoke the Add-On API but what are the chances the customer will get this right and make the correct requests?

## Swagger UI
> [Swagger UI](https://github.com/swagger-api/swagger-ui) is a dependency-free collection of HTML, Javascript, and CSS assets that dynamically generate beautiful documentation and sandbox from a Swagger-compliant API.

You add some Swagger annotations to the JAX-RS compliant REST interfaces, initialize the Swagger bean with the package to scan and some nice looking API documentation is generated OOTB. Not just docs, but a built-in REST client is also available for invoking the scanned REST endpoints.

## Swagger UI + DropWizard
This was the most complicated part of the Swagger integration process. With standard containers (e.g. tomcat, jetty, etc), the Swagger integration requires no additional code or configuration but to make this work with DropWizard, I had to create some additional logic and override code from the JDW AbstractApplication class. This logic was required to prevent DW from initializing these resources twice.
 
Swagger also seems to be more compatible with Spring and the integration with Guice is even a bigger challenge as we need to create a bundle and perform some ad-hoc Swagger configuration. As usual, there are some examples available in [GH](https://github.com/federecio/dropwizard-swagger) on how to achieve this.

## Add-On and Swagger UI
Below are some examples of how Swagger UI looks like while used to generate API documentation from a JDW enabled application. The first image is a screenshot of the landing page, which lists all top-level REST interfaces:

![Swagger Endpoints](/images/posts/2016-02-29-swagger-ui-and-dropwizard-api-documentation-and-rest-client/SwaggerEndpoints.png "Swagger Endpoints")