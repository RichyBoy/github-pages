---
title: "Docker registry on the home network and SSL certifcate creation"
date: 2025-01-09
layout: post
tags: blog linux development docker registry SSL TSL certificates pem crt CA
---

## Motivation for a Docker registry

As a home developer you may only have one machine - in which case you may not even care about 'deploying' a container as long as it runs under MS Visual Code et al, or is built and deployed into the local Docker desktop instance. However I have a raspberry pi, a microserver and a laptop all running linux, as well as WSL 2 on my main Windows host.. and I also like to 'emulate' a more corporate/enterprise environment because ultimately that's where my skills are deployed professionally. Pulling the container from a registry is how it is supposed to work after all..

Running a registry for Docker is therefore very useful - and the pre-packaged image <https://hub.docker.com/_/registry> ticks the boxes. Except it really only does run locally and direct http connections are out of the question. Thus, SSL certificates are required to enable https. I have chose to run the docker registry on my microserver.

This post isn't a tutorial, I'm not going through the finer points of certification generation but it is a aide-memoire to a bunch of commands I'll only ever run once in a blue moon.

## Creation of certificates

Two certificates are required for a host, a root CA (root certificate authority) and then a certificate for the host issued by the root CA. Once generated, all of your docker apps that you create or pull such as registry or apache can access these certificates for SSL credentials which you generally provide as arguments in your docker compose file or straight on the command line with docker run. This isn't quite the end of it, the client accessing these services will have a valid certificate, but it won't be a trusted certificate because the client will have no trust around the root CA that you created. For a sane environment, you also need to add this root CA certificate to your Windows trusted root CA store, add it to apps like Firefox if you so will etc.

Create a directory to store the certificates and create a root CA for yourself. When filling in a DN I use a fully-qualified domain name, in my instance, microserver.lan. So, a new key pair is generated in the first line and it's used to sign the root CA generated on the second line.

```
mkdir cert && cd cert
openssl genrsa -out CA.key -des3 2048
openssl req -x509 -new -nodes -key CA.key -sha256 -days 1825 -out CA.pem
```

touch a file called local.ext and fill it with these contents to for the generation of a host certificate :

```
# SpecifY the attributes for the certificate

# Set authorityKeyIdentifier to identify the key and issuer of the certificate
authorityKeyIdentifier = keyid,issuer

# Define basicConstraints, indicating this certificate is not a Certificate Authority (CA)
basicConstraints = CA:FALSE

# Specify the allowed key usages for the certificate
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment

# Define the subject alternative names section
subjectAltName = @alt_names

# This section includes alternative names for the certificate

[alt_names]
# Specify DNS name(s)
DNS.1 = localhost
DNS.2 = microserver
DNS.3 = microserver.lan

# Define LOCAL IP address
IP.1 = 127.0.0.1
IP.2 = 192.168.1.14
IP.3 = 2a0e:cb01:9:XXXX::X
```

Replace hosts and IP addresses as appropriate, I've obscured my IPv6 address at that's real!

```
openssl genrsa -out local.key -des3 2048
openssl req -new -key local.key -out local.csr
openssl x509 -req -in local.csr -CA ./CA.pem -CAkey ./CA.key -CAcreateserial -days 3650 -sha256 -extfile local.ext -out local.crt
openssl rsa -in local.key -out local.decrypted.key
````

So, as before, generate a new key pair, this time for the host certificate, and then generate a signing request using this new key. The root CA generates a certificate with a nice healthy 10 years in this instance, and finally, extract the private key so that an applciation can sign content against the generated certificate. 

The final step is to add the root CA to the linux system key store, as I'm running SuSE it's slightly different to ca-certificates:
```
trust anchor CA.pem 
```

## So you have all the certs.. what next ?

Two steps: Services on the host that require TSL certificates - such as docker registry.

Create a file called registry.sh and chmod it with +x and add the following contents:

```
#!/bin/bash

docker run -d \
  --restart=always \
  --name registry \
  -v "$(pwd)"/cert:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/local.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/local.decrypted.key \
  -p 5001:443 \
  registry:2

```
Take note of the volume bind (so this is ran in folder one-up from the cert folder created) and take a note at the certificate and key that is provided.

```docker logs -f registry```
is a useful command to run as it will let you know if any of the keys are broken (for instance, I re-created the host certificate but forgot to re-generate the decrypted key so it told me about the mis-match)

### Root CA installation(s)

Applications from other hosts can now connect and as long as it is to a host specified in the DNS section of the local certificate (as per local.ext) it will connect securely - it's a genuine certicicate. But there is no trust because the root CA isn't known about.

For Windows 11, in the Settings application search for certificate and this will open the Certificate Manager, navigate to Trusted Root Certificate Authorities and install your root CA.pem file. Unfortunately this is lazily populated and I needed to reboot for this to take affect. The root CA.pem can also be added to Firefox explicitly and many other apps.

![Image](/assets/images/rootca.png "microserver.lan root CA added to the Windows trusted root")

Visual Code itself was a problem - I had to install [https://marketplace.visualstudio.com/items?itemName=ukoloff.win-ca](https://marketplace.visualstudio.com/items?itemName=ukoloff.win-ca) and its abstract explains exactly why..

### Testing Docker registry

Perhaps I should have had two separate blog posts :) Bearing in mind the Visual Code extension above, and a reboot..

In visual studio code, with the Docker plugin (I was going to use IDEA PyCharm but the remote services require a Professional instance; I'm not paying for one at this moment in time) you can naviage to the docker pane and you can add a Generic Repository - which is what registry is. The important part here is to use the correct credentials - in my instance https://microserver.lan:5001 and there are no login/passwords to be entered. My certificate also states microserver however I usually prefer FQDN to prevent my pihole from going off looking up network internals in the outside world. Never mind that..

Here is a screen shot of a push. On the top left panel with the red splodge, right click to tag it with something (foofoo) and the right click again to "push". You then get to select a repository and a tag name, just let it do its thing here. YOu can then see foofoo listed in the registry, and also I've highlighted the remote registry push log. If your certificate setup is wrong this is when you will find out about it in unhelpful ways such as attempted pushes to docker.io and in fact, the VS Code tool is good, but ONLY if everything works.

![Image](/assets/images/vscode.png)

Here is a curl command on Linux before and after this push to enquire against the registry:

![Image](/assets/images/working.png)

and finally the docker pull and the docker run command for my container:

![Image](/assets/images/excellent.png "Happy days Richard, Happy days")

## Final notes

This isn't exactly production as any extracted key you should put onto a key ring, but hey-ho this is 'home' development. Certificates are nasty and dirty to generate and work - but set up the once, and forget about it. Environments such as AWS will generate all the required key components for you (and store them appropriately).

The other approach I may have taken is to run a reverse proxy and point a AAAA record in richy.net to it. This way, I would be able to create a letsencypt cerficate from a proper trusted root CA. That's certainly how I'd go if I ran a web server on the home network for public use, but I have no reason to publically expose my private registry!
