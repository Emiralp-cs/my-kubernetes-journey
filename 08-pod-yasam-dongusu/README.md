# 08 - Pod'un Yaşam Döngüsü

Bir pod oluşturulduğu andan silinene kadar farklı aşamalardan geçer. `kubectl get pods`'un `STATUS` sütunu bu aşamaları ve bir şey ters gittiğinde sebebini gösterir. Bu klasördeki dört YAML dosyası, en sık karşılaşılan durumları bilerek oluşturur:

```bash
kubectl apply -f .
kubectl get pods
```
```
NAME          READY   STATUS         RESTARTS      AGE
bekleyen      0/1     Pending        0             13s
coken         0/1     Error          1 (10s ago)   13s
tamamlanan    0/1     Completed      0             12s
yanlis-imaj   0/1     ErrImagePull   0             12s
```

## Faz (phase) ve STATUS aynı şey değil

Her pod'un `status.phase` alanında beş değerden biri bulunur:

| Faz | Anlamı |
|---|---|
| **Pending** | Pod kabul edildi ama container'ları henüz çalışmıyor: node bekleniyor ya da imaj indiriliyor |
| **Running** | Pod bir node'a yerleşti ve en az bir container'ı çalışıyor (veya yeniden başlatılıyor) |
| **Succeeded** | Bütün container'lar başarıyla (çıkış kodu 0) bitti ve yeniden başlatılmayacak |
| **Failed** | Bütün container'lar bitti ve en az biri hatayla (çıkış kodu 0 dışında) bitti |
| **Unknown** | Pod'un durumu öğrenilemiyor, genelde node'la bağlantı koptuğu için |

`STATUS` sütunu ise daha ayrıntılıdır: fazı ya da container'ın **neden** o durumda olduğunu gösterir. Yukarıdaki pod'ların gerçek fazları:

```bash
kubectl get pods -o custom-columns=AD:.metadata.name,FAZ:.status.phase
```

| Pod | STATUS | Faz |
|---|---|---|
| `tamamlanan` | `Completed` | Succeeded |
| `coken` | `Error`, `CrashLoopBackOff` | **Running** |
| `yanlis-imaj` | `ErrImagePull`, `ImagePullBackOff` | **Pending** |
| `bekleyen` | `Pending` | Pending |

Sürekli çöken bir pod'un fazının `Running` olması şaşırtıcı gelebilir: Kubernetes onu hala yeniden başlatmaya çalıştığı için pod "bitmiş" sayılmıyor.

> [!TIP]
> **Benzetme:** Faz bir hastanın genel durumu (yatıyor, taburcu oldu), STATUS ise doktorun dosyaya yazdığı not ("ateşi düşmüyor, tekrar deneyeceğiz"). Bir sorunu çözmek için nota bakarsın.

## restartPolicy: container durunca ne olsun?

Pod'un `spec.restartPolicy` alanı bütün container'ları için geçerlidir:

| Değer | Container başarıyla bitince | Container hatayla bitince |
|---|---|---|
| `Always` (varsayılan) | Yeniden başlatır | Yeniden başlatır |
| `OnFailure` | Bırakır | Yeniden başlatır |
| `Never` | Bırakır | Bırakır |

Aynı iki komutla üç politikayı denedik. 20 saniye sonra:

```
NAME              READY   STATUS      RESTARTS      AGE
coken-never       0/1     Error       0             20s
tamam-onfailure   0/1     Completed   0             20s
tamam-always      0/1     Completed   1 (16s ago)   20s
```

- `coken-never`: hatayla bitti, yeniden başlatılmadı. Faz: `Failed`.
- `tamam-onfailure`: başarıyla bitti, yeniden başlatılmadı. Faz: `Succeeded`.
- `tamam-always`: başarıyla bittiği halde yeniden başlatıldı. Bir web sunucusu gibi sürekli çalışması gereken uygulamalar için doğru seçim, bir kere çalışıp biten bir iş için yanlış.

Değerler büyük harfle başlar. `always` yazınca:

```
The Pod "coken2" is invalid: spec.restartPolicy: Unsupported value: "always": supported values: "Always", "OnFailure", "Never"
```

## Örnek 1: İşini bitiren pod (Completed)

[`tamamlanan.yaml`](tamamlanan.yaml): `restartPolicy: Never` ile 3 saniye çalışıp biten bir komut.

```bash
kubectl logs tamamlanan
```
```
Is basladi
Is bitti
```

Container çıkış kodu `0` ile bitti, STATUS `Completed`, faz `Succeeded`. Bu bir hata değil: iş bitti. Yedek alma, rapor üretme gibi tek seferlik işler böyle çalışır (ileride Job konusunda).

## Örnek 2: Sürekli çöken pod (CrashLoopBackOff)

[`coken.yaml`](coken.yaml): 2 saniye sonra `exit 1` ile hatayla biten bir uygulama, `restartPolicy: Always`.

```bash
kubectl get pod coken -w
```
```
NAME    READY   STATUS              RESTARTS   AGE
coken   0/1     ContainerCreating   0          0s
coken   1/1     Running             0          0s
coken   0/1     Error               0          3s
coken   1/1     Running             1 (1s ago)   3s
coken   0/1     Error               1 (4s ago)   6s
coken   0/1     CrashLoopBackOff    1 (13s ago)   18s
coken   1/1     Running             2 (13s ago)   18s
coken   0/1     Error               2 (16s ago)   21s
coken   0/1     CrashLoopBackOff    2 (25s ago)   45s
coken   1/1     Running             3 (25s ago)   45s
```

