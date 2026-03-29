+++
date = '2026-03-14T09:24:24+01:00'
draft = false
title = 'How to Store SSL Certificates in a Kubernetes Secret'
+++

In one of the systems I maintain, we establish mTLS connections between applications so they can communicate with a RabbitMQ broker. This connection mode relies on presenting SSL certificates with certificate authority verification, but I won't go into the details of how this mechanism works here.

While adding a new application to the system, I needed to expose the certificate allowing it to authenticate. I stored it in our Vault instance in `PEM` format, which allows for easier reading and better Vault support than a binary `.p12` file.

We use the Vault Secret Operator to bridge secrets stored in Vault with Kubernetes secrets, through `VaultStaticSecret` CRDs. When creating the `VaultStaticSecret` for my new certificate, I initially thought I would simply reference the secret path in Vault, which had two fields — one for the certificate and one for the private key. I could then push everything into an `Opaque` type Kubernetes secret, as I had already done with other sensitive information stored in Vault.

Out of curiosity, I went to check the `VaultStaticSecret` CRD documentation. In the `destination` object, there is a `type` field that corresponds to the desired Kubernetes secret type. Only knowing the `Opaque` type, I wanted to see what other options were available to me.

I then discovered in the Kubernetes documentation the `tls` secret type, which matches my use case exactly: certificate storage.

Fundamentally, there isn't a huge difference with the solution using an `Opaque` type secret — both can work.

However, I see several advantages to using this type when possible.

1. The secret is composed of two keys: `tls.crt` for the certificate chain, and `tls.key` for the private key. It standardizes how certificates are stored and can serve as a fixed interface between a certificate producer and an application that consumes them.

2. At creation time, the Kubernetes API validates the presence of both fields. It does not, however, validate the content.

3. It is an already well-established convention in the Kubernetes ecosystem, found in `certmanager` for example.

4. This secret type can be referenced in `Ingress` resources and integrates natively with them. It works very well with Traefik without having to specify how to extract the certificate chain and private key.

5. It is possible to have even finer-grained permission segregation per resource when defining RBAC policies.

In a few other cases, I find that the `Opaque` type can still be relevant.

1. An application (often legacy) where you cannot change the name of the injected certificate and key.

2. An application that needs a format other than `PEM`. Here, you can still work around it with an `InitContainer` that handles the transformation to `.p12` or `.pfx` for example.
