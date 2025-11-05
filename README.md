# Развертывание Rancher RKE2

## Топология

* **w-rke1** (server/control-plane): `10.0.0.161`
* **w-rke2** (worker): `10.0.0.162`
* **w-rke3** (worker): `10.0.0.163`
* **Общий endpoint API**: `10.0.0.161.sslip.io`
  Сертификаты Kubernetes API должны включать и это имя, и IP `10.0.0.161` (через `tls-san`).
* **OS**: Ubuntu 24.04  

---

## 1) Пререквизиты (выполнить на **всех трёх** ВМ)

```bash
# Базовые пакеты
sudo apt-get update -y
sudo apt-get install -y apparmor apparmor-utils curl jq

# Отключить swap
sudo swapoff -a
sudo sed -ri 's/^\s*([^#]\S*\s+swap\s+\S+\s+\S+.*)$/# \1/' /etc/fstab

# Модули и sysctl для сети Kubernetes
cat <<'EOF' | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
sudo modprobe br_netfilter overlay

cat <<'EOF' | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Если UFW включён – открыть нужные порты
sudo ufw allow 6443/tcp   # Kubernetes API (к серверу)
sudo ufw allow 9345/tcp   # Регистрация RKE2 (к серверу)
sudo ufw allow 8472/udp   # Flannel VXLAN (между всеми узлами)
```

---

## 2) Установка **сервера** на w-rke1 (10.0.0.161)

### 2.1 Конфиг с SAN для имени и IP

```bash
sudo mkdir -p /etc/rancher/rke2
cat <<'EOF' | sudo tee /etc/rancher/rke2/config.yaml
write-kubeconfig-mode: "0644"

# Сертификат API должен быть валиден для имени и IP
tls-san:
  - 10.0.0.161
  - 10.0.0.161.sslip.io
  - w-rke1

# Фиксация адреса/имени узла
node-ip: 10.0.0.161
node-name: w-rke1

# (Необязательно, задать ДО первого запуска)
# cluster-cidr: 10.42.0.0/16
# service-cidr: 10.43.0.0/16
# cni: canal
EOF
```

### 2.2 Установка и запуск

```bash
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server
sudo systemctl start rke2-server
```

> **Нормально ли, что старт “висит” 3–5 минут?** Да. Первый запуск тянет образы и поднимает etcd, обычно это несколько минут (зависит от сети). Ваши ресурсы (4 vCPU / 8 GB) подходят. Чтобы видеть прогресс:

```bash
sudo journalctl -u rke2-server -f -n 100
```

Если сильно дольше — проверьте доступ к реестрам, DNS/прокси, фаервол (6443/tcp, 9345/tcp).

### 2.3 kubeconfig + внешний endpoint

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/rke2-config
sudo ln -s ~/.kube/rke2-config ~/.kube/config
sudo chown "$(id -u)":"$(id -g)" ~/.kube/config
sudo ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl 2>/dev/null || true

# Указать общий endpoint (мы добавили его в tls-san)
kubectl config set-cluster default --server="https://10.0.0.161.sslip.io:6443"
```

### 2.4 Получить токен для присоединения воркеров

```bash
sudo cat /var/lib/rancher/rke2/server/node-token
```

---

## 3) Установка **агентов** (воркеров)

### 3.1 w-rke2 (10.0.0.162)

>** Выполнить раздел 1, если еще не выполнен!!! **

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
sudo mkdir -p /etc/rancher/rke2
cat <<'EOF' | sudo tee /etc/rancher/rke2/config.yaml
server: https://10.0.0.161:9345
token: <ВСТАВЬТЕ_NODE_TOKEN_ОТСЮДА:/var/lib/rancher/rke2/server/node-token:с мастера>

node-ip: 10.0.0.162
node-name: w-rke2
EOF
sudo systemctl enable rke2-agent
sudo systemctl start rke2-agent
sudo journalctl -u rke2-agent -f -n 100
# Курим 5-10 минут - оно там образы по гигу скачивает, так что не мгновенно!
```

