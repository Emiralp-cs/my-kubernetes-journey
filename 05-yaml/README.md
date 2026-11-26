# 05 - YAML

Kubernetes'e ne istediğimizi YAML dosyalarıyla söyleriz. Docker Compose ve GitHub Actions dosyaları da YAML. Kubernetes'teki hataların büyük kısmı YAML'daki küçük yazım hatalarından çıkar, bu yüzden kuralları iyi bilmek gerekiyor.

YAML, veriyi **girinti** ile yapılandıran bir metin biçimidir. JSON'un da üst kümesidir: geçerli her JSON aynı zamanda geçerli bir YAML'dır.

Bu bölümdeki bütün örnekler Python'un YAML okuyucusu (PyYAML) ve `yamllint` ile denendi. Kubernetes'e özel olanlar gerçek bir kümede denendi.

## Temel kurallar

```yaml
# Bu bir yorum
anahtar: deger
calisan:
  ad: Ayse
  departman: Altyapi
```

- Anahtar ve değer **iki nokta ve bir boşlukla** ayrılır: `ad: Ayse`
- Hiyerarşi girintiyle kurulur. Aynı seviyedeki satırlar **aynı hizada** olmalı.
- Büyük-küçük harf önemlidir: `Ad` ile `ad` farklı anahtarlardır.
- `#` ile başlayan kısım yorumdur.

> [!TIP]
> **Benzetme:** YAML, maddeler halinde tutulmuş bir defter gibidir. Bir maddenin hangi başlığın altında olduğunu sadece ne kadar içeriden başladığına bakarak anlarsın. Bu yüzden tek bir boşluk bile anlamı değiştirebilir.

## Girinti: sadece boşluk

Girinti için **sadece boşluk** kullanılır. Tab karakteri hata verir:

```
found character '\t' that cannot start any token
  in "tab.yaml", line 2, column 1
```

Kaç boşluk kullanılacağı serbesttir, 2 de olur 4 de. Önemli olan aynı seviyedeki satırların aynı hizada olması. Kubernetes dünyasında 2 boşluk gelenek haline gelmiş.

> [!TIP]
> VS Code'da sağ alttaki **"Spaces: 2"** yazısından girinti ayarını görebilirsin. Tab tuşu boşluk eklesin diye "Insert Spaces" açık olmalı. Red Hat'in YAML eklentisi girinti hatalarını yazarken gösterir.

## İki noktadan sonraki boşluk

En sık yapılan hata. Boşluk unutulursa YAML bunu anahtar-değer olarak değil, düz metin olarak okur:

```yaml
a: {id:1, ad: Ayse}
```
```
{'a': {'id:1': None, 'ad': 'Ayse'}}
```

`id:1` tek bir anahtar oldu, değeri boş. Satır başında olursa okuyucu hata verir:

```yaml
sayi:11
ad: x
```
```
mapping values are not allowed here
```

## Değer türleri

| Tür | Örnek | Not |
|---|---|---|
| Metin (string) | `ad: Ayse` | Tırnaksız, tek veya çift tırnaklı |
| Sayı | `port: 80`, `pi: 3.14` | |
| Mantıksal (boolean) | `aktif: true` | `true` veya `false` |
| Boş (null) | `deger: null` | `null`, `~` veya hiçbir şey yazmamak |
| Liste | `[80, 443]` | Aşağıda |
| Sözlük (map) | `{a: 1}` | Aşağıda |

### Tırnaksız değerlerin tuzakları

YAML bazı tırnaksız değerleri beklenmedik türlere çevirir:

```yaml
ulke: no
acik: on
surum: 1.10
posta: 012
```
```
{'ulke': False, 'acik': True, 'surum': 1.1, 'posta': 10}
```

