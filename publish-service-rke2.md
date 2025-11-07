# Публикация сервиса в RKE2 на bare-metal

## 0) Подготовка (один раз)

* `local-path-provisioner` установлен и **обновлён** так, чтобы тома создавались на `w-rke2:/opt/pods-pvs-here`.
* Проверка что путь на ноде существует:

  ```bash
  ssh w-ansible@w-rke2 'sudo mkdir -p /opt/pods-pvs-here && sudo chmod 0777 /opt/pods-pvs-here'
  ```

---

## 1) Установка MetalLB (controller + speaker)

```bash
./00-enable-metallb.sh
```

**Проверка**

```bash
kubectl -n metallb-system get pods
```

---

## 2) Создание пула адреслв лоад балансера и L2-анонса

```bash
kubectl apply -f 30-metallb-pool.yaml
```

**Проверка**

```bash
kubectl -n metallb-system get ipaddresspools,l2advertisements
```

---

## 3) Публикация встроенного RKE2 ingress-nginx через LoadBalancer

```bash
kubectl apply -f 31-ingress-nginx-lb.yaml
kubectl -n kube-system get svc rke2-ingress-nginx-lb -o wide
```

Должен появиться `EXTERNAL-IP` (в вашем случае: **10.0.0.180**).

---

## 4) Создание неймспейса приложения

```bash
kubectl apply -f 00-namespace.yaml
kubectl get ns sentry
```

---

## 5) Создание TLS-серта/секрета для тестового хоста

```bash
./01-ssl-certs.sh
```

Скрипт создаёт `secret/sentry/nginx-tls` для хоста **nginx.10.0.0.180.sslip.io**.

---

## 6) Деплой приложения с постоянным хранилищем (StatefulSet + Service)

```bash
kubectl apply -f 10-nginx-statefulset-and-service.yaml
```

Этот манифест:

* Закрепляет Pod на **w-rke2** (nodeAffinity),
* Создаёт **PVC** (SC `local-path`), который попадёт в `/opt/pods-pvs-here` на ноде,
* Монтирует PVC **напрямую** в `/usr/share/nginx/html`,
* Открывает **два порта** (80 и 443) внутри контейнера,
* Создаёт **ClusterIP**-сервис (`nginx-svc`) с портами 80/443.

**Проверка**

```bash
kubectl -n sentry get pods,svc,pvc -o wide
kubectl -n sentry describe pod nginx-pv-0 | sed -n '1,50p'
```

Ожидаем `Running` у пода, привязанный PVC и эндпоинты у сервиса.

---

## 7) Публикация через Ingress (на хост с LB-IP)

```bash
kubectl apply -f 20-nginx-ingress.yaml
kubectl -n sentry get ingress nginx
```

**Тест**

```bash
curl -I http://nginx.10.0.0.180.sslip.io/
curl -kI https://nginx.10.0.0.180.sslip.io/
```

Должен быть `HTTP/1.1 200 OK`. В браузере тоже доступно по 80/443.
(HTTPS использует самоподписанный секрет `nginx-tls`.)

---

## Быстрая диагностика (копипаст)

* Путь трафика через Ingress:

  ```bash
  kubectl -n kube-system get svc rke2-ingress-nginx-lb -o wide
  kubectl -n sentry describe ingress nginx
  kubectl -n kube-system get pods -l app.kubernetes.io/name=rke2-ingress-nginx -o wide
  ```

* Приложение/том:

  ```bash
  kubectl -n sentry get pods,svc,pvc -o wide
  kubectl -n sentry logs -l app=nginx-pv --tail=100
  ssh w-rke2 'sudo find /opt/pods-pvs-here -maxdepth 2 -type d -name "pvc-*" -print'
  ```

* Привязка к ноде (если вдруг под окажется не на w-rke2, чего быть не должно):

  ```bash
  kubectl get node -L kubernetes.io/hostname
  kubectl -n sentry get pod nginx-pv-0 -o wide
  ```

---

## Заметки / что можно менять дальше

* **Смена публичного хоста**: поправьте `20-nginx-ingress.yaml` (имя хоста в `rules` и CN в TLS) и пересоздайте сертификат через `01-ssl-certs.sh`.
* **Масштабирование**: увеличьте `replicas` в StatefulSet (помните про `ReadWriteOnce` и привязку к ноде).
* **Реальный Sentry**: замените контейнер `nginx` на `sentry/sentry` и добавьте Postgres/Redis (и ClickHouse/Snuba) + init-job’ы для миграций; можно оставить ту же схему ingress и хранилища.

---

## Манифесты и скрипты

00-enable-metallb.sh
```bash
#!/bin/bash

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
kubectl -n metallb-system rollout status deploy/controller
```

00-namespace.yaml
```bash
---
apiVersion: v1
kind: Namespace
metadata:
  name: sentry

```

01-ssl-certs.sh
```bash
#!/bin/bash

openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -subj "/CN=nginx.10.0.0.180.sslip.io" \
  -keyout /tmp/nginx.key -out /tmp/nginx.crt
kubectl -n sentry create secret tls nginx-tls --key /tmp/nginx.key --cert /tmp/nginx.crt

```

10-nginx-statefulset-and-service.yaml
```bash
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: sentry
  labels: { app: nginx-pv }
spec:
  selector: { app: nginx-pv }
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
  namespace: sentry
data:
  default.conf: |
    server {
      listen 80;
      server_name _;
      root /usr/share/nginx/html;
      location / { try_files $uri $uri/ =404; }
    }
    server {
      listen 443 ssl;
      server_name _;
      ssl_certificate     /etc/nginx/tls/tls.crt;
      ssl_certificate_key /etc/nginx/tls/tls.key;
      root /usr/share/nginx/html;
      location / { try_files $uri $uri/ =404; }
    }
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-pv
  namespace: sentry
spec:
  serviceName: nginx-svc
  replicas: 1
  selector:
    matchLabels: { app: nginx-pv }
  template:
    metadata:
      labels: { app: nginx-pv }
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values: ["w-rke2"]
      initContainers:
      - name: init-content
        image: busybox:1.36
        command: ["/bin/sh","-lc"]
        args:
          - |
            mkdir -p /work && \
            [ -f /work/index.html ] || echo "Hello from $(hostname) at $(date)" > /work/index.html
        volumeMounts:
        - name: data
          mountPath: /work
      containers:
      - name: nginx
        image: nginx:stable
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html    # <-- mount PVC directly into nginx html dir
        - name: tls
          mountPath: /etc/nginx/tls
          readOnly: true
        - name: conf
          mountPath: /etc/nginx/conf.d
          readOnly: true
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
      volumes:
      - name: tls
        secret:
          secretName: nginx-tls
      - name: conf
        configMap:
          name: nginx-conf
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: local-path
      resources:
        requests:
          storage: 2Gi
```

20-nginx-ingress.yaml
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
  namespace: sentry
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - nginx.10.0.0.180.sslip.io
    secretName: nginx-tls
  rules:
  - host: nginx.10.0.0.180.sslip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```

30-metallb-pool.yaml
```bash
# 30-metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelan
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.180-10.0.0.189   # <--- CHANGE if needed
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelan-l2
  namespace: metallb-system
spec:
  ipAddressPools: ["homelan"]

```

31-ingress-nginx-lb.yaml
```bash
# 31-ingress-nginx-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: rke2-ingress-nginx-lb
  namespace: kube-system
  annotations:
    metallb.universe.tf/address-pool: homelan
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: rke2-ingress-nginx
    app.kubernetes.io/component: controller
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443

```

