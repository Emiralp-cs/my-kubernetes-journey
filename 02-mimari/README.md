# 02 - Mimari

Bir Kubernetes kümesi (cluster) iki tür makineden oluşur: kümeyi **yöneten** control plane ve uygulamaları **çalıştıran** worker node'lar.

```
                   Kubernetes kümesi
 ┌───────────────────────────┐   ┌────────────────┐ ┌────────────────┐
 │      Control plane        │   │  Worker node 1 │ │  Worker node 2 │
 │                           │   │                │ │                │
 │  kube-apiserver  ◄────────┼───┤  kubelet       │ │  kubelet       │
 │    ▲    ▲    ▲            │   │  kube-proxy    │ │  kube-proxy    │
 │  etcd  scheduler          │   │  containerd    │ │  containerd    │
 │        controller-manager │   │  [pod] [pod]   │ │  [pod] [pod]   │
 └───────────────────────────┘   └────────────────┘ └────────────────┘
```

> [!TIP]
> **Benzetme:** Bir fabrika düşün. Yönetim katında siparişleri alan bir sekreter (api-server), her şeyin kaydedildiği bir arşiv (etcd), işi hangi makineye vereceğine karar veren bir planlamacı (scheduler) ve hedeflerin tutup tutmadığını sürekli kontrol eden müdürler (controller'lar) var. Üretim katında ise makineler (node'lar), her makinenin başında bir ustabaşı (kubelet) ve makineler arası taşımayı yapan bir lojistikçi (kube-proxy) çalışıyor. Yönetim katıyla ilgili herkes sadece sekreterle konuşur.

## Control plane bileşenleri

