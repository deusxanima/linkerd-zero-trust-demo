# Install Linkerd w/ Cert-Manager Self-Signed Certs

### Set up cert-manager rotation for webhook and linkerd control-plane certificates:

1. Apply the `linkerd-issuers.yaml`, `linkerd-certificates.yaml`, and `linkerd-ca-bundle.yaml` files that were attached to the e-mail. These will create the following resources for you automatically:

	a. self-signed root ClusterIssuer that will issue the control-plane root certificate for `root.linkerd.cluster.local` and the webhook anchor for `webhook.linkerd.cluster.local`
	b. ClusterIssuers that will issue all of the control-plane identity certs using the root cert and webhook cert created above.  
	c. All of the control-plane, webhook, linkerd-viz, and linkerd-jaeger certificates using the correct issuers created in the previous step.   
	d. A trust-bundle that is copied into the `linkerd` namespace so that the `linkerd-identity` component can use it to validate the certs when issuing the mTLS certs to the individual proxies.
	
	<b>Note:</b> I've defaulted to ClusterIssuers for ease of installation and management and we will write the certs directly to k8s since this is a demo setup, but in your actual prod setup we can discuss setting up namespaced Issuers and integrating with a secret manager (like Vault, for example) for added security and hardening in the actual prod env.

### Prep for HA Helm Installation:

1. Set up a new working directory
2. `cd` into that directory
3. Download the latest Linkerd Helm Chart with `helm fetch --untar linkerd/linkerd-control-plane`
4. Modify the `ha-values.yaml` by adding the following config options to the bottom of the file (please note the commented sections for additional instructions where relevant):

```
proxyInit:

  # If you are using Docker as your runtime in the cluster you must uncomment the option below to tell Linkerd initContainer that Docker is the containerruntime and allow Linkerd to run properly with the required access. Otherwise, you can remove this from the file.

  #runAsRoot: true

  # If you are running on RedHat nodes you must uncomment the option below to tell Linkerd initContainer that it is running on RedHat and should use the newer iptables binary when writing the access rules. Otherwise you can remove this from the file.

  #iptablesMode: “nft”


proxyInjector:

  # Set the following to use the webhook certs created by cert-manager in the previous section:

  externalSecret: true

  injectCaFrom: “<cert_namespace>/<cert_name>”

policyValidator:

  # Set the following to use the webhook certs created by cert-manager in the previous section:

  externalSecret: true

  injectCaFrom: “<cert_namespace>/<cert_name>”

profileValidator:

  # Set the following to use the webhook certs created by cert-manager in the previous section:

  externalSecret: true

  injectCaFrom: “<cert_namespace>/<cert_name>”

identity:
  
  # Since we’re using auto-rotating cert-manager certs, the following configuration modifications are required for Linkerd to be able to automatically read them from a k8s secret:

  externalCA: true
  issuer:
    scheme: kubernetes.io/tls
```

<b>Note:</b> for the purposes of the demo I'd recommend bootstrapping everything with Helm just to make it easier and faster, but we should discuss your planned gitops/install method for the actual prod environment(s) next week and can make a longer-term plan to better align with your existing CI/CD process going forward.

### Prep `kube-system` namespace for Linkerd installation:

To avoid catastrophic scenarios in the presence of complete proxy-injector failure, you must apply the `config.linkerd.io/admission-webhooks: disabled` label to the `kube-system` namespace before deploying the Linkerd Control Plane: `kubectl label namespace kube-system config.linkerd.io/admission-webhooks=disabled`

### Install Linkerd CRDs
`helm install linkerd-crds linkerd/linkerd-crds -n linkerd --create-namespace`

### Deploy Linkerd in HA Mode
`helm install linkerd-control-plane -n linkerd -f linkerd-control-plane/values-ha.yaml linkerd/linkerd-control-plane`

### Install Linkerd-Viz

`helm install linkerd-viz linkerd/linkerd-viz -n linkerd-viz --create-namespace`

### Install Linkerd-Jaeger

`helm install linkerd-jaeger linkerd/linkerd-jaeger -n linkerd-jaeger --create-namespace`

### Validate Linkerd Control Plane & Plugin Installation:
`linkerd check`

### Install demo workload using the BooksApp application:

For the purposes of the demo we recommend using the BooksApp application as it does a great job of demonstrating the flow of authz policies and is very easy to set up. You can read more about the BooksApp architecture and demo options here: https://linkerd.io/2.13/tasks/books/

Install the BooksApp by running the following: `kubectl create ns booksapp && curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/booksapp.yml | kubectl -n booksapp apply -f -`

### Mesh Workloads
Now that the Linkerd Control Plane is successfully deployed and running, you will want to add the following annotation to all namespaces you wish to mesh: `linkerd.io/inject: enabled` 

For the purposes of the demo, this will likely be just the booksapp namespace for now, but if you have others you want to test feel free to add them as well.

Once annotated, rollout restart the deployments in the namespace when downtime windows allow to have Linkerd automatically inject a Linkerd proxy into the workload and mesh it: `kubectl rollout restart deploy -n <newly_meshed_namespace>`

### Apply custom Prometheus scrape config if bringing your own Prometheus instance:

Start by reading the documentation here: https://linkerd.io/2.13/tasks/external-prometheus/

Once familiar with the documentation, apply the sample Prometheus configuration on that page to your external Prometheus instance. This will automatically scrape Linkerd metrics going forward and can then be used to send those to your Grafana dashboard collector.

### Applying Zero-Trust Policies:

By default Linkerd is installed with an `all-unauthenticated` default policy mode to ensure that the new Linkerd installation does not interrupt your existing networking traffic flow. You can read more about the defaults and available options here: https://linkerd.io/2.13/reference/authorization-policy/

Since you are interested in demonstrating what the environment would look like after locking it down completely and only allowing specific workloads to communicate with each other if meshed and authenticated, we will have to modify this existing behavior and apply a new default policy. To do this you will want to use the following steps:

1. Modify the default from `all-unauthenticated` to `deny` (this will lock down and prevent all cluster traffic communication until workloads are specifically enabled to communicate with each other, so please do not run this in a non-demo environments for now): `helm upgrade linkerd-control-plane -n linkerd --set proxy.defaultInboundPolicy linkerd/linkerd-control-plane`
2. Run `kubectl rollout restart deploy/<deployment_name> -n <namespace>` on all previously meshed demo workloads.
3. At this point, you should start seeing all of the requests start to fail cluster-wide. 
4. Create `Server` resources for the authors, books, and webapp workloads by applying the attached `linkerd-servers.yaml` resource files.
5. Create authorization policies for the authors, books, and webapp workloads by applying the attached `linkerd-authz.yaml` resource files.
6. At this point, you should start seeing traffic requests in the booksapp namespace start succeeding again. All other traffic will still be failing and workloads not specifically authorized by the created authorization policies will be unable to communicate with the authors, books, and webapp workloads as expected. Only the workloads specified in the individual `AuthorizationPolicy` specs will be allowed to communicate with each individual service.

### Confirm everything in Buoyant Cloud

Once you've confirmed your Buoyant Cloud access we will want to connect your demo cluster by using the following process:

1. Log into Buoyant Cloud at buoyant.cloud/signin
2. Navigate to Configure > Clusters in the left-side menu
3. Click "Add Cluster" in the top right
4. Name your cluster (this should ideally be the same name as the actual cluster name to help you maintain consistency and make it easier to identify when troubleshooting)
5. Copy/paste the commands that are provided in the guided install instructions to connect your demo cluster with Buoyant Cloud