- `no`, `yes`, `on`, `off` metin değil **mantıksal değer** olarak okunur. Kubernetes'in YAML okuyucusu buna `y` ve `n`'yi de ekler (Python'unki eklemez, okuyucular arasında küçük farklar var).
- `1.10` ondalık sayı olur ve sondaki sıfır kaybolur: `1.1`.
- `012` sekizlik (octal) sayı olarak okunur: `10`!

**Bu Kubernetes'te de geçerli.** Gerçek kümede denedik:

```yaml
metadata:
  name: y
```
```
json: cannot unmarshal bool into Go struct field ObjectMeta.metadata.name of type string
```

Pod'a `y` adını vermek bile hata verdi: `y`, "evet" anlamında `true` olarak okundu. Bir ConfigMap'e `bakim_modu: no` yazınca da aynı hata alındı. Çözüm: metin olarak kalması gereken değerleri tırnak içine al: `name: "y"`, `bakim_modu: "no"`, `surum: "1.10"`.

### Tek ve çift tırnak

```yaml
a: 'satir\nbir'
b: "satir\nbir"
```
```
{'a': 'satir\\nbir', 'b': 'satir\nbir'}
```

Tek tırnakta `\n` olduğu gibi kalır. Çift tırnakta ise gerçek bir satır sonuna dönüşür.

### Çok satırlı metin

```yaml
korunan: |
  satir1
  satir2
birlesen: >
  satir1
  satir2
```
```
{'korunan': 'satir1\nsatir2\n', 'birlesen': 'satir1 satir2\n'}
```

`|` satır sonlarını korur (betik veya ayar dosyası yazarken), `>` satırları tek satırda birleştirir (uzun bir açıklamayı bölmek için).

## Listeler ve sözlükler

**Liste (sequence):** Her eleman `- ` (tire ve boşluk) ile başlar:

```yaml
portlar:
  - 80
  - 443
portlar_kisa: [80, 443]      # aynı şey, tek satırda
```

**Sözlük (map):** İç içe anahtar-değer çiftleri:

```yaml
calisan:
  ad: Ayse
  departman:
    ad: Altyapi
    kod: "07"                # tırnaklı: metin olarak kalsın
```

**Liste içinde sözlük:** Kubernetes'te en sık görülen yapı. Pod'un container listesi böyledir:

```yaml
containers:
  - name: web                # 1. eleman başlıyor
    image: nginx:1.30-alpine
    ports:
      - containerPort: 80
  - name: yardimci           # 2. eleman başlıyor
    image: alpine:3.24
```

Tire listenin yeni bir elemanını başlatır. `image` satırı `name` ile **aynı hizada** olduğu için aynı elemanın parçasıdır. Bir boşluk kaysa yapı bozulur.

Aynı veri satır içi (flow) biçimde de yazılabilir: `{ad: Ayse, departman: Altyapi}`. İki biçimin sonucu aynıdır, ama Kubernetes dosyalarında girintili (block) biçim kullanılır.

## Bir dosyada birden fazla belge

`---` belgeleri birbirinden ayırır. `kubectl apply -f` böyle bir dosyadaki bütün nesneleri tek seferde oluşturur:

```yaml
apiVersion: v1
kind: Pod
# ...
---
apiVersion: v1
kind: Service
# ...
```

## Tekrarı önlemek: anchor ve alias

```yaml
temel: &ortak
  imaj: nginx
  yeniden: always
web:
  <<: *ortak
  port: 80
```
```
{'temel': {...}, 'web': {'imaj': 'nginx', 'yeniden': 'always', 'port': 80}}
```

`&ortak` bir bloğa isim verir, `*ortak` onu çağırır, `<<:` içeriğini birleştirir. Docker Compose dosyalarında sık görülür, Kubernetes manifest'lerinde pek kullanılmaz.

## Daha az kullanılan türler

```yaml
sayilar: !!set               # küme: tekrarsız, sırasız
  ? bir
  ? iki
veri: !!binary aGVsbG8=      # Base64 kodlu ikili veri
```
```
{'sayilar': {'bir', 'iki'}, 'veri': b'hello'}
```

## Dosyayı kontrol etmek: yamllint

```bash
sudo apt install yamllint      # veya: pip install yamllint
yamllint dosya.yaml
```

Aynı anahtarı iki kez yazmak, YAML okuyucularının **hata vermeden** kabul ettiği bir durum: ikinci değer birinciyi ezer.

```yaml
a: 1
a: 2
```

Python bunu `{'a': 2}` olarak okudu, hiçbir uyarı vermedi. yamllint ise yakaladı:

```
  1:1       warning  missing document start "---"  (document-start)
  2:1       error    duplication of key "a" in mapping  (key-duplicates)
```

Uzun manifest'lerde fark edilmeden kalan bu tür hataları bulmanın en kolay yolu yamllint. Varsayılan kurallar biraz katıdır (her dosyanın `---` ile başlamasını ister). Daha hoşgörülü kurallar için: `yamllint -d relaxed dosya.yaml`.

Kubernetes YAML'ını kümeye uygulamadan önce kontrol etmek için:

```bash
kubectl apply -f dosya.yaml --dry-run=server
```

Bu komut dosyayı api-server'a gönderir ve doğrulatır, ama hiçbir nesne oluşturmaz.

## Alıştırma

[`hatali.yaml`](hatali.yaml) dosyasında 4 hata var. `yamllint` ile tek tek bulup düzelt.

```yaml
uygulama:
  ad: kitapci
  surum: 1.10
  aktif:yes
  portlar: [80,443]
	ortam: test
  sahibi: Ayse
  sahibi: Mert
```

<details>
<summary>Çözüm</summary>

1. `surum: 1.10` sayı olarak okunur ve `1.1` olur. Metin kalması için: `surum: "1.10"`
2. `aktif:yes` satırında iki noktadan sonra boşluk yok. Ayrıca `yes` yerine `true` yazmak daha açık: `aktif: true`
3. `ortam` satırı tab ile girintilenmiş. Boşlukla değiştir.
4. `sahibi` iki kez yazılmış. Biri silinmeli.

```yaml
---
uygulama:
  ad: kitapci
  surum: "1.10"
  aktif: true
  portlar: [80, 443]
  ortam: test
  sahibi: Ayse
```

Bu hali `yamllint` ile hatasız geçer. (`[80,443]`'te virgülden sonraki eksik boşluk YAML için hata değil ama yamllint stil uyarısı verir.)
</details>

---

Önceki: [04 - kubectl](../04-kubectl/) · Sonraki: [06 - Pod](../06-pod/)
