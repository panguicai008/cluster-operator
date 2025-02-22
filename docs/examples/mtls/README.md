# Mutual TLS Peer Verification (Mutual TLS Authentication, mTLS) Example

Both RabbitMQ and clients can [verify each other's certificate chain](https://www.rabbitmq.com/ssl.html#peer-verification) for
trust. When such verification is performed on both ends, the practice is sometimes
referred to "mutual TLS authentication" or simply "mTLS". This example
focuses on enabling mutual peer verifications for client connections (as opposed to [node-to-node communication](../mtls-inter-node)).

For clients to perform peer verification of RabbitMQ nodes, they must be provided the necessary TLS certificates and private keys in Secret objects.
You must set `.spec.tls.secretName` to the name of a secret containing the RabbitMQ server's TLS certificate and key,
and set `spec.tls.caSecretName` to the name of a secret containing the certificate of the Certificate Authority which
has signed the certificates of your RabbitMQ clients.

First, you need to create the Secret which will contain the public certificate and private key to be used for TLS on the RabbitMQ nodes.
Assuming you already have these created and accessible as `server.pem` and `server-key.pem`, respectively, this Secret can be created by running:

```shell
kubectl create secret tls tls-secret --cert=server.pem --key=server-key.pem
```

In order for peer verification to work, the RabbitMQ nodes must trust a Certificate Authority which has signed
the public certificates of any clients which try to connect.

You must create a Secret containing the CA's public certificate so that the RabbitMQ nodes know to trust any certificates signed by the CA.
Assuming the CA's certificate is accessible as `ca.pem`, you can create this Secret by running:

```shell
kubectl create secret generic ca-secret --from-file=ca.crt=ca.pem
```

The Secret must be stored with key 'ca.crt'. These Secrets can also be created by [Cert Manager](https://cert-manager.io/).

Once the secrets exist, you can deploy this example as follows:

```shell
kubectl apply -f rabbitmq.yaml
```

## Peer verification

With mTLS enabled, any clients that attempt to connect to the RabbitMQ server that present a TLS certificate will have that
certificate verified against the CA certificate provided in `spec.tls.caSecretName`. This is because the RabbitMQ configuration option
`ssl_options.verify_peer` is enabled with mTLS by default.

If you require RabbitMQ to reject clients that do not present certificates by enabling `ssl.options.fail_if_no_peer_cert`,
this can be done by editing the RabbitmqCluster object's spec to include the field in the `additionalConfig`:

```yaml
spec:
  rabbitmq:
    additionalConfig: |
      ssl_options.fail_if_no_peer_cert = true
```

