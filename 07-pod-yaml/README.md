# 07 - YAML ile Pod

[06](../06-pod/)'da pod'ları `kubectl run` ile oluşturduk. Gerçek işte ise nesneler YAML dosyalarıyla tanımlanır: dosya Git'te saklanır, değişiklikler takip edilir ve aynı dosya her kümede aynı sonucu verir. Bu, [02](../02-mimari/)'de anlatılan declarative yaklaşımdır.

## Dört temel alan

Bütün Kubernetes nesnelerinin YAML'ı aynı dört alanla başlar. Bu klasördeki [`pod.yaml`](pod.yaml):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: karsilama
  labels:
    uygulama: karsilama
    ortam: test
spec:
  containers:
    - name: web
      image: nginx:1.30-alpine
      ports:
        - containerPort: 80
```

| Alan | Ne söyler? |
|---|---|
| `apiVersion` | Nesnenin hangi API grubunda ve sürümünde tanımlı olduğu |
| `kind` | Nesnenin türü: `Pod`, `Service`, `Deployment`... |
| `metadata` | Nesneyi tanıtan bilgiler: `name` (zorunlu), `labels`, `namespace`... |
| `spec` | İstenen durum: pod için hangi container'lar, hangi imajlar, hangi portlar |

> [!TIP]
> **Benzetme:** Bir sipariş formu gibi. `apiVersion` ve `kind` formun hangi form olduğu (ör. "Yemek siparişi, 2. sürüm"), `metadata` siparişi verenin adı ve notları, `spec` ise siparişin kendisi.

Kümeye uygulandıktan sonra Kubernetes bir beşinci alan ekler: `status`. Burası **gerçek durumdur** (pod'un IP'si, container'ların çalışıp çalışmadığı) ve sen yazmazsın, Kubernetes doldurur. `kubectl get pod karsilama -o yaml` ile görebilirsin.

### Doğru apiVersion'ı bulmak

Her türün `apiVersion`'ı farklıdır. Ezberlemek yerine:

```bash
kubectl explain pod | head -3
```
```
KIND:       Pod
VERSION:    v1
```

Deployment için `apps/v1`, Pod ve Service için `v1`. Bütün liste `kubectl api-resources` çıktısındaki `APIVERSION` sütununda ([04](../04-kubectl/)).

## Kümeye uygulamak: apply ve create

```bash
kubectl apply -f pod.yaml
```
```
pod/karsilama created
```

Aynı komutu hiçbir şey değiştirmeden tekrar çalıştırınca:

```
pod/karsilama unchanged
```

Dosyada bir etiketi değiştirip (`ortam: test` → `ortam: canli`) tekrar çalıştırınca:

```
pod/karsilama configured
```

`apply` "dosyadaki hale getir" demektir: nesne yoksa oluşturur, varsa farkı uygular, fark yoksa dokunmaz. Bu yüzden aynı komutu güvenle tekrar tekrar çalıştırabilirsin.

`create` ise sadece oluşturur. Nesne zaten varsa hata verir:

```bash
kubectl create -f pod.yaml
```
```
Error from server (AlreadyExists): error when creating "pod.yaml": pods "karsilama" already exists
```

| | `kubectl create -f` | `kubectl apply -f` |
|---|---|---|
| Nesne yoksa | Oluşturur | Oluşturur |
| Nesne varsa | Hata verir | Farkları uygular |
| Kullanım | Tek seferlik işler | Günlük iş, Git'teki dosyalar |

Silmek için de aynı dosya kullanılabilir: `kubectl delete -f pod.yaml`.

## Pod'da her şey değiştirilemez

Etiketi değiştirmek sorun olmadı. Ama container portunu `80`'den `8080`'e çevirip `apply` deyince:

```
The Pod "karsilama" is invalid: spec: Forbidden: pod updates may not change fields other than `spec.containers[*].image`,`spec.initContainers[*].image`,`spec.activeDeadlineSeconds`,`spec.tolerations` (only additions to existing tolerations),`spec.terminationGracePeriodSeconds` (allow it to be set to 1 if it was previously negative)
```

Çalışan bir pod'un `spec` kısmında sadece birkaç alan değiştirilebilir. En önemlisi **imaj**. Geri kalan her değişiklik için pod silinip yeniden oluşturulmalıdır:

```bash
kubectl delete -f pod.yaml
kubectl apply -f pod.yaml
```

Gerçek işte bu yüzden pod'lar doğrudan oluşturulmaz. Deployment gibi üst nesneler, değişiklik olunca eski pod'ları silip yenilerini kendisi oluşturur (ileride gelecek).

## Sık yapılan hatalar

**`kind` küçük harfle yazılırsa:**

```yaml
kind: pod
```
```
Error from server (BadRequest): error when creating "pod.yaml": pod in version "v1" cannot be handled as a Pod: no kind "pod" is registered for version "v1" ...
```

Tür adları büyük harfle başlar: `Pod`, `Service`, `Deployment`. (`kubectl get pod` yazarken küçük harf sorun değil, bu sadece YAML için geçerli.)

**`apiVersion` yanlış olursa:**

```yaml
apiVersion: apps/v1
kind: Pod
```
```
error: resource mapping not found for name: "karsilama" namespace: "" from "pod.yaml": no matches for kind "Pod" in version "apps/v1"
ensure CRDs are installed first
```

`apps/v1` diye bir grup var ama Pod orada değil, `v1`'de.

**Alan adı yanlış yazılırsa** (`image` yerine `imge`):

```
Error from server (BadRequest): error when creating "pod.yaml": Pod in version "v1" cannot be handled as a Pod: strict decoding error: unknown field "spec.containers[0].imge"
```

## Uygulamadan önce kontrol: dry-run

```bash
kubectl apply -f pod.yaml --dry-run=server
```

Dosyayı api-server'a gönderip bütün kontrollerden geçirir ama hiçbir şey oluşturmaz. Yukarıdaki `imge` hatasını bu komut da yakaladı.

`--dry-run=client` ise dosyayı kümeye hiç göndermez. Denemede `imge` yazım hatasını **yakalayamadı** ve `pod/deneme2 created (dry run)` dedi. Bu yüzden bir dosyayı kontrol etmek için `server`'ı kullan. `client`, [04](../04-kubectl/)'teki gibi YAML taslağı üretmek için kullanışlıdır.

## containerPort sadece bilgi amaçlı

```yaml
ports:
  - containerPort: 80
