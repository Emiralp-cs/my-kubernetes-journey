# 03 - kubeconfig ve Context

`kubectl` hangi kümeye, hangi kullanıcıyla bağlanacağını bir ayar dosyasından okur: **kubeconfig**. minikube, kind, Docker Desktop veya bir bulut servisi bu dosyayı senin için oluşturur ya da günceller.

## Üç parça: cluster, user, context

| Parça | Ne söyler? |
|---|---|
| **cluster** | **Nereye:** api-server'ın adresi ve sertifikası |
| **user** | **Kim olarak:** kullanıcı sertifikası veya token |
| **context** | **İkisinin birleşimi:** bir cluster + bir user (+ isteğe bağlı bir namespace) |

`current-context`, o an kullanılan context'tir. `kubectl`'e verdiğin her komut oraya gider.

> [!TIP]
> **Benzetme:** Telefonundaki rehber gibi. Cluster aranacak numara, user hangi SIM kartla arayacağın, context ise ikisini birleştiren hızlı arama tuşu. current-context o an basılı tuttuğun tuş.

## Dosyanın yapısı

Denemelerin yapıldığı kümenin kubeconfig'i (kısaltılmış, anahtarlar gizli):

```yaml
apiVersion: v1
kind: Config
clusters:                       # nereye
- name: default
  cluster:
    server: https://127.0.0.1:6443
    certificate-authority-data: DATA+OMITTED
users:                          # kim olarak
- name: default
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
contexts:                       # hangi cluster + hangi user
- name: yerel
  context:
    cluster: default
    user: default
    namespace: default
current-context: yerel          # şu an kullanılan
```

## Dosya nerede?

`kubectl` varsayılan olarak `~/.kube/config` dosyasına bakar. `~` her sistemde kullanıcının ev dizinini gösterir:

| Sistem | Tam yol |
|---|---|
| Linux | `/home/kullanici/.kube/config` |
| macOS | `/Users/kullanici/.kube/config` |
| Windows | `C:\Users\kullanici\.kube\config` |

Farklı bir dosya kullanmak için `KUBECONFIG` ortam değişkeni:

```bash
export KUBECONFIG=~/.kube/config-test
```

`KUBECONFIG` yanlış veya boş bir dosyayı gösterirse kubectl hiçbir küme bilmez ve varsayılan adresi dener:

```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Bu hatayı görürsen ilk bakılacak yer `echo $KUBECONFIG` ve `kubectl config current-context`.

## kubectl config komutları

Dosyayı elle düzenlemek mümkün ama girintide yapılacak küçük bir hata dosyayı bozar. Bunun yerine `kubectl config` alt komutları daha güvenli:

```bash
kubectl config get-contexts
```
```
CURRENT   NAME     CLUSTER   AUTHINFO   NAMESPACE
          sistem   default   default    kube-system
*         yerel    default   default    default
```

`*` işareti şu an kullanılan context'i gösterir.

```bash
kubectl config current-context         # yerel
kubectl config use-context sistem      # Switched to context "sistem".
```

Olmayan bir context'e geçmeye çalışınca:

```
error: no context exists with the name: "yok"
```

### Varsayılan namespace'i değiştirmek

```bash
kubectl config set-context --current --namespace=kube-system
kubectl get pods                         # artık kube-system'deki pod'lar
kubectl config set-context --current --namespace=default
```

Namespace verilmemiş komutlar context'teki namespace'te çalışır. Context'te namespace yoksa `default` kullanılır.

### Diğer kullanışlı komutlar

```bash
kubectl config view --minify             # sadece aktif context'in ayarları
kubectl config rename-context eski yeni  # context'in adını değiştir
kubectl config set-context sistem --cluster=default --user=default --namespace=kube-system
                                         # yeni bir context oluştur
```

> [!CAUTION]
> `kubectl config view` sertifika ve anahtarları `DATA+OMITTED` diye gizler. Ama `--raw` eklenirse hepsi açık metin olarak görünür. Bu çıktının ekran görüntüsünü alıp paylaşma: içindeki anahtarlarla kümene erişilebilir. Aynı sebeple dosyayı sadece senin okuyabileceğin şekilde tut: `chmod 600 ~/.kube/config`.

## Birden fazla küme arasında geçiş

Birden fazla kümen varsa (ör. minikube ve iş yerindeki küme) üç yol var:

**1. Bütün context'ler tek dosyada:** En basiti. `kubectl config use-context <ad>` ile geç.

**2. Her küme ayrı dosyada, terminale göre seç:** Dosyaların üzerine hiçbir şey kopyalanmaz, her terminal farklı bir kümeye bakabilir.

```bash
# ~/.bashrc veya ~/.zshrc'ye ekle
kc() { export KUBECONFIG=~/.kube/config-$1; }

kc minikube      # bu terminal artık ~/.kube/config-minikube kullanır
```

**3. Birden fazla dosyayı birlikte okumak:** `KUBECONFIG`'e `:` ile ayrılmış birden fazla dosya verilebilir. kubectl hepsindeki context'leri birlikte görür.

```bash
export KUBECONFIG=~/.kube/config:~/.kube/config-minikube
kubectl config get-contexts
```

> [!WARNING]
> Bir dosyayı `cp ~/.kube/config-minikube ~/.kube/config` gibi asıl config'in **üzerine kopyalayarak** geçiş yapma. Asıl dosyada yaptığın her değişiklik (yeni context, namespace ayarı) bir sonraki kopyalamada kaybolur.

Context'ler arasında hızlı geçiş için **kubectx** ve **kubens** adlı küçük araçlar da çok kullanılır.

> [!IMPORTANT]
> Silme gibi riskli bir komuttan önce **hangi context'te olduğunu** kontrol et. Yanlış context'teyken verilen bir `kubectl delete` yanlış kümede çalışır. Birden fazla küme yönetenler genelde terminal satırında aktif context'i gösteren bir ayar kullanır.

---

Önceki: [02 - Mimari](../02-mimari/) · Sonraki: [04 - kubectl](../04-kubectl/)
