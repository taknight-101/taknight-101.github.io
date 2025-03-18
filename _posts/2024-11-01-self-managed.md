---
layout: post
title: "A Self-Managed Solution Architecture For 1st Party Applications"
permalink: "self-managed"
categories: [Platform Engineering, ServiceNow]
tags: [Automation, hookdeck, Ansible, Solution Architecture] # TAG names should always be lowercase
image: /images/self-managed/self-managed.jfif
---

## Intro

Described in this blog a **solution architecture** that allows for seamless integration of any agonistically implemented service to be employed and integrated to support any possible use case within the ServiceNow platform echo system

The purpose of this draft, is to put together a doable semantic into widening the horizon of incorporating diverse skillset into intriguing use cases that might not fit in ServiceNow’s environment but rather empower it with new capabilities to extend its business acumen

The presented draft is a core element that can in itself evolve by employing systems engineering ideas and end to end automation to account for more complex use cases

## Architecture

![]({{ site.baseurl }}/images/self-managed/architecture.png)

The idea is simple,

it implies 2 main personas, the snow admin & the developer.

the developer can spin up a webserver and expose a gateway endpoint to which the ServiceNow applications reach out with requested features and their processing payloads, and the server redirects to the target API that implements that requested feature.

the central component in the architecture is the webhook relay like [hookdeck](http://hookdeck.com/), a service whose CLI registers a server into a public webhook to register & maintain in a ServiceNow table to mange its state, features offerings, security controls etc,

by doing this, a reliable & whitelisted inbound connection from snow to the target feature's API is established as the hook relays the request directly to the webserver process to do its job and return back its result, which could be an actual response or a correlation id in case of async APIs,

as can be perceived from the draft design, many use cases and concerns are addressed like:

1. support for wide range of specific technical concerns that might pose a challenge in the normal setting

2. security concerns can be easily implemented by custom security controls & ACL rules in both the requesting side & processing side

3. allow for easier migration to public or private cloud and runtimes since the architecture is decoupled to the target infrastructure

4. open various ways to automation by employing tools & techniques from system design

5. extend self-managed capabilities to drive more business use cases in various domains

## A Basic POC

Follow the overall steps below to implement & wire the architecture components:

1. login to [hookdeck](http://hookdeck.com/) and download its CLI, it's a free service and its developer plan is sufficient for our purposes

2. spin up your webserver locally to listen for a target port say 8000

3. configure the CLI to relay the incoming requests to port 8000

4. access the webserver through its provided public webhook from SNOW's rest api or other means

Final consideration

The only manual setup that's is required for the initial phase is to setup and register the hookdeck service in the target server, “this can be also automated by [ansible](https://docs.ansible.com/ansible/latest/index.html) for convenience”

the admin can maintain all registered instances much like they would in case of mid-servers in snow's own alike solutions in case of any failures, and this process itself can benefit from ITSM flows like incident management

That's all for now.