```

Bu satır portu **açmaz**, sadece "bu container 80'i dinliyor" diye belgeler. Nginx bu satır olmasa da 80'i dinler ve pod'a 80'den erişilebilir. Yine de yazmak iyi bir alışkanlıktır: okuyan herkes uygulamanın hangi portu kullandığını görür ve bazı araçlar (ör. Service'ler port ismiyle eşleşirken) bu bilgiyi kullanır.

## Çalışan nesneyi düzenlemek: edit

```bash
kubectl edit pod karsilama
```

Nesnenin YAML'ı bir metin editöründe açılır. Kaydedip çıkınca değişiklik kümeye uygulanır. Linux'ta varsayılan editör genelde `vi`'dir:

- Yazmaya başlamak için `i`
- Kaydedip çıkmak için `Esc`, sonra `:wq` ve `Enter`
- Kaydetmeden çıkmak için `Esc`, sonra `:q!`

Başka bir editör kullanmak için: `export KUBE_EDITOR=nano`.

> [!NOTE]
> `edit` ile yaptığın değişiklik YAML dosyana yansımaz. Bir sonraki `kubectl apply -f pod.yaml` değişikliği geri alabilir. Kalıcı değişiklikleri dosyada yapıp `apply` etmek daha güvenli.

## Dene

1. [`pod.yaml`](pod.yaml)'ı uygula ve `kubectl get pod karsilama -o yaml` ile Kubernetes'in eklediği alanlara bak (`status`, `uid`, `restartPolicy`...).
2. Dosyada imajı `nginx:1.30-alpine` yerine `alpine:3.24`'e çevir ve `apply` et. Neden kabul edildi? Pod'a ne oldu? (İpucu: `kubectl get pod karsilama` ve `RESTARTS` sütunu)
3. `kubectl run` ile `--dry-run=client -o yaml` kullanarak kendi pod YAML'ını oluştur, dosyaya kaydet ve `apply` et.

---

Önceki: [06 - Pod](../06-pod/) · Sonraki: [08 - Pod'un Yaşam Döngüsü](../08-pod-yasam-dongusu/)
