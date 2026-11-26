# 09 - Çok Container'lı Pod

Bir pod'da birden fazla container çalışabilir. Ama bu, "her şeyi tek pod'a koy" demek değil.

## Tek pod mu, ayrı pod'lar mı?

Bir web uygulaması ve veritabanını düşün. İkisini aynı pod'a koymak ilk bakışta pratik görünür, ama:

- Pod, Kubernetes'te **birlikte** ölçeklenen birimdir. Web uygulamasından 5 kopya istersen veritabanından da 5 kopya çıkar.
- Pod'daki container'lar hep **aynı node'da** çalışır. Biri çok kaynak harcarsa diğeri de etkilenir.
- Biri güncellenirken ikisi birden yeniden başlar.

Bu yüzden kural basit: **Her uygulama kendi pod'unda.** Bir pod'a ikinci bir container sadece ana container'a **yardım eden** ve onsuz bir anlamı olmayan işler için eklenir: loglarını okumak, dosyalarını hazırlamak, trafiğini aracılık etmek gibi.

> [!TIP]
> **Benzetme:** Bir aşçı ve yanındaki komi (yardımcı) aynı mutfakta çalışır, malzemeyi aynı tezgahtan alır. Ama iki ayrı restoranı tek mutfağa sığdırmazsın: biri kalabalıklaşınca diğeri de sıkışır.

## Aynı pod'daki container'lar neyi paylaşır?

| Paylaşılan | Ne demek? |
|---|---|
| **Ağ** | Aynı IP adresi, aynı portlar. Birbirlerine `localhost` ile ulaşırlar. Aynı portu iki container dinleyemez |
| **Volume'lar** | Pod'da tanımlanan bir volume'u birden fazla container bağlayabilir ve aynı dosyaları görür |
| **Yaşam döngüsü** | Aynı node'da birlikte başlar, pod silinince birlikte silinir |

Her container'ın **dosya sistemi ise ayrıdır**: bir container'ın `/tmp`'ye yazdığı dosyayı diğeri görmez. Dosya paylaşmak için ortak bir volume gerekir.

## Örnek 1: Sidecar ile log okumak

[`sidecar.yaml`](sidecar.yaml): `yazici` her 2 saniyede bir log dosyasına satır ekliyor, `okuyucu` aynı dosyayı okuyup ekrana basıyor. İkisi aynı `emptyDir` volume'unu farklı yollara bağlamış:

```yaml
volumes:
  - name: kayitlar
    emptyDir: {}             # pod'la birlikte oluşan, pod'la birlikte silinen boş bir klasör
containers:
  - name: yazici
    volumeMounts:
      - name: kayitlar
        mountPath: /veri     # yazici için /veri
  - name: okuyucu
    volumeMounts:
      - name: kayitlar
        mountPath: /kayit    # okuyucu için /kayit, ama aynı klasör
```

```bash
kubectl apply -f sidecar.yaml
kubectl get pod gunluk
```
```
NAME     READY   STATUS    RESTARTS   AGE
gunluk   2/2     Running   0          12s
```

`READY 2/2`: pod'daki iki container'ın ikisi de hazır.

Pod'da birden fazla container varsa `logs` ve `exec` hangisini kastettiğini bilmek ister. Belirtmezsen ilkini seçer ve bunu söyler:

```bash
kubectl logs gunluk
```
```
Defaulted container "yazici" out of: yazici, okuyucu
```

`yazici` ekrana hiçbir şey yazmadığı için başka çıktı yok. Container'ı `-c` ile seçince:

```bash
kubectl logs gunluk -c okuyucu
```
```
18:51:08 siparis alindi
18:51:10 siparis alindi
18:51:12 siparis alindi
```

`yazici`'nın dosyaya yazdığı satırlar `okuyucu`'nun çıktısında görünüyor. Gerçek hayatta sidecar'lar genelde bu satırları merkezi bir log sistemine gönderir.

## Örnek 2: localhost ile konuşmak

[`localhost.yaml`](localhost.yaml): `web` container'ında Nginx, `kontrol` container'ı her 5 saniyede bir `http://localhost:80` adresine istek atıyor.

```bash
kubectl apply -f localhost.yaml
kubectl logs komsular -c kontrol
```
```
web cevap verdi
web cevap verdi
```

`kontrol`'ün içinde Nginx yok, ama `localhost:80`'e gelen istek aynı pod'daki `web`'e ulaştı. İkisi aynı ağı paylaşıyor. Hostname de ortak:

```bash
kubectl exec gunluk -c okuyucu -- hostname     # gunluk
kubectl exec gunluk -c yazici -- hostname      # gunluk
```

## Örnek 3: Init container

**Init container'lar**, ana container'lar başlamadan **önce** çalışan, işini bitirip çıkan container'lardır. Birden fazlaysa sırayla çalışırlar ve her biri başarıyla bitmeden bir sonraki başlamaz. Kullanım örnekleri: ayar dosyası hazırlamak, bir veritabanının hazır olmasını beklemek, gereken dosyaları indirmek.

