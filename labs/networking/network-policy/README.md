# Lab: Configure Network Policy

In this lab, you will restrict access to pods within a namespace.

## Prerequisites

* Complete previous labs for [AKS](../../create-aks-cluster/README.md) and [ACR](../../build-application/README.md).

#### Kubernetes and Kube-Router Overview
Kubernetes networking has following security model:
* For every pod by default ingress is allowed, so a pod can receive traffic from any one
* Default allow behaviour can be changed to default deny on per namespace basis. When a namespace is configured with isolation tye of DefaultDeny no traffic is allowed to the pods in that namespace
* when a namespace is configured with DefaultDeny isolation type, network policies can be configured in the namespace to whitelist the traffic to the pods in that namespace

Kubernetes network policies are application centric compared to infrastructure/network centric standard firewalls. There are no explicit CIDR or IP used for matching source or destination IP’s. Network policies build up on labels and selectors which are key concepts of Kubernetes that are used to organize (for e.g all DB tier pods of app) and select subsets of objects.

In this lab we will use Kube-Router for Network Policy Management. Kube-Router will use ipsets with iptables to ensure your firewall rules have as little performance impact on your cluster as possible.

#### Install Kube-Router
1. Run the following commands to deploy kube-router on your cluster
   ```
   cd kubernetes-hackfest/labs/networking/network-policy
   ```
   ```bash
   kubectl apply -f kube-router.yaml
   ```
2. Check to verify kube-router pods are running
   ```bash
   kubectl get daemonset -n kube-system -l k8s-app=kube-router
   ```

#### Deploy a new copy of the application to the `uat` namespace

Create the uat namespace:

```
kubectl create namespace uat
```

Install a copy of the app to the `uat` namepspace:

```
helm upgrade --install data-api-uat ./charts/data-api --namespace uat
helm upgrade --install quakes-api-uat ./charts/quakes-api --namespace uat
helm upgrade --install weather-api-uat ./charts/weather-api --namespace uat
helm upgrade --install flights-api-uat ./charts/flights-api --namespace uat
helm upgrade --install service-tracker-ui-uat ./charts/service-tracker-ui --namespace uat
```

Wait for the public IP to be assigned to the service-tracker-ui:

```
kubectl get svc -n uat -w
# CTRL+C to exit
```

Test the web app in your browser.

#### Update the namespaces to have labels (needed later)

kubectl label namespace uat namespace=uat
kubectl label namespace kube-system namespace=kube-system

#### Test that you can access a pod in the uat namespace from default namespace

```
kubectl run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
```

At the prompt, type:

```
wget -qO- --timeout=2 http://data-api.uat:3009
```

To exit, type `exit` and press ENTER.

#### Test that you can no longer access a pod in the uat namespace from default namespace

Now, create a network policy to deny traffic except from the `uat` and `kube-system` namespaces.

Paste the following into a file called `deny-from-other-namespaces.yaml`:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: uat
  name: deny-from-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          namespace: uat
    - namespaceSelector:
        matchLabels:
          namespace: kube-system
```

Then apply the changes:

```
kubectl apply -f ./deny-from-other-namespaces.yaml -n uat
```

Now test if you can access a pod in the `uat` namepace from the `default` namespace:

```
kubectl run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
```

At the prompt, type:

```
wget -qO- --timeout=2 http://data-api.uat:3009
wget: can't connect to remote host (10.0.122.115): Connection refused
```

To exit, type `exit` and press ENTER.

#### Cleanup (optional)

```
delete networkpolicy deny-from-other-namespaces -n uat
```

## Docs / References

* [Kube-Router](https://www.kube-router.io/)
* [DENY all traffic from other namespaces](https://github.com/ahmetb/kubernetes-network-policy-recipes/blob/master/04-deny-traffic-from-other-namespaces.md)
* [ALLOW traffic from apps using multiple selectors](https://github.com/ahmetb/kubernetes-network-policy-recipes/blob/master/10-allowing-traffic-with-multiple-selectors.md)


#### Next Lab: [Monitoring and Logging](labs/monitoring-logging/README.md)
