## Nginx Deployment

---

### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cluster-health
```

---

### ConfigMap & TLS Secret
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: cluster-health
data:
  enable-vts-status: "true"
  proxy-body-size: "20m"
  ssl-protocols: "TLSv1.2 TLSv1.3"
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nginx-tls
  namespace: cluster-health
type: kubernetes.io/tls
data:
  tls.crt: <BASE64_CERT_HERE>
  tls.key: <BASE64_KEY_HERE>
```

---

### Persistent Volume Claim
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-health-pvc
  namespace: cluster-health
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-rbd
```

---

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-health
  namespace: cluster-health
  labels:
    app: nginx-health
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-health
  template:
    metadata:
      labels:
        app: nginx-health
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-storage
              mountPath: /var/log/nginx
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
        - name: storage-checker
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Starting storage health monitor..."
              while true; do
                echo "storage test $(date)" > /var/log/nginx/testfile.txt && cat /var/log/nginx/testfile.txt > /dev/null || echo "Storage I/O error at $(date)";
                sleep 10;
              done
          volumeMounts:
            - name: nginx-storage
              mountPath: /var/log/nginx
      volumes:
        - name: nginx-storage
          persistentVolumeClaim:
            claimName: nginx-health-pvc
```

---

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-health-svc
  namespace: cluster-health
spec:
  selector:
    app: nginx-health
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-health-ingress
  namespace: cluster-health
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: nginx-health.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-health-svc
                port:
                  number: 80
```

---

### ServiceAccount & Role
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-health-sa
  namespace: cluster-health
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-health-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "secrets", "endpoints", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: nginx-health-rb
  namespace: cluster-health
subjects:
  - kind: ServiceAccount
    name: nginx-health-sa
    namespace: cluster-health
roleRef:
  kind: ClusterRole
  name: cluster-health-role
  apiGroup: rbac.authorization.k8s.io
```  