### 3.2 w-rke3 (10.0.0.163)

>** Выполнить раздел 1, если еще не выполнен!!! **

```bash
curl -sfL https://get.rke2.io | sudo INSTALL_RKE2_TYPE="agent" sh -
sudo mkdir -p /etc/rancher/rke2
cat <<'EOF' | sudo tee /etc/rancher/rke2/config.yaml
server: https://10.0.0.161:9345
token: <ВСТАВЬТЕ_NODE_TOKEN_ОТСЮДА:/var/lib/rancher/rke2/server/node-token:с мастера>

node-ip: 10.0.0.163
node-name: w-rke3
EOF
sudo systemctl enable rke2-agent
sudo systemctl start rke2-agent
sudo journalctl -u rke2-agent -f -n 100
# Курим 5-10 минут - оно там образы по гигу скачивает, так что не мгновенно!
```

---

## 4) Проверка кластера

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

```bash
# Пример вывода:
# Все ноды должны быть Ready!
MacBook-Pro ~/.kube % kubectl get nodes -o wide
kubectl get pods -A -o wide
NAME     STATUS   ROLES                       AGE   VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
w-rke1   Ready    control-plane,etcd,master   41m   v1.33.5+rke2r1   10.0.0.161    <none>        Ubuntu 24.04.3 LTS   6.8.0-45-generic   containerd://2.1.4-k3s2
w-rke2   Ready    <none>                      24m   v1.33.5+rke2r1   10.0.0.162    <none>        Ubuntu 24.04.3 LTS   6.8.0-45-generic   containerd://2.1.4-k3s2
w-rke3   Ready    <none>                      14m   v1.33.5+rke2r1   10.0.0.163    <none>        Ubuntu 24.04.3 LTS   6.8.0-45-generic   containerd://2.1.4-k3s2
NAMESPACE            NAME                                                   READY   STATUS      RESTARTS      AGE   IP           NODE     NOMINATED NODE   READINESS GATES
kube-system          cloud-controller-manager-w-rke1                        1/1     Running     0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          etcd-w-rke1                                            1/1     Running     1 (42m ago)   41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          helm-install-rke2-canal-gk2jx                          0/1     Completed   0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          helm-install-rke2-coredns-fwrfj                        0/1     Completed   0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          helm-install-rke2-ingress-nginx-9ddqs                  0/1     Completed   0             41m   10.42.0.2    w-rke1   <none>           <none>
kube-system          helm-install-rke2-metrics-server-slbd2                 0/1     Completed   0             41m   10.42.0.7    w-rke1   <none>           <none>
kube-system          helm-install-rke2-runtimeclasses-zkszd                 0/1     Completed   0             41m   10.42.0.4    w-rke1   <none>           <none>
kube-system          helm-install-rke2-snapshot-controller-crd-gfmgw        0/1     Completed   0             41m   10.42.0.8    w-rke1   <none>           <none>
kube-system          helm-install-rke2-snapshot-controller-mgd9g            0/1     Completed   1             41m   10.42.0.5    w-rke1   <none>           <none>
kube-system          kube-apiserver-w-rke1                                  1/1     Running     6 (42m ago)   41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          kube-controller-manager-w-rke1                         1/1     Running     0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          kube-proxy-w-rke1                                      1/1     Running     0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          kube-proxy-w-rke2                                      1/1     Running     0             24m   10.0.0.162   w-rke2   <none>           <none>
kube-system          kube-proxy-w-rke3                                      1/1     Running     0             14m   10.0.0.163   w-rke3   <none>           <none>
kube-system          kube-scheduler-w-rke1                                  1/1     Running     0             41m   10.0.0.161   w-rke1   <none>           <none>
kube-system          rke2-canal-462dz                                       2/2     Running     0             40m   10.0.0.161   w-rke1   <none>           <none>
kube-system          rke2-canal-hs6gg                                       2/2     Running     0             24m   10.0.0.162   w-rke2   <none>           <none>
kube-system          rke2-canal-vqvmr                                       2/2     Running     0             14m   10.0.0.163   w-rke3   <none>           <none>
kube-system          rke2-coredns-rke2-coredns-6464f98784-wx9kw             1/1     Running     0             40m   10.42.0.6    w-rke1   <none>           <none>
kube-system          rke2-coredns-rke2-coredns-6464f98784-zdq6q             1/1     Running     0             17m   10.42.1.3    w-rke2   <none>           <none>
kube-system          rke2-coredns-rke2-coredns-autoscaler-67bb49dff-dt89z   1/1     Running     0             40m   10.42.0.3    w-rke1   <none>           <none>
kube-system          rke2-ingress-nginx-controller-4vlxj                    1/1     Running     0             37m   10.42.0.12   w-rke1   <none>           <none>
kube-system          rke2-ingress-nginx-controller-ppmmg                    1/1     Running     0             10m   10.42.2.2    w-rke3   <none>           <none>
kube-system          rke2-ingress-nginx-controller-tkr6s                    1/1     Running     0             17m   10.42.1.2    w-rke2   <none>           <none>
kube-system          rke2-metrics-server-75d485c65b-f7pbw                   1/1     Running     0             38m   10.42.0.9    w-rke1   <none>           <none>
kube-system          rke2-snapshot-controller-696989ffdd-76ndb              1/1     Running     0             38m   10.42.0.11   w-rke1   <none>           <none>
local-path-storage   local-path-provisioner-7d6dddf9dd-s85cq                1/1     Running     0             15m   10.42.1.4    w-rke2   <none>           <none>

```

