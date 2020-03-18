# Deploying CF for K8s

## Prerequisites

You need the following CLIs on your system to be able to run the script:

* [`kapp`](https://k14s.io/#install)
* [`ytt`](https://k14s.io/#install) (v0.26.0+)

In addition, you will also probably want [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for your own debugging and inspection of the system.

Make sure that your Kubernetes config (e.g, `~/.kube/config`) is pointing to the cluster you intend to deploy CF for K8s to.

## IaaS Requirements

To deploy cf-for-k8s as is, the cluster should:
* be running version 1.14.x or 1.15.x
* have a minimum of 5 nodes
* have a minimum of 2 CPU, 7.5GB memory per node
* support LoadBalancer services
  * requires a workaround on Minikube and Kind, for example
* define a default StorageClass
  * requires [additional config on vSphere](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/storageclass.html), for example

## Steps to deploy

1. Clone and initialize this git repository:
   ```bash
   $ git clone https://github.com/cloudfoundry/cf-for-k8s.git
   $ cd cf-for-k8s
   ```

1. Create a "CF Installation Values" file and configure it:

   You have the option of auto-generating the installation values or creating the values by yourself.

   #### Option 1 - Run generate-values.sh to generate the install values
   **NOTE:** The script requires the [BOSH CLI](https://bosh.io/docs/cli-v2-install/#install) to generate the install values from `bosh intepolate`
   ```bash
   # expects bosh cli
   $ ./hack/generate-values.sh cf.example.com > /tmp/cf-values.yml
   ```
   #### Option 2 - Create the install values by hand
   1. Create a file called `/tmp/cf-values.yml`. You can use `sample-cf-install-values.yml` in this directory as a starting point.
   1. Open the file and change the `system_domain` and `app_domain` to your desired domain address
   1. Generate certificates for the above domains and paste them in `crt`, `key`, `ca` values

   Make sure that your certificates include a subject alternative name entry for the internal `*.cf-system.svc.cluster.local` domain in addition to your chosen external domain.

1. Run the install script with your "CF Install Values" file
   ```bash
   $ ./bin/install-cf.sh /tmp/cf-values.yml
   ```

   If you wish to include one or more overlays in your installation, you can use `ytt` and `kapp` directly, instead of the `install-cf.sh` script:
   ```bash
   $ ytt -f ./config -f ./config-optional/remove-resource-requirements.yml -f /tmp/cf-values.yml > /tmp/cf-template.yml
   $ kapp deploy -a cf -f /tmp/cf-template.yml
   ```
   Note: you can include as many overlay files as desired in the `ytt` command, but each one requires its own `-f` argument

1. Configure DNS on your IaaS provider to point the wildcard subdomain of your
   system domain and the wildcard subdomain of all apps domains to point to external IP
   of the Istio Ingress Gateway service. You can retrieve the external IP of this service by running
   `kubectl get svc -n istio-system istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[*].ip}'`
   1. If you used the `./hack/generate-values.sh` script then you should only
      configure a single DNS record for the domain you passed as input to the
      script and have it resolve to the Ingress Gateway's external IP

      e.g.
      ```
      # sample A record in Google cloud DNS. The IP address below is the address of Ingress gateway's external IP
      Domain                  Record Type       TTL         IP Address
      *.<system_domain>	      A	            30	      35.111.111.111
      ```

## Validate the deployment

1. Set up cf cli to point to CF:
   ```bash
   $ cf api --skip-ssl-validation https://api.<system_domain>
   $ cf auth admin <cf_admin_password>
   ```

1. Enable docker feature:
   ```bash
   $ cf enable-feature-flag diego_docker
   ```

1. Deploy an app:
   ```bash
   $ cf push diego-docker-app -o cloudfoundry/diego-docker-app
   ```
   Note that the above command will return an error but the app is successfully pushed to CF and is routable via Http. The reason the command fails is due to a missing logging component, which we, the Release Integration, are working with the Logging team to integrate into CF4K8s

1. Validate the app is reachable
   ```bash
   $ curl http://diego-docker-app.<system-domain>/env
   # should return JSON value
   ```

## Delete CF4K8s install
You can delete CF4K8s deployment by running the following command.

```
# Assuming that you ran `bin/install.sh...`
$ kapp delete -a cf
```
