# Screenplay for POC

### Preliminaries
We assume that the following tools are already properly installed under the demo's base directory.
* [repo analysis tool](https://github.com/shift-left-netconfig/cluster-topology-analyzer)
* [synthesis tool](https://github.com/shift-left-netconfig/netpol-synthesizer)
* [network connectivity analyzer tool](https://github.com/shift-left-netconfig/network-config-analyzer)
* [baseline-rules verifier](https://github.com/shift-left-netconfig/baseline-rules-verifier)

### Deploy Google's microservices demo
1. Clone a copy of this repo: `git clone git@github.com:shift-left-netconfig/online-boutique.git`
1. `cd online-boutique`
1. Change context to `net-demo` namespace: `kubectl config set-context --current --namespace=net-demo`
1. `kubectl apply -f release/kubernetes-manifests.yaml`
1. Make sure all pods are running
1. Get the external IP of the frontend service by running `kubectl get svc frontend-external`
1. Copy-paste the external IP into a browser, to make sure the application is running

### Expose a security issue with the application
Frontend can actually connect to payment-service directly.
1. Run `kubectl exec frontend-54d58d8c84-dz9p2 -- wget paymentservice:50051` (replace pod-name with the actual name)

The output looks like
```
Connecting to paymentservice:50051 (172.21.109.255:50051)
wget: error getting response: Invalid argument
command terminated with exit code 1
```
So frontend was able to connect to the payment service, it just didn't use the correct arguments.

### Analyze the repo and synthesize relevant network policies
**Create a branch for the change (can also use a fork)**
```
git checkout -b add_netpols
```

**Run analysis**
```
../cluster-topology-analyzer/bin/net-top -dirpath . -commitid bfa5ba496e1ad30cba4545ab93ffa7082bd17eb9 -giturl https://github.com/shift-left-netconfig/online-boutique -gitbranch matser -outputfile microservices-demo.json
```
**Run synthesis**
```
../netpol-synthesizer/venv/Scripts/python ../netpol-synthesizer/src/netpol_synth.py microservices-demo.json -o release/microservices-netpols.yaml
```
**Push the change and create a PR**
```
git add release/microservices-netpols.yaml
git commit -m"adding network policies to enforce minimal connectivity"
git push --set-upstream origin add_netpols
```

Go to https://github.com/shift-left-netconfig and open a PR. Checks will show the new connectivity map, the diff from the previous state (diff is huge because we previously had all connections allowed, and now almost all conenctions are denied), and checks against baseline rules. The `require-label-to-access-payments-service` rule is expected to fail, because `checkoutservice` misses the `usesPayments: true` label.

Synthesis did not detect the baseline-requirements violation, because we didn't provide it with baseline rules.

### Add baseline requirements in the synthesis process

We would like to enforce that only deployments with a given label, say `payment_access: yes`,
will be allowed to access the payment service.
We will rerun the synthesis tool and provide it with a baseline-rules file:
```
../netpol-synthesizer/venv/Scripts/python ../netpol-synthesizer/src/netpol_synth.py microservices-demo.json -o release/microservices-netpols.yaml -b ../baseline-rules/examples/ciso_denied_ports.yaml -b ../baseline-rules/examples/restrict_access_to_payment.yaml
```

Since `checkoutservice` currently does not have the required label, the synthesis tool reports a conflict (currently as a warning), and the synthesized network policies should deny access. As a result, if the policies are deployed, the application breaks.

After adding the right label to the deployment, synthesis should go smooth, and the application should not break.

### Apply network policies and verify security issue is fixed

1. `kubectl apply -f release/microservices-netpols.yaml`
1. Run again `kubectl exec frontend-b4df774c5-gpxlt -- wget paymentservice:50051` (replace pod-name with the actual name)

The output should look like
```
Connecting to paymentservice (172.21.109.255:80)
wget: can't connect to remote host (172.21.109.255): Operation timed out
command terminated with exit code 1
```
This shows that the frontend can no longer connect to the payment-service.

### Show connectivity diff
The Network Connectivity Analyzer tool also allows us to compare connectivity configurations, producing a semantic diff showing which connections are added/removed.
```
 ../network-config-analyzer/venv/Scripts/python ../network-config-analyzer/network-config-analyzer/nca.py --semantic_diff release/microservices-netpols.yaml --base_np_list release/microservices-netpols2.yaml --pod_list release/kubernetes-manifests.yaml --ns_list release
```

### Check connectivity configuration against baseline requirements
Whenever the connectivity configuration is automatically synthesized using baseline requirements, it is correct by construction (should satisfy all baseline rules). However, if a configuration is manually changed or if it is synthesized with only a partial set of rules, it may not satisfy all rules. We can verify all rules are satisfied by a given configuration by running the **baseline-rules verifier**.
```
../baseline-rules-verifier/venv/Scripts/python ../baseline-rules-verifier/src/baseline_verify.py release/microservices-netpols2.yaml --repo release -b ../netpol-synthesizer/baseline-rules/examples/restrict_access_to_payment.yaml
../baseline-rules-verifier/venv/Scripts/python ../baseline-rules-verifier/src/baseline_verify.py release/microservices-netpols.yaml --repo release -b ../netpol-synthesizer/baseline-rules/examples/restrict_access_to_payment.yaml
```


### Cleaning
* Clean deployments and services: `kubectl delete -f release/kubernetes-manifests.yaml`
* Clean network policies: `kubectl delete -f release/microservices-netpols.yaml`