---

## 5) Быстрый StorageClass (local-path-provisioner)

Установить динамический провиженер локального диска:

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.32/deploy/local-path-storage.yaml
kubectl -n local-path-storage rollout status deploy/local-path-provisioner
kubectl get storageclass
```

Ищем SC с именем `local-path`. Если он **не** помечен по умолчанию, можно сделать:

```bash
kubectl patch storageclass local-path -p \
'{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

> **По умолчанию путь на ноде** — `/opt/local-path-provisioner` (можно менять в манифесте провиженера).

---

## 6) Пример PVC + Pod для `local-path`

Ниже — минимальный пример: создаём 1Gi PVC и Pod, который монтирует том в `/data` и пишет туда штамп времени.
Сохраните как `pvc-pod-local-path.yaml` и примените.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
---
apiVersion: v1
kind: Pod
metadata:
  name: demo-local-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
              - w-rke2
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "date >> /data/out.txt; echo 'started'; tail -f /dev/null"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: demo-local-pvc
```

Применяем и проверяем:

```bash
kubectl apply -f pvc-pod-local-path.yaml
kubectl get pvc
kubectl get pod demo-local-pod -o wide
kubectl exec -it demo-local-pod -- sh -c 'cat /data/out.txt'
```

Проверьте, что файл существует и содержит время запуска.
Дальше можно удалить Pod (`kubectl delete pod demo-local-pod`) и создать его снова — содержимое останется (PVC и PV не удалялись).
Учтите, что такой том привязан к **конкретной ноде**: если Pod попытается стартовать на другой ноде, монтирование не получится.


---

## Конфиг crictl (на всех нодах)

```bash
sudo tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
timeout: 10
debug: false
EOF

ln -s /var/lib/rancher/rke2/bin/crictl /usr/local/bin/
# Потом можно смотреть контейнеры в container-runtime
crictl ps
```

## Быстрые советы по неполадкам

* `server`/`agent` долго стартуют: смотрите `journalctl -u rke2-... -f`, проверьте сеть к Docker Hub/реестрам, DNS, прокси/фаервол.
* Ноды `NotReady`: чаще всего блокировка UDP/8472 (VXLAN) или вмешательство NetworkManager/файрвола.
* Проблемы с PVC: `kubectl describe pvc/pv/pod` подскажет детали привязки и ошибки провиженинга.
