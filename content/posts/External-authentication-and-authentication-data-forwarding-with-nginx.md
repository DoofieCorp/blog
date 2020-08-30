+++
title = "External authentication and authentication data forwarding with nginx"
description = "Using OAuth2 Proxy to protect applications hosted on Kubernetes exposed by Nginx Ingress"
keywords = ["kubernetes", "nginx", "oauth2 proxy", "ingress", "authentication"]
categories = ["Cloud", "Security", "Kubernetes"]
date = "2020-08-29T8:00:00Z"

+++

In today's world, it is normal to see applications running on [Kubernetes](https://kubernetes.io/) and exposed with [ingress nginx](https://kubernetes.github.io/ingress-nginx/).

Authentication can be added to any application exposed by ingress nginx by using [oauth2-proxy](https://kubernetes.github.io/ingress-nginx/). Using the following annotations configures nginx's [http auth request module](https://nginx.org/en/docs/http/ngx_http_auth_request_module.html). 

```
nginx.ingress.kubernetes.io/auth-signin: https://oauth2-proxy-ingress/oauth2/start?rd=https://$host$request_uri$is_args$args
nginx.ingress.kubernetes.io/auth-url: https://oauth2-proxy-ingress/oauth2/auth
```

When a user is not authenticated, nginx will redirect them to the OAuth2 proxy, prompt for login, generate a cookie and redirect back to the application. While this protects the application, it doesn't enable the application to know who the user is. This can be achieved by using additional configuration.

For OAuth2 proxy this means setting the [`--set-xauthrequest` flag](https://oauth2-proxy.github.io/oauth2-proxy/configuration), which will forward headers to nginx.

With this configuration added to OAuth2 proxy additional annotations must be added:

```
nginx.ingress.kubernetes.io/auth-response-headers: X-Auth-Request-Email
```

This will result in nginx sending a header of `http-x-auth-request-email` to the application.

Many applications can be configured to consume this header to identify the user. For example, the popular Django framework has [RemoteUserMiddleware](https://docs.djangoproject.com/en/3.1/howto/auth-remote-user/) for this exact purpose.