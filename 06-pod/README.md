# 06 - Pod

Kubernetes'te çalıştırılan her şeye **nesne** (object) denir. En temel nesne **Pod**'dur. Docker'da doğrudan container oluşturuyorduk. Kubernetes'te ise doğrudan container oluşturulmaz: container'lar bir pod'un içinde çalışır.

- Bir pod bir ya da birden fazla container içerebilir. Genelde her pod'da tek bir ana container olur.
- Her pod'un benzersiz bir kimliği (UID) ve kendine ait bir IP adresi vardır.
- Aynı pod'daki container'lar aynı node'da çalışır ve birbirlerine `localhost` ile ulaşır ([09](../09-cok-containerli-pod/)).

> [!TIP]
> **Benzetme:** Container bir yolcuysa pod onun kamarasıdır. Aynı kamaradaki yolcular aynı kapıyı (IP adresi) ve aynı dolabı (volume) paylaşır, birbirleriyle konuşmak için koridora çıkmaları gerekmez. Gemi (node) değiştiğinde kamara da değişir: pod yeniden oluşturulunca IP'si de değişir.

**UID ve IP ne zaman verilir?** UID, pod oluşturulurken api-server tarafından verilir. IP ise pod bir node'a yerleştirilip container'ları hazırlanırken atanır. Pod silinip yeniden oluşturulunca IP değişir. Bu yüzden uygulamalara pod IP'siyle değil, Service ile erişilir.

## Komutla pod oluşturmak: kubectl run

```bash
kubectl run selam --image=alpine:3.24 --restart=Never -- echo "Merhaba Kubernetes"
kubectl get pod selam
kubectl logs selam
```
```
NAME    READY   STATUS      RESTARTS   AGE
selam   0/1     Completed   0          2s

Merhaba Kubernetes
```

Container `echo` komutunu çalıştırıp işini bitirdi. `--restart=Never` olduğu için yeniden başlatılmadı ve pod `Completed` durumunda kaldı.

| Seçenek | Anlamı |
|---|---|
| `--image=nginx:1.30-alpine` | Container'ın imajı |
| `--port=80` | Container'ın dinlediği port (bilgi amaçlı) |
| `--labels="uygulama=web,ekip=altyapi"` | Pod'a etiketler |
| `--restart=Never` | Container durursa yeniden başlatma. Diğer değerler: `Always` (varsayılan), `OnFailure` |
| `-- komut` | `--`'dan sonrası container'da çalışacak komut |

Etiketler (labels) `anahtar=deger` biçimindedir, birden fazlası virgülle ayrılır:

```bash
kubectl run web --image=nginx:1.30-alpine --port=80 --labels="uygulama=web,ekip=altyapi"
kubectl get pods --show-labels
```
```
NAME    READY   STATUS    RESTARTS   AGE   LABELS
web     1/1     Running   0          1s    ekip=altyapi,uygulama=web
```

Aynı isimle ikinci bir pod oluşturulamaz:

```
Error from server (AlreadyExists): pods "web" already exists
```

## Pod'u incelemek

```bash
kubectl get pods -o wide        # IP ve node da görünür
kubectl describe pod web        # bütün ayrıntılar
```

`describe` çıktısının sonundaki **Events** bölümü, Kubernetes'in pod için sırayla ne yaptığını gösterir. Bir pod çalışmıyorsa ilk bakılacak yer burası:

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  23s   default-scheduler  Successfully assigned default/web to dugum1
  Normal  Pulled     23s   kubelet            spec.containers{web}: Container image "nginx:1.30-alpine" already present on machine ...
  Normal  Created    23s   kubelet            spec.containers{web}: Container created
  Normal  Started    23s   kubelet            spec.containers{web}: Container started
```

Sıra, [02](../02-mimari/)'deki akışın aynısı: scheduler node'u seçer, kubelet imajı hazırlar, container'ı oluşturur ve başlatır.

## Loglar ve komut çalıştırmak

```bash
kubectl logs web                     # container'ın çıktısı
kubectl logs -f web                  # canlı takip (Ctrl+C ile çık)

