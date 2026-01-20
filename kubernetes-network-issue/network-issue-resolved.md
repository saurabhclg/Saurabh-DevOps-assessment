# Incident Report: AKS External Egress Communication Failure
### Incident Overview
A connectivity issue was reported where pods in the network-debug-demo namespace were unable to reach external internet services (e.g., Google, external APIs).

# Investigation & Diagnostics
### The following steps were taken to identify the root cause:

- Connectivity Test: Executed a curl command from within the pod.

- Result: Connection timed out (Confirming the issue).

- Policy Audit: Checked for existing NetworkPolicies.

- Result: Identified a policy named block-external-access with no defined Egress rules.

- Platform Verification: Verified cluster-level enforcement.

### Additional Findings with our AKS
Finding: The cluster was initially set to "networkPolicy": "none". After re-provisioning with Azure Network Policy, enforcement was successfully confirmed.

# Resolution
- The resolution involved updating the NetworkPolicy to explicitly allow traffic to DNS (Port 53) and HTTPS (Port 443) providers.

- The "Fix" Applied:
YAML

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: # Allow DNS for name resolution
    ports:
    - protocol: UDP
      port: 53
  - to: # Allow HTTPS for external API calls
    ports:
    - protocol: TCP
      port: 443
- Final Verification
Running curl -m 5 https://google.com now returns an HTTP 301 response, proving that the pod can successfully resolve DNS and establish an outbound secure connection.

## Network debugging session (example)

The following is a captured terminal session showing how I debugged egress/network policy behaviour in the `network-debug-demo` namespace. Keep this as an audit/log for troubleshooting.

```bash
snehasaurabh@Snehas-MacBook-Air ~ % kubectl get ns
NAME                 STATUS   AGE
default              Active   26h
helm-deployment      Active   15h
kube-node-lease      Active   26h
kube-public          Active   26h
kube-system          Active   26h
network-debug-demo   Active   22m

snehasaurabh@Snehas-MacBook-Air ~ % kubectl run access-test -n network-debug-demo --image=curlimages/curl -- sleep 3600
pod/access-test created

snehasaurabh@Snehas-MacBook-Air ~ % kubectl get pods -n network-debug-demo
NAME          READY   STATUS    RESTARTS   AGE
access-test   1/1     Running   0          9s

snehasaurabh@Snehas-MacBook-Air ~ % kubectl exec -it access-test -n network-debug-demo -- curl -m 5 https://google.com
curl: (28) Resolving timed out after 5000 milliseconds
command terminated with exit code 28

snehasaurabh@Snehas-MacBook-Air ~ % kubectl get netpol -n network-debug-demo
NAME                    POD-SELECTOR   AGE
block-external-access   <none>         22m

snehasaurabh@Snehas-MacBook-Air ~ % cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-external-access
  namespace: network-debug-demo
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    ports:
    - protocol: TCP
      port: 443
EOF
networkpolicy.networking.k8s.io/block-external-access configured

snehasaurabh@Snehas-MacBook-Air ~ % kubectl exec -it access-test -n network-debug-demo -- curl -I https://google.com
HTTP/2 301
location: https://www.google.com/
content-type: text/html; charset=UTF-8
content-security-policy-report-only: object-src 'none';base-uri 'self';script-src 'nonce-sdQM7IHmssulzW7T0hpCwg' 'strict-dynamic' 'report-sample' 'unsafe-eval' 'unsafe-inline' https: http:;report-uri https://csp.withgoogle.com/csp/gws/other-hp
date: Tue, 20 Jan 2026 15:38:31 GMT
expires: Thu, 19 Feb 2026 15:38:31 GMT
cache-control: public, max-age=2592000
server: gws
content-length: 220
x-xss-protection: 0
x-frame-options: SAMEORIGIN
alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
```

# Notes:
- The session shows initial DNS timeout, then applying a NetworkPolicy that permits DNS (TCP/UDP:53) and HTTPS (TCP:443) egress, after which HTTP/2 301 from google.com succeeds.
- Keep sensitive output out of the log; redact secrets if present.