| Bileşen | Görevi |
|---|---|
| **kube-apiserver** | Kümenin giriş kapısı. `kubectl` dahil bütün istekler buraya gelir. Kimin ne yapabileceğini kontrol eder (kimlik doğrulama ve yetkilendirme). Diğer bütün bileşenler birbiriyle değil, api-server üzerinden konuşur |
| **etcd** | Kümenin hafızası. Bütün nesnelerin ve ayarların tutulduğu anahtar-değer veritabanı. **Sadece api-server** etcd'ye yazar ve okur |
| **kube-scheduler** | Yeni oluşturulan ve henüz bir node'a atanmamış pod'ları izler. Kaynak ihtiyacına ve kurallara göre uygun node'u seçer. Pod'u kendisi başlatmaz, sadece "bu pod şu node'da çalışsın" kararını verir |
| **kube-controller-manager** | Mevcut durumu sürekli istenen durumla karşılaştıran döngüleri (controller'ları) çalıştırır. Örneğin "3 kopya istenmiş ama 2 tane var" görünce yenisini oluşturur. Node, Deployment, Job gibi her konunun kendi controller'ı var |
| **cloud-controller-manager** | Bulut sağlayıcısıyla (AWS, Azure, GCP) konuşan kısım: yük dengeleyici açmak, disk bağlamak gibi. Kendi bilgisayarında kurduğun kümelerde genelde yoktur ya da bir işe yaramaz |

## Worker node bileşenleri

| Bileşen | Görevi |
|---|---|
| **Container runtime** | Container'ları gerçekten çalıştıran yazılım. Bugün genelde **containerd** veya **CRI-O**. Kubernetes onlarla CRI adlı standart bir arayüzle konuşur |
| **kubelet** | Her node'da çalışan ajan. Kendi node'una atanan pod'ları api-server'dan öğrenir, container'larını runtime'a başlattırır ve durumlarını api-server'a bildirir |
| **kube-proxy** | Node'lardaki ağ kurallarını yönetir. Bir Service'e gelen trafiğin doğru pod'lara ulaşmasını sağlar |

> [!NOTE]
> **Docker ile ilişkisi:** Kubernetes eskiden container'ları Docker üzerinden çalıştırabiliyordu. Bu destek 2022'de (v1.24) kaldırıldı ve bugün containerd veya CRI-O kullanılıyor. Docker ile oluşturduğun imajlar ise aynen çalışır: imaj biçimi (OCI) standart.

## Bir pod nasıl ayağa kalkar?

`kubectl apply -f pod.yaml` dediğinde arka planda şunlar olur:

1. **kube-apiserver** isteği alır, yetkini kontrol eder, pod tanımını **etcd**'ye yazar. Pod henüz hiçbir node'a atanmamıştır.
2. **kube-scheduler** atanmamış pod'u görür, uygun bir node seçer ve kararını api-server'a bildirir.
3. O node'daki **kubelet** kendisine yeni bir pod atandığını görür ve container runtime'a container'ları başlattırır.
4. **kubelet** pod'un durumunu (çalışıyor, IP adresi...) api-server'a bildirir, api-server bunu etcd'ye yazar.

Fark ettiysen hiçbir bileşen diğerine doğrudan emir vermiyor. Herkes api-server üzerinden durumu izleyip kendi işini yapıyor.

## Imperative ve declarative

Kubernetes'e iki şekilde iş yaptırılabilir:

| | Imperative (emir kipi) | Declarative (bildirimsel) |
|---|---|---|
| Ne söylersin? | "Şunu yap, sonra şunu yap" | "Böyle bir durum istiyorum" |
| Örnek | `kubectl create`, `kubectl scale` | Bir YAML dosyası ve `kubectl apply` |
| Bir şey bozulursa? | Düzeltmek sana kalır | Kubernetes istenen duruma kendisi döner |

> [!TIP]
> **Benzetme:** Taksiye binip "sağa dön, iki sokak düz git, ışıktan sola" demek imperative. "Beni Kadıköy'e götür" demek declarative: rotayı şoför bulur, yol kapalıysa başka yoldan gider.

Declarative yaklaşımı gerçek bir kümede gördük. 3 kopyalı bir uygulama oluşturup kopyalardan birini sildik:

```bash
kubectl create deployment web --image=nginx:1.30-alpine --replicas=3
kubectl delete pod web-6f5d6d9c94-czmcw
kubectl get pods -l app=web
```
```
NAME                   READY   STATUS    RESTARTS   AGE
web-6f5d6d9c94-hr786   1/1     Running   0          3s
web-6f5d6d9c94-jwfcc   1/1     Running   0          4s
web-6f5d6d9c94-sm7q6   1/1     Running   0          4s
```

Silinen pod'un yerine 3 saniye içinde yenisi (`hr786`, yaşı 3 saniye) gelmiş. İstenen durum "3 kopya" olduğu için controller eksiği hemen tamamladı. Deployment konusuna ileride ayrıca gelinecek.

## Kümenin bileşenlerini görmek

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

Bu komutların çıktısı kümenin nasıl kurulduğuna göre değişir:

- **kubeadm** veya **minikube** ile kurulan kümelerde control plane bileşenleri `kube-system` içinde ayrı pod'lar olarak görünür: `etcd-...`, `kube-apiserver-...`, `kube-controller-manager-...`, `kube-scheduler-...`, `kube-proxy-...`, `coredns-...`
- Bu repodaki örneklerin denendiği **k3s** gibi hafif dağıtımlar ise control plane'i tek bir program içinde çalıştırır. Orada bu pod'ları göremezsin, sadece CoreDNS gibi eklentiler görünür:

```
NAME     STATUS   ROLES           AGE    VERSION
dugum1   Ready    control-plane   4m1s   v1.37.2+k3s1
```

- **EKS, AKS, GKE** gibi bulut servislerinde control plane'i bulut sağlayıcısı yönetir ve sana hiç gösterilmez. Sen sadece worker node'ları görürsün.

---

Önceki: [01 - Neden Kubernetes?](../01-neden-kubernetes/) · Sonraki: [03 - kubeconfig ve Context](../03-kubeconfig/)