kubectl exec web -- ls /             # tek bir komut
kubectl exec web -- printenv         # ortam değişkenleri
kubectl exec -it web -- sh           # kabuk aç, exit ile çık
```

`printenv` çıktısında `HOSTNAME=web`: pod'un hostname'i adıyla aynı. Kubernetes ayrıca her container'a `KUBERNETES_SERVICE_HOST` gibi küme bilgilerini ortam değişkeni olarak verir.

> [!WARNING]
> `--` ile komut arasındaki boşluğu unutma. `kubectl exec web -c web --printenv` yazınca kubectl `--printenv`'i kendi seçeneği sanar:
> ```
> error: unknown flag: --printenv
> ```
> Doğrusu: `kubectl exec web -c web -- printenv`

## Değişiklikleri canlı izlemek: -w

```bash
kubectl get pods -w
```

Başka bir terminalden yeni bir pod oluşturulunca:

```
NAME    READY   STATUS              RESTARTS   AGE
izle    0/1     Pending             0          0s
izle    0/1     ContainerCreating   0          0s
izle    1/1     Running             0          1s
```

`-w` (watch) çıktıyı bir kez basıp bitmek yerine açık kalır ve her değişikliği yeni bir satır olarak ekler.

## Pod'a tünel açmak: port-forward

Pod'un IP'si sadece kümenin içinden erişilebilir. Kendi bilgisayarından denemek için:

```bash
kubectl port-forward pod/web 8080:80
```
```
Forwarding from 127.0.0.1:8080 -> 80
Handling connection for 8080
```

Komut açık kaldığı sürece tarayıcıda **http://localhost:8080** pod'daki Nginx'i açar. `Ctrl+C` ile tünel kapanır. Kalıp: `tür/isim yerel_port:pod_portu`. Bu bir test aracıdır. Uygulamayı gerçekten dışarıya açmak Service ile yapılır.

## Silmek

```bash
kubectl delete pod web
```

Silme komutu onay sormaz. Canlı bir ortamda komutu yazmadan önce hangi context ve namespace'te olduğunu kontrol et ([03](../03-kubeconfig/)).

Pod hemen silinmez: Kubernetes container'a durması için sinyal (`SIGTERM`) gönderir ve **30 saniyeye kadar** düzgün kapanmasını bekler. Denemede Nginx pod'u 1 saniyede silindi. `sleep` çalıştıran bir Alpine pod'u ise sinyale cevap vermediği için **31 saniye** sürdü: Kubernetes 30 saniye bekleyip zorla kapattı. Test ortamında beklemek istemezsen:

```bash
kubectl delete pod izle --now                        # 1 saniye bekle
kubectl delete pod izle --grace-period=0 --force     # hiç bekleme (dikkatli kullan)
```

## Komut özeti

| Komut | Ne yapar? |
|---|---|
| `kubectl run <pod> --image=<imaj>` | Pod oluştur |
| `kubectl run <pod> --image=<imaj> --restart=Never -- <komut>` | Bir komut çalıştırıp biten pod |
| `kubectl get pods -o wide --show-labels` | IP, node ve etiketlerle listele |
| `kubectl get pods -w` | Canlı izle |
| `kubectl describe pod <pod>` | Ayrıntılar ve Events |
| `kubectl logs [-f] <pod>` | Loglar |
| `kubectl exec -it <pod> -- sh` | İçinde kabuk aç |
| `kubectl port-forward pod/<pod> 8080:80` | Bilgisayarından pod'a tünel |
| `kubectl delete pod <pod>` | Sil |

## Dene

1. `merhaba` adında, `alpine:3.24` imajıyla, ekrana bugünün tarihini yazıp biten bir pod oluştur (`-- date`). Logunu oku.
2. `nginx:1.30-alpine` ile `site` adında bir pod oluştur, `port-forward` ile tarayıcıdan aç.
3. `site` pod'unu sil ve aynı komutla tekrar oluştur. `kubectl get pods -o wide` ile IP'sinin değiştiğini gör.

---

Önceki: [05 - YAML](../05-yaml/) · Sonraki: [07 - YAML ile Pod](../07-pod-yaml/)
