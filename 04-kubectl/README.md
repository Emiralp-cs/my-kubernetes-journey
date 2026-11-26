# 04 - kubectl Komutları

`kubectl` komutları hep aynı kalıpla yazılır:

```
kubectl <fiil> <nesne türü> [isim] [seçenekler]

kubectl get pods
kubectl describe pod web
kubectl delete pod web
```

Fiiller: `get`, `describe`, `create`, `apply`, `edit`, `delete`, `logs`, `exec`... Nesne türleri: `pod`, `deployment`, `service`, `node`, `namespace`...

Örneklerde iki pod çalışıyor:

```bash
kubectl run web --image=nginx:1.30-alpine --port=80 --labels="uygulama=web,ekip=altyapi"
kubectl run araci --image=alpine:3.24 --labels="uygulama=araci" -- sleep 3600
```

## Küme bilgisi

```bash
kubectl version
kubectl cluster-info
```
```
Client Version: v1.37.2+k3s1
Kustomize Version: v5.8.1
Server Version: v1.37.2+k3s1

Kubernetes control plane is running at https://127.0.0.1:6443
```

`Client` senin kubectl'in, `Server` kümenin sürümü. Aralarında en fazla bir minor sürüm fark olması önerilir. Fark büyürse bazı komutlar beklenmedik davranabilir.

## Listelemek: get

| Komut | Ne getirir? |
|---|---|
| `kubectl get pods` | Aktif namespace'teki pod'lar |
| `kubectl get pods web` | Sadece `web` adlı pod |
| `kubectl get pods -n kube-system` | `kube-system` namespace'indeki pod'lar |
| `kubectl get pods -A` | Bütün namespace'lerdeki pod'lar (`--all-namespaces`) |
| `kubectl get pods -l uygulama=web` | Etiketi `uygulama=web` olanlar |
| `kubectl get pods --show-labels` | Etiketleriyle birlikte |
| `kubectl get pods -w` | Değişiklikleri canlı izle (`Ctrl+C` ile çık) |

Aynı kalıp bütün nesne türleri için çalışır: `kubectl get nodes`, `kubectl get services -A`, `kubectl get deployments`...

### Nesnelerin kısa adları

```bash
kubectl api-resources
```
```
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
configmaps    cm           v1           true         ConfigMap
namespaces    ns           v1           false        Namespace
nodes         no           v1           false        Node
pods          po           v1           true         Pod
services      svc          v1           true         Service
deployments   deploy       apps/v1      true         Deployment
```

(Çıktı kısaltıldı.) `kubectl get po` ile `kubectl get pods` aynıdır. `NAMESPACED` sütunu, nesnenin bir namespace'e ait olup olmadığını gösterir: pod'lar aittir, node'lar ait değildir.

## Çıktı biçimleri: -o

| Seçenek | Ne döndürür? |
|---|---|
| `-o wide` | Ek sütunlarla tablo (IP, node) |
| `-o yaml` / `-o json` | Nesnenin bütün alanları |
| `-o name` | Sadece `tür/isim` |
| `-o custom-columns=...` | Kendi seçtiğin sütunlar |
| `-o jsonpath=...` | Bir şablona göre seçilen alanlar |

```bash
kubectl get pods -o wide
```
```
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
araci   1/1     Running   0          1s    10.42.0.49   dugum1   <none>           <none>
web     1/1     Running   0          1s    10.42.0.48   dugum1   <none>           <none>
```

```bash
kubectl get pods -o name
```
```
pod/araci
pod/web
```

### Kendi sütunlarını seçmek

```bash
kubectl get pods -o custom-columns=AD:.metadata.name,IMAJ:.spec.containers[0].image,IP:.status.podIP
```
```
AD      IMAJ                IP
araci   alpine:3.24         10.42.0.49
web     nginx:1.30-alpine   10.42.0.48
```

Her sütun `BAŞLIK:.alanın.yolu` biçiminde yazılır. Alan yolları `-o yaml` çıktısındaki yapıyla aynıdır. Bir alanın nerede olduğunu bulmak için önce `kubectl get pod web -o yaml`'a bak.

### jsonpath ve jq

```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```
```
araci web
```

Her pod'u ayrı satıra yazmak için `range`:

```bash
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
```
```
araci	10.42.0.49
web	10.42.0.48
```

