# NetBox

Alternative installation instructions for NetBox using Kubernetes. The main value is the netbox configurations but postgresql and redis instructions briefly provided via helm charts.

## Notes

* The NetBox deployment will work with existing redis and postgresql installations, just ensure that the configurations are changed in `netbox-configmap.yaml` and `netbox-deployment.yaml`
* Relevant places that will need to be changed are indicated by the inline comment `#changeme`
* The manifests were tested against the following:
    * NetBox: 2.11.12
    * Postgresql: 14.2.0
    * Redis: 6.2.6
    * Kubernetes: 1.17.4
    * Rook: 1.4.2
    * Ceph: 15.2.4
* LDAP configurations were tested against the following:
    * Windows Server 2012 - Active Directory
    * JumpCloud - Cloud Directory
* SSO configurations were tested against the following:
    * JumpCloud - Cloud Directory

## Installation

To get NetBox up and running on a Kubernetes cluster:

1. Deploy `netbox-namespace.yaml`
    * If the namespace is changed, ensure all manifests are updated accordingly
1. Deploy Postgresql. One way to do this is via a helm chart. **Make sure to change the password!**
    `helm repo add bitnami https://charts.bitnami.com/bitnami`
    `helm install netbox-postgresql --namespace netbox-community bitnami/postgresql -f postgresql-values.yaml`
    * `storageClass` should reflect the appropriate storage class for the environment
    * `clusterDomain` should reflect the appropriate domain name for the kubernetes cluster if applicable
    * Further configuration can be seen here: https://github.com/bitnami/charts/tree/master/bitnami/postgresql
1. Deploy Redis. One way to do this is via a helm chart:
    `helm install netbox-redis --namespace netbox-community bitnami/redis -f redis-values.yaml`
    * `storageClass` should reflect the appropriate storage class for the environment
    * `clusterDomain` should reflect the appropriate domain name for the kubernetes cluster if applicable
    * Further configuration can be seen here: https://github.com/bitnami/charts/tree/master/bitnami/redis
1. Change the values you need in the `netbox-configMap.yaml` and `netbox-secrets.yaml`
    * This would include any host, DB, Redis, Email, abd user changes that would be needed to reflect your environment
    * Update the `AUTH_LDAP_SERVER_URI` to point to your LDAP server
    * The `AUTH_LDAP_BIND_DN` is the Distinguished Name (DN) of the service account that will be making queries when a user requests login
    * `AUTH_LDAP_USER_SEARCH`, `AUTH_LDAP_GROUP_SEARCH`, and `AUTH_LDAP_USER_FLAGS_BY_GROUP` will have to be filled in
    * If the DNS resolution for the LDAP server doesn't work use the IP address of your LDAP server
    * The `AUTH_LDAP_BIND_PASSWORD` password (`auth_ldap_bind_password` in `secrets.yaml`) would also need to change depending on the AUTH_LDAP_BIND_DN chosen
    * Any other password changes will need to be reflected in `netbox-secrets.yaml`
1. Deploy the startup and SSO configmap
    `kubectl apply -f netbox-startup-configmap.yaml`
    `kubectl apply -f sso-saml2-configmap.yaml`
    * `sso-saml2-configmap.yaml` must be deployed regardless of whether or not SSO is to be implemented (even if the file is empty), as the NetBox Deployment refers to it
    * If you would like to enable Single Sign-On with your LDAP service, refer to the Enabling SSO section below
    * If you would like to include other plugins, they will have to be added under `install-plugins.sh` to install the plugin packages, and under `PLUGINS` in `netbox-plugins.py` to enable it
1. Edit the media-files pvc's storageClassName in `netbox-media-pvc.yaml` and deploy it
    `kubectl apply -f netbox-media-pvc.yaml`
    * To allow for HA across your pods, ensure that the pvc allows for multiple pods to attach to it, such as through the use of a shared filesystem storage class
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

## Deploying an Ingress (Optional)

This isn't necessary for just testing. If you don't want to deploy the ingress resource, you should switch the service to use a NodePort so you can access it. nginx-ingress was used as the Ingress Controller in this example

1. Change the `netbox-ingress.yaml` file to your subdomain and TLS certificate for https
    * Consider using cert-manager to manage certificates: https://github.com/jetstack/cert-manager
    * The `kubernetes.io/tls-acme: "true"` line works with cert-manager ingress shim and requires a default certificate issuer to be set; see https://cert-manager.io/docs/usage/ingress/ for configuration details
1. `kubectl apply -f netbox-ingress.yaml`

### Verification

1. You can navigate to your netbox instance via a FQDN
1. https enabled with the certificate configured

##  Enabling SSO (Optional)

There are plugins available in NetBox in order to setup NetBox to authenticate against an SSO authentication service. In this instance, we have used the django3_auth_saml2, and netbox-plugin-auth-saml2 plugins, and this was tested against JumpCloud.

1. Modify `netbox-configmap.yaml` and `startup-configmap.yaml`
    * Uncomment `REMOTE_AUTH_BACKEND: 'django3_saml2_nbplugin.backends.SAML2CustomAttrUserBackend'` in `netbox-configmap.yaml`, making sure to comment out the other instance of this variable
    * Uncomment `PLUGINS = ['django3_saml2_nbplugin']` in `netbox-startup-configmap.yaml`, making sure to comment out the other instance of this variable
    * Fill in the `FLAGS_BY_GROUP` variable in `netbox-startup-configmap.yaml`, as the same configurations from `netbox-configmap.yaml` function inconsistently when SSO is enabled
1. Redeploy both configmaps
    `kubectl apply -f netbox-configmap.yaml`
    `kubectl apply -f netbox-startup-configmap.yaml`
1. Fill in `sso-saml2-configmap.yaml` with the metadata acquired from your SSO authentication source,
1. Deploy the SSO metadata configmap
    `kubectl apply -f sso-saml2-configmap.yaml`
1. Modify `sso-service-ingress.yaml` file with your subdomain and TLS certificate for https
    * Consider using cert-manager to manage certificates: https://github.com/jetstack/cert-manager
    * The `kubernetes.io/tls-acme: "true"` line works with cert-manager ingress shim and requires a default certificate issuer to be set; see https://cert-manager.io/docs/usage/ingress/ for configuration details
1. Deploy the SSO service and ingress
    `kubectl apply -f sso-service-ingress.yaml`
1. Restart the NetBox deployment to enable the service
    `kubectl rollout restart deployment -n netbox-community netbox`

### Verification

1. Login should automatically redirect to your SSO service if not already logged in there
    * You might have to logout of your SSO service before being able to log in to NetBox for the first time
1. Once logged in, confirm members of the groups described in `FLAGS_BY_GROUP` have the correct access
    * `is_active` defined group members only are allowed to navigate the regular NetBox interface. Specific available actions are determined by individual user and group permissions defined in NetBox permissions
    * `is_staff` defined group members are allowed to navigate the NetBox Administration menu. Specific available actions are determined by individual user and group permissions defined in NetBox permissions
    * `is_superuser` defined group members are given full read/write admin permissions across this NetBox deployment regardless of individual user and group permissions defined in NetBox permissions

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
