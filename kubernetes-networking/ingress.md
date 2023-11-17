# Ingress

{% embed url="https://kodekloud.com/blog/kubernetes-ingress/" %}

<figure><img src="../.gitbook/assets/ingress pod network - Page 1.png" alt=""><figcaption></figcaption></figure>

### Simple fact

Ingress provides a more flexible routing rule while the LoadBalancer service simply cannot.

The traffic reaches the ingress controller in the first place through the LoadBalancer service.

### Ingress controller

The Ingress controller is the actual engine that executes the rules of the ingress by translating them and routing the traffic accordingly.

There are a couple of ingress controller implementations out there, but the most popular are Nginx and HAProxy.

### Ingress resources

Ingress resource will update the controller with the rules to be applied to the traffic
