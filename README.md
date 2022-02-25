# NetBox

Alternative installation instructions for NetBox using Kubernetes. The main value is the netbox configurations but postgresql and redis instructions briefly provided via helm charts.

## Notes

* The netbox deployment will work with existing redis and postgresql installations, just ensure that the configurations are changed in `netbox-configmap.yaml` and `netbox-deployment.yaml`
* Relevant places that will need to be changed are indicated by the inline comment `#changeme`
* The manifests were tested against the following:
    * NetBox: 2.11.12
    * Postgresql: 11.6.0
    * Redis: 5.0.5
    * Kubernetes: 1.17.4
    * Rook: 1.4.2
    * Ceph: 15.2.4

## Installation

To get NetBox up and running on a Kubernetes cluster:

1. Deploy `netbox-namespace.yaml`
    * If the namespace is changed, ensure all manifests are updated accordingly
1. Deploy Postgresql. One way to do this is via a helm chart. **Make sure to change the password!**
    `helm install --name netbox-community --namespace netbox-community stable/postgresql -f postgresql-values.yaml`
    * storageClass should reflect the appropriate storage class for the environment
    * Further configuration can be seen here: https://github.com/helm/charts/tree/master/stable/postgresql
1. Deploy Redis. One way to do this is via a helm chart:
    `helm install --name netbox-redis --namespace netbox-community stable/redis -f redis-values.yaml`
    * storageClass should reflect the appropriate storage class for the environment
    * Further configuration can be seen here: https://github.com/helm/charts/tree/master/stable/redis
1. Change the values you need in the `netbox-configMap.yaml` and `netbox-secrets.yaml`
    * This would include any host, DB name, user changes that would be needed
    * The `AUTH_LDAP_BIND_PASSWORD` password would also need to change depending on the AUTH_LDAP_BIND_DN chosen
    * The `AUTH_LDAP_BIND_DN` is the Distinguished Name (DN) of the service account that will be making queries when a user requests login
    * Update the `AUTH_LDAP_SERVER_URI` to point to your LDAP server
    * If the DNS resolution for the LDAP server doesn't work use the IP address of your LDAP server
    * Any password changes will need to be reflected in `netbox-secrets.yaml`
    * Change appropriate values for DB, Email, and Redis to reflect your environment
1. Edit the media-files pvc in `netbox-media-pvc.yaml`. To allow for HA across your pods, ensure that the pvc allows for multiple pods to attach to it, such as through the use of a shared filesystem storage class
1. Deploy the main NetBox pod
    `kubectl apply -f netbox-deployment.yaml`
    * NetBox will initialize the needed tables into the DB
    * The readiness probe will check and ensure that the service is up and running
1. `kubectl apply -f netbox-service.yaml`
    * Usual deployment is via an Ingress but if needed, change the service to use NodePort in order to access it from outside the cluster

### Verification

1. A netbox instance is up and running with multiple pods
1. A PostgreSQL DB for netbox is up and running with all the necessary schema
1. Redis caching enabled and working

##  Enabling SSO (Optional)

There are plugins available in NetBox in order to setup NetBox to authenticate against an SSO authentication service. In this instance, we have used the django3_auth_saml2, and netbox-plugin-auth-saml2 plugins, and this was tested against JumpCloud.

1. Uncomment `REMOTE_AUTH_BACKEND: 'django3_saml2_nbplugin.backends.SAML2CustomAttrUserBackend'` in `netbox-configmap.yaml` and `PLUGINS = ['django3_saml2_nbplugin']` in `startup-configmap.yaml`, making sure to comment out the other instance of those variables
1. Fill in `sso-saml2-configmap.yaml` with the metadata acquired from your SSO authentication source
1. Deploy the

### Verification

1.

## Deploying an Ingress (Optional)

This isn't necessary for just testing. If you don't want to deploy the ingress resource, you should switch the service to use a NodePort so you can access it. nginx-ingress was used as the Ingress Controller in this example

1. Change the `netbox-ingress.yaml` file to your subdomain and TLS certificate for https
    * Consider using cert-manager to manage certificates: https://github.com/jetstack/cert-manager
    * The `kubernetes.io/tls-acme: "true"` line works with cert-manager ingress shim and requires a default certificate issuer to be set; see https://cert-manager.io/docs/usage/ingress/ for configuration details
1. `kubectl apply -f netbox-ingress.yaml`

### Verification

1. You can navigate to your netbox instance via a FQDN
1. https enabled with the certificate configured

## Deploying Metrics (Optional)

The manifests were generated against a prometheus/grafana deployment deployed via the kube-prometheus project. Consider checking them out at https://github.com/coreos/kube-prometheus

1. `kubectl apply -f netbox-monitoring-rbac.yaml`
    * Necessary if RBAC is enabled on the cluster
1. `kubectl apply -f netbox-monitoring.yaml`
    * This will create a ServiceMonitor object that you can see when navigating to Prometheus/Targets
1. Grafana dashboard # 9528 can be used to display the exposed Django metrics: https://grafana.com/grafana/dashboards/9528

### Verification

1. Prometheus target exists and is reporting ready for all pods deployed
1. Dashboard should report information regarding the metrics from Django backend

## Upgrading from NetBox v2.8.x

If the database was instantiated using a previous version of NetBox used in this repository, cable tracing will be broken when changing versions. It is necessary to run the the cable trace fix command in the NetBox container

1. `kubectl exec -it -n netbox-community netbox-<id> -- bash`
1. `./manage.py trace_paths`

## References

* https://github.com/digitalocean/netbox
* https://netbox.readthedocs.io/en/stable/
* https://github.com/netbox-community/netbox