[`init.yaml`](init.yaml): `hazirla` 5 saniye bekleyip bir HTML dosyası yazıyor. Nginx bu dosyayı sunuyor.

```yaml
initContainers:
  - name: hazirla
    image: alpine:3.24
    command: ["sh", "-c", "echo 'Sayfa hazirlaniyor...'; sleep 5; echo '<h1>Init container hazirladi</h1>' > /is/index.html"]
containers:
  - name: web
    image: nginx:1.30-alpine
```

```bash
kubectl apply -f init.yaml
kubectl get pod hazirlikli -w
```
```
NAME         READY   STATUS     RESTARTS   AGE
hazirlikli   0/1     Init:0/1   0          1s
hazirlikli   0/1     PodInitializing   0          7s
hazirlikli   1/1     Running           0          7s
```

`Init:0/1`: 1 init container'dan 0'ı bitti. Init bitince Nginx başladı. `READY 1/1`: init container'lar bu sayıya dahil değil.

```bash
kubectl exec hazirlikli -- wget -qO- localhost
```
```
Defaulted container "web" out of: web, hazirla (init)
<h1>Init container hazirladi</h1>
```

### Init container başarısız olursa

[`bozuk-init.yaml`](bozuk-init.yaml)'daki init container `exit 1` ile bitiyor:

```
NAME         READY   STATUS       RESTARTS      AGE
bozuk-init   0/1     Init:Error   2 (31s ago)   36s
```

Ana container (`web`) **hiç başlamadı**. Kubernetes init container'ı tekrar tekrar deniyor ve aradaki bekleme süresi uzadıkça STATUS `Init:CrashLoopBackOff` olur. Ana container'ın logunu istemek işe yaramaz:

```
Error from server (BadRequest): container "web" in pod "bozuk-init" is waiting to start: PodInitializing
```

Sorunun kaynağı init container'ın logunda:

```bash
kubectl logs bozuk-init -c veritabani-bekle
```
```
Veritabani bekleniyor...
Baglanilamadi
```

## Örnek 4: Yerleşik sidecar (native sidecar)

Örnek 1'deki sidecar normal bir container'dı: ana container'la aynı anda başlar, hangisinin önce hazır olacağı belli değildir. Kubernetes v1.33'ten beri bunun için resmi bir yol var: **`restartPolicy: Always` verilmiş init container**.

[`yerlesik-sidecar.yaml`](yerlesik-sidecar.yaml):

```yaml
initContainers:
  - name: okuyucu
    image: alpine:3.24
    restartPolicy: Always     # bu satır onu sidecar yapar
    command: ["sh", "-c", "touch /kayit/uygulama.log; tail -f /kayit/uygulama.log"]
containers:
  - name: yazici
    # ...
```

```
NAME               READY   STATUS    RESTARTS   AGE
yerlesik-sidecar   2/2     Running   0          10s
```

Normal bir init container işini bitirip çıkardı. Bu ise:

- **Ana container'dan önce başlar** ve ana container başlamadan önce çalışır durumda olur. Log toplayıcısı, ilk satır yazılmadan hazırdır.
- **Bitmez**, ana container'larla birlikte çalışmaya devam eder (`READY 2/2`).
- Pod kapanırken **ana container'lardan sonra durur**, böylece son log satırları da kaçmaz.
- Çökerse yeniden başlatılır.

## Özet

| | Normal container | Init container | Yerleşik sidecar |
|---|---|---|---|
| YAML'daki yeri | `containers` | `initContainers` | `initContainers` + `restartPolicy: Always` |
| Ne zaman çalışır? | Init'lerden sonra | Ana container'lardan önce, sırayla | Ana container'lardan önce başlar, onlarla birlikte devam eder |
| Bitmesi gerekir mi? | Hayır | Evet, başarıyla | Hayır |
| `READY`'ye dahil mi? | Evet | Hayır | Evet |

## Dene

1. `sidecar.yaml`'ı uygula ve `kubectl exec -it gunluk -c yazici -- sh` ile içine gir. `/veri` ve `/kayit` klasörlerine bak. Hangisi var, hangisi yok? Neden?
2. `init.yaml`'daki `sleep 5`'i `sleep 20` yap, pod'u silip yeniden oluştur ve `kubectl get pod hazirlikli -w` ile izle.
3. `localhost.yaml`'a üçüncü bir container ekle: yine `nginx:1.30-alpine`. Ne oldu? (İpucu: `kubectl logs komsular -c <yeni container adı>`)
4. Hepsini temizle: `kubectl delete -f .`

---

Önceki: [08 - Pod'un Yaşam Döngüsü](../08-pod-yasam-dongusu/) · [Ana sayfa](../README.md)