Aynı işi JSON sorgulama aracı **jq** ile de yapabilirsin (`brew install jq` veya `sudo apt install jq`). Bütün kümede hangi imajların kullanıldığı:

```bash
kubectl get pods -A -o json | jq -r '.items[].spec.containers[].image' | sort -u
```
```
alpine:3.24
nginx:1.30-alpine
rancher/klipper-helm:v0.13.3-build20261021
rancher/local-path-provisioner:v0.0.37
rancher/mirrored-coredns-coredns:1.14.7
```

jsonpath kubectl'in içinde gelir, kurulum gerektirmez. jq daha esnektir ama ayrıca kurulması gerekir.

## Yardım ve açıklama: explain

`kubectl apply --help` bir komutun nasıl kullanıldığını anlatır. Ama `pod` bir komut değil, bir nesnedir:

```bash
kubectl pod --help
```
```
error: unknown command "pod" for "kubectl"
```

Nesneler için `explain` kullanılır:

```bash
kubectl explain pod
```
```
KIND:       Pod
VERSION:    v1

DESCRIPTION:
    Pod is a collection of containers that can run on a host. ...
```

`VERSION`, YAML dosyasında `apiVersion` alanına yazılacak değerdir. Bir alanın içine inmek için nokta kullanılır:

```bash
kubectl explain pod.spec.containers.image
kubectl explain pod.spec.containers --recursive    # bütün alt alanlar, ağaç şeklinde
```

YAML yazarken bir alanın tam adını hatırlamadığında bu komut çok işe yarar.

## Her şeyi görmek: get all

```bash
kubectl get all
```
```
NAME        READY   STATUS    RESTARTS   AGE
pod/araci   1/1     Running   0          7s
pod/web     1/1     Running   0          7s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   7m55s
```

Adına rağmen `all` her şeyi göstermez. Pod, Service, Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob gibi temel türleri kapsar. **ConfigMap, Secret, Ingress ve PersistentVolumeClaim listede çıkmaz**, onları ayrıca istemek gerekir.

## Pod'un içine girmek: exec

```bash
kubectl exec web -- hostname                 # tek bir komut çalıştır
kubectl exec -it araci -- sh                 # içinde kabuk aç
```

`--`, kubectl'in kendi seçeneklerinin bittiğini ve container'da çalışacak komutun başladığını gösterir.

Alpine gibi küçük imajlarda `bash` yoktur:

```
exec: "bash": executable file not found in $PATH
```

Bu hatayı alırsan `bash` yerine `sh` dene. Distroless imajlarda ise hiç kabuk yoktur.

## YAML'ı komuta yazdırmak

Sıfırdan YAML yazmak yerine kubectl'e taslak hazırlatabilirsin. `--dry-run=client` hiçbir şey oluşturmaz, sadece çıktıyı yazar:

```bash
kubectl run ornek --image=nginx:1.30-alpine --port=80 --dry-run=client -o yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: ornek
  name: ornek
spec:
  containers:
  - image: nginx:1.30-alpine
    name: ornek
    ports:
    - containerPort: 80
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Sonuna `> pod.yaml` eklersen dosyaya yazar. Sınavlarda ve günlük işte çok zaman kazandırır.

## Komut özeti

| Komut | Ne yapar? |
|---|---|
| `kubectl version` | kubectl ve küme sürümü |
| `kubectl cluster-info` | Control plane adresi |
| `kubectl api-resources` | Bütün nesne türleri ve kısa adları |
| `kubectl get <tür> -A -o wide` | Bütün namespace'lerde, geniş tablo |
| `kubectl get <tür> -o yaml` | Nesnenin bütün alanları |
| `kubectl get <tür> -o custom-columns=AD:.metadata.name` | Kendi sütunların |
| `kubectl get <tür> -o jsonpath='{...}'` | Belirli alanlar |
| `kubectl explain <tür>[.alan]` | Nesne ve alan açıklaması, `apiVersion` |
| `kubectl exec -it <pod> -- sh` | Pod'un içinde kabuk |
| `kubectl run ... --dry-run=client -o yaml` | Hazır YAML taslağı |

---

Önceki: [03 - kubeconfig ve Context](../03-kubeconfig/) · Sonraki: [05 - YAML](../05-yaml/)
