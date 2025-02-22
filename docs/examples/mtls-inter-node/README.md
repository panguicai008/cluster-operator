# Mutual TLS Peer Verification (Mutual TLS Authentication, mTLS) for Inter-node Traffic Example

When a clustered RabbitMQ node connects to its cluster peer, both
can [verify each other's certificate chain](https://www.rabbitmq.com/ssl.html#peer-verification) for trust.

When such verification is performed on both ends, the practice is sometimes
referred to "mutual TLS authentication" or simply "mTLS". This example
focuses on enabling mutual peer verifications for inter-node connections (as opposed to [client communication](../mtls)).

This example first makes RabbitMQ cluster nodes [communicate via TLS-enabled cluster links](https://www.rabbitmq.com/clustering-ssl.html)
for additional security.
In the future, the RabbitMQ Cluster Operator may make this easier to configure but it is already possible with the [`envConfig`](https://www.rabbitmq.com/kubernetes/operator/using-operator.html#env-config) and [`override`](https://www.rabbitmq.com/kubernetes/operator/using-operator.html#override) properties.

The most important parts of this example are:

- `rabbitmq.yaml` - `RabbitmqCluster` definition with all the necessary configuration
- `inter_node_tls.config` - inter-node communication configuration (Erlang distribution) file that will be mounted as a volume

The other files serve as an example for setting up certificates with [Cert Manager](https://cert-manager.io/docs/).

- `rabbitmq-ca.yaml` - defines an `Issuer` (CA)
- `rabbitmq-certificate.yaml` - defines a certificate that will be provisioned by Cert Manager and then mounted as a volume

**NOTE** `rabbitmq-certificate.yaml` contains the word "examples" multiple times - in the `namespace` and `dnsNames` properties.
You need to replace all occurrences with your desired namespace. `dnsNames` values need to contain the actual namespace name this cluster will be deployed to, otherwise TLS will fail due to hostname mismatch.

`setup.sh` should perform all the necessary steps but may need to be adjusted to work on your system.

```shell
# install Cert Manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.yaml
# deploy the example
./setup.sh
```

To validate that RabbitMQ nodes connect over TLS you can run the following checks:

```shell
# check that the distribution port has TLS enabled (this command should return `Verification: OK`)
kubectl exec -it mtls-inter-node-server-0 -- bash -c 'openssl s_client -connect ${HOSTNAME}${K8S_HOSTNAME_SUFFIX}:25672 -state -cert /etc/rabbitmq/certs/tls.crt  -key /etc/rabbitmq/certs/tls.key -CAfile /etc/rabbitmq/certs/ca.crt 2>&1 | grep Verification'

# check that distribution uses TLS (this command should return `{ok,[["inet_tls"]]}`)
kubectl exec -it mtls-inter-node-server-0 -- rabbitmqctl eval 'init:get_argument(proto_dist).'
```
