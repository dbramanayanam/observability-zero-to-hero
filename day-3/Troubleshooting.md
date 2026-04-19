# EKS Monitoring Ingress Troubleshooting Cheat Sheet

This cheat sheet captures the end-to-end steps used to troubleshoot and expose the `kube-prometheus-stack` on EKS using the AWS Load Balancer Controller.

---

## Goal

Expose these monitoring components through an ALB-backed Kubernetes Ingress:

- Prometheus
- Grafana
- Alertmanager

Namespace used:

```bash
monitoring
```

Cluster used:

```bash
observability
```

Region used:

```bash
us-east-1
```

---

## 1. Apply the Ingress

Apply the ingress manifest for the monitoring stack:

```bash
kubectl apply -f ingress_kube_prom_stack.yaml -n monitoring
```

Validate resources:

```bash
kubectl get all -n monitoring
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
```

---

## 2. Install AWS Load Balancer Controller

### Download IAM policy

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json
```

### Create IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Create IAM service account for controller

```bash
eksctl create iamserviceaccount \
  --cluster=observability \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<accountID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### Add Helm repo and install controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Initial install that caused trouble because region was wrong:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=observability \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1a \
  --set vpcId=vpc-08256894ca4399788
```

> **Issue:** `us-east-1a` is an Availability Zone, not a valid AWS region for STS/API calls.

### Correct controller configuration

```bash
helm upgrade -n kube-system aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=observability \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC-ID>
```

---

## 3. Validate Controller Health

Check pods:

```bash
kubectl get pods -n kube-system | grep aws-load-balancer-controller
```

Check logs:

```bash
kubectl logs -n kube-system deploy/aws-load-balancer-controller
```

Check rendered deployment config:

```bash
kubectl get deployment aws-load-balancer-controller -n kube-system -o yaml
kubectl get deployment aws-load-balancer-controller -n kube-system -o yaml | grep -i region -A2 -B2
```

---

## 4. Troubleshoot the Ingress

Check ingress:

```bash
kubectl get ingress -n monitoring
kubectl describe ingress kubernetes-prometheus-stack -n monitoring
```

### Common issue found

Ingress backend service names were placeholders and did not exist:

- `prometheus-service`
- `grafana-service`
- `alertmanager-service`

### Verify actual monitoring services

```bash
kubectl get svc -n monitoring
```

Actual services found:

- `monitoring-kube-prometheus-prometheus`
- `monitoring-grafana`
- `monitoring-kube-prometheus-alertmanager`

---

## 5. Correct Ingress Manifest

Use this corrected ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubernetes-prometheus-stack
  namespace: monitoring
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /prometheus
            pathType: Prefix
            backend:
              service:
                name: monitoring-kube-prometheus-prometheus
                port:
                  number: 9090
          - path: /grafana
            pathType: Prefix
            backend:
              service:
                name: monitoring-grafana
                port:
                  number: 80
          - path: /alertmanager
            pathType: Prefix
            backend:
              service:
                name: monitoring-kube-prometheus-alertmanager
                port:
                  number: 9093
```

Apply it:

```bash
kubectl apply -f ingress_kube_prom_stack.yaml
```

Validate:

```bash
kubectl get ingress -n monitoring
kubectl describe ingress kubernetes-prometheus-stack -n monitoring
```

---

## 6. Useful Validation Commands

### Monitoring namespace

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
kubectl describe ingress kubernetes-prometheus-stack -n monitoring
```

### Controller namespace

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system deploy/aws-load-balancer-controller
```

---

## 7. Common Mistakes Seen During Troubleshooting

### Typo in namespace

Wrong:

```bash
kubectl get ingress -n mnonitoring
kubectl get ingress -n moniotoring
```

Correct:

```bash
kubectl get ingress -n monitoring
```

### Wrong AWS region

Wrong:

```bash
--set region=us-east-1a
```

Correct:

```bash
--set region=us-east-1
```

### Wrong backend service names in Ingress

Wrong placeholders:

- `prometheus-service`
- `grafana-service`
- `alertmanager-service`

Correct services:

- `monitoring-kube-prometheus-prometheus`
- `monitoring-grafana`
- `monitoring-kube-prometheus-alertmanager`

### Missing namespace in Ingress manifest

Always set:

```yaml
namespace: monitoring
```

### Service port mismatch

Grafana service port is:

```bash
80
```

Not:

```bash
3000
```

Ingress must use the **service port**, not the container port.

---

## 8. Quick Recovery Flow

If ALB is not created:

1. Check controller pods are running
2. Check controller logs
3. Verify controller region is correct
4. Verify Ingress class is `alb`
5. Verify backend service names exist
6. Verify service ports match
7. Re-apply ingress
8. Re-check `kubectl describe ingress`

---

## 9. Handy Command Summary

```bash
kubectl apply -f ingress_kube_prom_stack.yaml -n monitoring
kubectl get all -n monitoring
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
kubectl describe ingress kubernetes-prometheus-stack -n monitoring

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=observability \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<Account-ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm upgrade -n kube-system aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=observability \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=<VPC-ID>

kubectl get pods -n kube-system | grep aws-load-balancer-controller
kubectl logs -n kube-system deploy/aws-load-balancer-controller
kubectl get deployment aws-load-balancer-controller -n kube-system -o yaml | grep -i region -A2 -B2

kubectl get svc -n monitoring
kubectl apply -f ingress_kube_prom_stack.yaml
kubectl get ingress -n monitoring
kubectl describe ingress kubernetes-prometheus-stack -n monitoring
```

---

## 10. Final Notes

- Deploy order usually does **not** require restarting everything.
- The AWS Load Balancer Controller reconciles existing Ingress resources after it starts.
- Most failures come from:
  - wrong AWS region
  - incorrect Ingress backend service names
  - missing namespace
  - incorrect service ports
  - IAM/service account issues
