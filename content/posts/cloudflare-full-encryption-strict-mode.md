---
title: "Cloudflare with full encryption in strict mode"
date: 2022-07-08T20:00:00Z
draft: false
---

When using cloudflare to manage your domains, in most cases you will want to use cloudflare as a proxy. That means all traffic goes through the cloudflare network before reaching your server. That method has some benefits like the SSL-Certificates are created and managed by cloudflare and you can apply the manifold functions (e.g. CDN, Zero Trust, ...) from cloudflare to your network.

While it is fairly easy to connect cloudflare to your server over http (encryption mode off or flexible) it is not secure. That way the browser will connect to cloudflare with https but cloudflare to your server with http. To achieve complete encryption for the whole connection you have to set encryption mode to full. That way cloudflare also connects to your server with https.

The encryption mode full has a more strict option called "strict mode". That requires your server to serve an SSL-Certificate that cloudflare can trust and cloudflare can be sure to communicate with your server.

![Cloudflare set encryption mode](/static/images/cloudflare-full-encryption-strict-mode-ssl-settings.png)

# Setup full encryption mode (strict)

To set the encryption mode, enter the specific domain from the cloudflare dashboard and navigate to "SSL/TLS" -> "Overview". If you use a server provider that already provides a trusted certificate you may be done here. A request over the cloudflare network should work as before. But if you by now don't use https at all or have a self signed certificate, a request over cloudflare will now fail.

To change the failing behavior and to make cloudflare trust our server we have to use a cloudflare certificate. To create one you go to "SSL/TLS" -> "Origin Server" and there click "Create Certificate".

![Cloudflare origin server](/static/images/cloduflare-full-encryption-origin-server.png)

On the next screen all default settings should be sufficient. You can just click "Create".

![Cloudflare create origin certificarte](/static/images/cloudflare-full-encryption-create-certificate.png)

You will receive an origin certificate and the corresponding private key. Save them in cert.pem and key.pem. The private key will not be accessible later.

![Cloudflare create origin certificarte](/static/images/cloudflare-full-encryption-certificates-created.png)

The usage of these files to configure https on certain servers is a bit different, for that reason i'm not going into detail here.

[Setup with NGINX](http://nginx.org/en/docs/http/configuring_https_servers.html)

[Setup with HAPROXY](https://www.haproxy.com/de/blog/haproxy-ssl-termination/)

[Setup with Emissary Ingress](https://www.getambassador.io/docs/emissary/latest/topics/running/tls/)

# Improved security with Authenticated Origin Pulls

Until now cloudflare knows that it is talking to our server, but we don't know who is talking to us. To assure that, there is a mechanism called [Client Certificate Authentication](https://medium.com/@sevcsik/authentication-using-https-client-certificates-3c9d270e8326). With client certificate authentication at every SSL-Handshake the server requires a certificate sent by the requester. That certificate must match the server certificate and authenticates the client against the server.

To achieve that we can use the already created certificate - private key pair. But the server must be configured additionally to TLS termination described before with the client certificate (often called ca_cert):

[Setup client certificate with NGINX](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_client_certificate)

[Setup client certificate with HAPROXY](https://www.haproxy.com/de/blog/ssl-client-certificate-management-at-application-level/)

[Setup client certificate with Emissary Ingress](https://www.getambassador.io/docs/emissary/latest/howtos/client-cert-validation/)

Often there is a pitfall! Most servers require the root certificate of the Certificate Authority (CA) to function. For cloudflare you can find that one [here](https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem). You can copy the text content into the cert.pem file at the end.

To make cloudflare initiate the SSL-Handshake with the client certificate we have to activate a feature called [authenticated origin pulls](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/set-up/).

![Activate authenticated origin pulls](/static/images/cloudflare-full-encyption-authenticated-origin-pulls.png)

# Conclusion

Every request from the browser to cloudflare and from cloudflare to your server should now be encrypted. Additionally cloudflare can trust your server and you can be sure that every request comes from cloudflare. 