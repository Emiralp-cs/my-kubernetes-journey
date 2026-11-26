# ☸️ My Kubernetes Journey

Kubernetes öğrenirken aldığım notlar ve konu konu ilerleyen küçük, uygulamalı örnekler. Özgür Öztürk hocamın Kubernetes eğitimini takip ederken aldığım notları yapay zeka yardımıyla düzenledim ve her konuyu kendi örneklerimle gerçek bir kümede tekrar ettim. Özgür Öztürk hocama çok teşekkürler. Bu repo [my-docker-journey](https://github.com/Emiralp-cs/my-docker-journey)'nin devamı: orada container'ları tek tek çalıştırmayı öğrendik, burada onları Kubernetes'le yönetmeyi öğreniyoruz.

Her klasör tek bir konuya odaklanıyor: kısa bir anlatım, çalıştırılacak komutlar, gerçek çıktılar, sık yapılan hatalar ve sonunda kendi başına deneyebileceğin küçük alıştırmalar. Örneklerde sadece Alpine ve Nginx imajları var.

## 📚 Konular

| # | Konu | Ne öğreniliyor? |
|---|---|---|
| 01 | [Neden Kubernetes?](01-neden-kubernetes/) | Container orkestrasyonu, kısa tarihçe, sürümler |
| 02 | [Mimari](02-mimari/) | Control plane, worker node, bir pod'un ayağa kalkışı, imperative ve declarative |
| 03 | [kubeconfig ve Context](03-kubeconfig/) | cluster, user, context, kümeler arasında geçiş |
| 04 | [kubectl](04-kubectl/) | `get`, `describe`, `explain`, `exec`, `-o` çıktı biçimleri |
| 05 | [YAML](05-yaml/) | Girinti, değer türleri, tuzaklar, yamllint |
| 06 | [Pod](06-pod/) | `kubectl run`, loglar, `exec`, `port-forward`, silme |
| 07 | [YAML ile Pod](07-pod-yaml/) | `apiVersion`, `kind`, `metadata`, `spec`, `apply` ve `create` |
| 08 | [Pod'un Yaşam Döngüsü](08-pod-yasam-dongusu/) | Fazlar, `restartPolicy`, CrashLoopBackOff, sorun giderme |
| 09 | [Çok Container'lı Pod](09-cok-containerli-pod/) | Sidecar, init container, `localhost` ve ortak volume |

Sırayla gitmek en kolayı: her bölüm bir öncekinin üzerine kuruluyor.

### 🔜 Sırada

Öğrendikçe eklenecek: Label ve Selector, Namespace, Deployment, Service, ConfigMap ve Secret, Volume...

## 🚀 Başlamak için

Bir Kubernetes kümesi ve `kubectl` gerekiyor. Kendi bilgisayarında küçük bir küme kurmanın kolay yolları:

| Araç | Nasıl? |
|---|---|
| [Docker Desktop](https://docs.docker.com/desktop/use-desktop/kubernetes/) | Kubernetes sekmesi → **Create cluster**. Docker zaten kuruluysa en kolayı |
| [minikube](https://minikube.sigs.k8s.io/docs/start/) | `minikube start` |
| [kind](https://kind.sigs.k8s.io/) | `kind create cluster` (Kubernetes'i Docker container'ları içinde çalıştırır) |

```bash
git clone https://github.com/Emiralp-cs/my-kubernetes-journey.git
cd my-kubernetes-journey
kubectl get nodes          # küme hazır mı?
```

Her klasördeki README'yi aç ve komutları kendi terminalinde çalıştır. Okumak yerine yazmak çok daha kalıcı.

Bütün örnekler **Kubernetes v1.37** ile (tek node'lu bir k3s kümesinde) denendi. Çıktılardaki IP adresleri, süreler ve node adı (`dugum1`) senin kümende farklı olacaktır.

## 📖 Faydalı kaynaklar

| Kaynak | Ne için? |
|---|---|
| [kubernetes.io/docs](https://kubernetes.io/docs/home/) | Resmi dokümantasyon. CKA ve CKAD sınavlarında da açık |
| [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/quick-reference/) | En sık kullanılan komutlar tek sayfada |
| [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/) | Her nesnenin bütün alanları |
| [Killercoda](https://killercoda.com/playgrounds) | Tarayıcıda ücretsiz, hazır Kubernetes kümesi |

## 📄 Lisans

[CC0 1.0](LICENSE): İzin almadan, kaynak göstermeden istediğin gibi kullanabilirsin.
