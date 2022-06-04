+++
title = "SSO in Kubernetes"
date = 2020-08-03T10:24:28+02:00
description = "A simple, yet elegant way to add SSO to any service in Kubernetes"
tags = [
    "thumbnail",
]
thumbnail = "images/sso.png"
+++

## Introduction

When I've just started working on this, I thought it would be just peanuts to add SSO to static website,
but as it often goes with solving problems on Kubernetes - when you want to solve issue for one service in your infrastructure,
get ready to adopting a solution that will solve this problem for all your services.

Of course with something like [Ambassador](https://www.getambassador.io/) its really become quite easy to introduce,
but unless you're already using it - there is no point in dragging it solely for adding SSO for couple of your services
(as it's kinda like dragging Kubernetes to deploy three services, which nobody does, right?)

## Prerequisits

- Kubernetes cluster up and running
- Helm (preferably third generation of it, but its kinda doable even with second, with some caviats) that is configured for deployment on your cluster
- NGINX Ingress configured to expose your services
- Ability to register DNS names in your provider
- Certificate manager configured
- (Optional) ExternalDNS configured

## Overview

Here we will walk through creating service, vouch proxy for it, to introduce Single Sign On and configuring that SSO using Okta
(but of course you're free to use any other OIDC SSO provider)

## Creating simple web service

For that purpose we will create primitive analogue of the service you plan to expose, for experimenting on it.
It will be super simple `echo` service, that will response with "echo" on our `GET` request to its home.

Let's create `echo.yaml` and fill it with following:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "cert-manager"
spec:
  tls:
    - hosts:
        - echo.example.com
      secretName: echo-tls
  rules:
    - host: echo.example.com
      http:
        paths:
          - backend:
              serviceName: echo
              servicePort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  ports:
    - port: 80
      targetPort: 5678
  selector:
    app: echo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  selector:
    matchLabels:
      app: echo
  replicas: 1
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - name: echo
          image: hashicorp/http-echo
          args:
            - "-text=echo"
          ports:
            - containerPort: 5678
```

Now after running `$ kubectl apply -f echo.yaml` we should see something like:

![Echo service created](https://i.imgur.com/VfyYSkh.png&text=echo_created)

Now, if you have [ExternalDNS](https://github.com/kubernetes-sigs/external-dns) wait couple minutes for it to sync your services
with your DNS provider and after running `curl https://echo.example.com` you should be able to see the `echo` response.
Do you? Awesome! We're halfway there.

If you don't use ExternalDNS - don't get sad. Just go and configure your DNS manually to point to our newly created service.

## Configure Okta

If you're not admin of your org `Okta`, then I'd suggest creating private account for testing and playing around, better understandmant and in general
it won't hurt (except increasing enthropy, but its the price we're willing to pay anyhow).

In your dev console https://[your_domain].okta.com/dev/console click on Applications and the create a new one.

It will be of type Web.

On the next page you will be promted for its configuration. Name it to your like and tick on `Authorization Code` and `Refresh Token`
among "Allowed grant types". Except of this we're interested only in "Login redirect URIs". Put there `https://vouch.example.com/auth`
(replace example.com with domain you're owning). Yes, its not existing yet, but we'll be there soon.

## Installing Vouch

_Vouch Proxy_ is SSO solution for NGINX services, and since we're using NGINX Ingress for exposing our services - this is just the tool we need.
After its installation we would be able to add SSO to any our service in cluster with just as little as _4_ annotation. Isn't it the dream of any person
doing infrastructure?

Lets start with adding helm repo, which contains Vouch by running
`helm repo add vouch https://halkeye.github.io/helm-charts`

Next create file `vouch.yaml` and fill it with following:

```yaml
replicaCount: 1
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "cert-manager"
  hosts:
    - vouch.example.com
  paths: ["/"]
  tls:
    - secretName: vouch-tls
      hosts:
        - vouch.example.com
config:
  vouch:
    allowAllUsers: true
    cookie:
      name: vouchCookie
      domain: example.com
      secure: true
      httpOnly: true
      maxAge: 14400
    jwt:
      issuer: Vouch
      maxAge: 240
      compress: true
    session:
      name: vouchSession
  oauth:
    provider: oidc
    auth_url: https://[your okta domain].okta.com/oauth2/v1/authorize
    token_url: https://[your okta domain].okta.com/oauth2/v1/token
    user_info_url: https://[your okta domain].okta.com/oauth2/v1/userinfo
    scopes:
      - openid
      - email
      - profile
    callback_url: https://vouch.example.com/auth
    client_id: <your client id>
    client_secret:
```

And after execution of `$ helm upgrade --install vouch-proxy vouch/vouch -f vouch.yaml --set config.oauth.client_secret=<your client secret>` you will see:

![Vouch installed](https://i.imgur.com/T7uMHho.png&text=vouch_installed)

## Adding SSO to our echo service

Now the last step on the way to our super simple setup is just adding couple annotations to our service ingress and let the magic happen:

```yaml
nginx.ingress.kubernetes.io/auth-response-headers: X-Vouch-User
nginx.ingress.kubernetes.io/auth-signin: https://vouch.example.com/login?url=$scheme://$http_host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err
nginx.ingress.kubernetes.io/auth-snippet: |
  auth_request_set $auth_resp_jwt $upstream_http_x_vouch_jwt;
  auth_request_set $auth_resp_err $upstream_http_x_vouch_err;
  auth_request_set $auth_resp_failcount $upstream_http_x_vouch_failcount;
nginx.ingress.kubernetes.io/auth-url: https://vouch.example.com/validate
```

Run the `$ kubectl apply -f echo.yaml`, check that new configuration is applied and lets go test our SSO.
Navigate to https://echo.example.com and you should be promted to authenticate via Okta!

We're done, great job!
