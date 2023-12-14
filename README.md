# Tracing reconliliation

## Test using Knative Services

```bash
kubectl patch cm config-autoscaler -n knative-serving -p '{"data": {"allow-zero-initial-scale": "true", "initial-scale": "0"}}'

kubectl create ns knative-performance
for i in {1..100}
do
   kn service create "hello-go-$i" --image=gcr.io/knative-samples/helloworld-go -n knative-performance &
done
```

## Test using Pod/Service and KIngresses

```bash
kubectl create ns knative-performance

# Enable profiling
kubectl patch cm config-observability -n knative-serving -p '{"data": {"profiling.enable": "true"}}'

# Create a pod and a service
kubectl create -f yaml/pod-svc.yaml

# Create a multiple KIngresses pointing to that service
for i in {1..1}
do
  cat <<-EOF | kubectl apply -f -
apiVersion: networking.internal.knative.dev/v1alpha1
kind: Ingress
metadata:
  annotations:
    networking.knative.dev/ingress.class: kourier.ingress.networking.knative.dev
  labels:
    serving.knative.dev/route: app-$i
    serving.knative.dev/routeNamespace: knative-performance
    serving.knative.dev/service: app-$i
  name: app-$i
  namespace: knative-performance
spec:
  httpOption: Enabled
  rules:
  - hosts:
    - app-$i.knative-performance
    - app-$i.knative-performance.svc
    - app-$i.knative-performance.svc.cluster.local
    http:
      paths:
      - splits:
        - appendHeaders:
            Knative-Serving-Namespace: knative-performance
            Knative-Serving-Revision: app-$i-00001
          percent: 100
          serviceName: test-svc
          serviceNamespace: knative-performance
          servicePort: 443
    visibility: ClusterLocal
  - hosts:
    - app-$i.knative-performance.172.17.0.100.sslip.io
    http:
      paths:
      - splits:
        - appendHeaders:
            Knative-Serving-Namespace: knative-performance
            Knative-Serving-Revision: app-$i-00001
          percent: 100
          serviceName: test-svc
          serviceNamespace: knative-performance
          servicePort: 443
    visibility: ExternalIP
EOF
done
```
