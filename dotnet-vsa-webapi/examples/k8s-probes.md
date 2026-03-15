# Example: Kubernetes probes

This example assumes the app exposes:

- `/health/live`
- `/health/ready`

## Deployment snippet

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shipments-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shipments-api
  template:
    metadata:
      labels:
        app: shipments-api
    spec:
      containers:
        - name: shipments-api
          image: ghcr.io/acme/shipments-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_URLS
              value: http://+:8080
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 2
            failureThreshold: 3
```

## Guidance

- readiness should fail when the instance cannot safely serve traffic
- liveness should fail only when the process is unhealthy enough to restart
- do not put fragile downstream checks into liveness
- do not make probes too aggressive for cold starts or migrations