İlk çöküşte container hemen yeniden başlatıldı. Sonra Kubernetes her seferinde **daha uzun bekledi**: CrashLoopBackOff, "çöküyor, yeniden başlatmadan önce bekliyorum" demektir. Bekleme süresi her seferinde yaklaşık iki katına çıkar (10 sn, 20 sn, 40 sn...) ve en fazla 5 dakika olur. Denemede yeniden başlatmalar arasındaki süreler yaklaşık 13, 27, 46 ve 84 saniye ölçüldü (container'ın kendi çalışma süresi de dahil).

Sorunu bulmak için önce loglara bakılır:

```bash
kubectl logs coken
```
```
Basliyorum...
Ayar dosyasi bulunamadi!
```

Container yeniden başlatıldığı anda yeni bir log başlar. Bir önceki (çöken) container'ın loglarını görmek için:

```bash
kubectl logs coken --previous      # kısaltması: -p
```

Çıkış kodu ve sebebi `describe` çıktısındaki `Last State` kısmında da görünür:

```
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

## Örnek 3: İmaj bulunamıyor (ImagePullBackOff)

[`yanlis-imaj.yaml`](yanlis-imaj.yaml)'da imaj adında bir yazım hatası var: `nginx:1.30-alpne`.

```bash
kubectl describe pod yanlis-imaj
```
```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  12s   default-scheduler  Successfully assigned default/yanlis-imaj to dugum1
  Normal   Pulling    12s   kubelet            spec.containers{web}: Pulling image "nginx:1.30-alpne"
  Warning  Failed     12s   kubelet            spec.containers{web}: Failed to pull image "nginx:1.30-alpne": ...
  Warning  Failed     12s   kubelet            spec.containers{web}: Error: ErrImagePull
  Normal   BackOff    11s   kubelet            spec.containers{web}: Back-off pulling image "nginx:1.30-alpne"
  Warning  Failed     11s   kubelet            spec.containers{web}: Error: ImagePullBackOff
```

Pod node'a yerleşti ama imaj indirilemedi. `ErrImagePull` ilk başarısız denemedir, `ImagePullBackOff` ise "tekrar denemeden önce bekliyorum" demektir. Mantık CrashLoopBackOff ile aynı. `Failed to pull image` satırının devamı sebebi söyler (imaj yok, etiket yok, özel bir registry için yetki yok...).

Çözüm: imaj adını düzeltip `kubectl apply` etmek. İmaj, çalışan bir pod'da değiştirilebilen birkaç alandan biridir ([07](../07-pod-yaml/)).

## Örnek 4: Yer bulamayan pod (Pending)

[`bekleyen.yaml`](bekleyen.yaml) 500 GiB bellek istiyor:

```yaml
resources:
  requests:
    memory: "500Gi"
```

```bash
kubectl describe pod bekleyen
```
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  12s   default-scheduler  0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

Bu sefer mesaj kubelet'ten değil **scheduler**'dan geliyor: bu isteği karşılayabilecek bir node yok, pod hiçbir node'a yerleştirilmedi. `kubectl get pods -o wide`'da `NODE` sütunu `<none>` görünür. Pod, uygun bir node çıkana kadar (ör. kümeye yeni bir node eklenene kadar) `Pending`'de bekler.

## Sorun giderme sırası

Bir pod beklendiği gibi çalışmıyorsa:

1. `kubectl get pods`: STATUS ne diyor?
2. `kubectl describe pod <pod>`: **Events** bölümünde ne yazıyor? Mesaj scheduler'dan mı (yerleşme sorunu), kubelet'ten mi (imaj veya container sorunu)?
3. `kubectl logs <pod>` ve `kubectl logs <pod> --previous`: uygulama ne söylemiş?

| STATUS | Muhtemel sebep | Bakılacak yer |
|---|---|---|
| `Pending` | Uygun node yok (kaynak yetersiz) | `describe` → Events |
| `ErrImagePull`, `ImagePullBackOff` | İmaj adı veya etiketi yanlış, registry'ye erişim yok | `describe` → Events |
| `CrashLoopBackOff`, `Error` | Uygulama başlayıp çöküyor | `logs --previous`, `describe` → Last State |
| `Completed` | İş bitti (hata değil) | `logs` |

## Dene

1. `kubectl apply -f .` ile dört pod'u oluştur. Her biri için yukarıdaki sorun giderme sırasını uygula.
2. `yanlis-imaj.yaml`'daki yazım hatasını düzelt ve `apply` et. Pod'un yeniden oluşturulmadan `Running`'e geçtiğini gör.
3. `coken.yaml`'da `restartPolicy`'yi `Never` yap. (İpucu: bu alan çalışan pod'da değiştirilemez, önce pod'u sil.) Fazı ne oldu?
4. Hepsini temizle: `kubectl delete -f .`

---

Önceki: [07 - YAML ile Pod](../07-pod-yaml/) · Sonraki: [09 - Çok Container'lı Pod](../09-cok-containerli-pod/)
