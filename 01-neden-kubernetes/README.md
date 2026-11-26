# 01 - Neden Kubernetes?

Docker ile bir container'ı tek komutla çalıştırabiliyoruz. Peki yüzlerce container'ı onlarca sunucuda çalıştırmak, çöken bir container'ı yeniden başlatmak, trafik artınca kopya sayısını çoğaltmak gerekirse? Bu bölüm Kubernetes'in hangi sorunu çözdüğünü anlatıyor.

## Sorun: elle yönetilemeyen container'lar

Bir e-ticaret sitesinin zamanla nasıl büyüdüğünü düşünelim:

1. **Tek sunucu.** Sitenin bütün parçaları (sepet, ödeme, ürün listesi) tek bir sunucuda container olarak çalışıyor. Sunucu güncelleme için kapatılınca site de kapanıyor.
2. **İkinci sunucu.** Kesinti olmasın diye ikinci bir sunucu kiralandı, container'lar ikisine de kopyalandı ve önüne trafiği dağıtan bir yük dengeleyici (load balancer) kondu. Ama kendiliğinden kapanan bir container'ı fark edip yeniden başlatan hala bir insan.
3. **Trafik arttı.** Kampanya dönemlerinde 2 sunucu yetmiyor, 5 oluyor, 10 oluyor. Container'ları hangi sunucuya koyacağına, hangisinin ne kadar boş yeri olduğuna hep elle karar veriliyor.
4. **Zaman kalmadı.** Ekibin bütün zamanı container'ları elle taşımaya ve yeniden başlatmaya gidiyor.

Bu noktada gereken şey, container'ları bizim yerimize yerleştiren, izleyen ve düzelten bir sistem: **container orkestrasyonu.**

> [!TIP]
> **Benzetme:** Bir orkestrada her müzisyen kendi notasını çalar ama kimin ne zaman gireceğine şef karar verir. Bir müzisyen hastalanırsa yerine başkası geçer. Container'lar müzisyense, Kubernetes şeftir.

Bilinen orkestrasyon araçları: **Kubernetes**, **Docker Swarm**, **HashiCorp Nomad** ve bugün pek kullanılmayan **Apache Mesos**. Bunların arasında açık ara en yaygın olanı Kubernetes.

## Kısa tarihçe

| Tarih | Olay |
|---|---|
| 2000'lerin başı | Google, kendi container'larını yönetmek için **Borg** adlı bir iç sistem geliştirdi |
| 2013 | Borg'un deneyimlerinden **Omega** sistemi doğdu |
| Haziran 2014 | Google, bu deneyimlerden yola çıkan **Kubernetes**'i açık kaynak olarak yayınladı |
| Temmuz 2015 | Kubernetes 1.0 çıktı ve proje, aynı tarihte kurulan **CNCF**'e (Cloud Native Computing Foundation) devredildi |

Kubernetes Yunanca "dümenci, kaptan" demek, logosunun gemi dümeni olması bundan. Kısaca **k8s** yazılır: baştaki "k" ile sondaki "s" arasında 8 harf var.

## Kubernetes nedir?

Kubernetes, container'ları birden fazla sunucuya yerleştiren, çalışır durumda tutan ve ölçekleyen açık kaynak bir platformdur. Hiçbir şirkete bağlı değildir, CNCF tarafından yönetilir.

En önemli özelliği **declarative** (bildirimsel) olması: Kubernetes'e ne yapacağını adım adım söylemezsin, **nasıl bir durum istediğini** söylersin. "Bu uygulamadan 3 kopya çalışsın" dersin. Bir kopya çökerse Kubernetes istenen durum ile mevcut durumun farklı olduğunu fark eder ve yenisini başlatır. Bu konu [02 - Mimari](../02-mimari/) bölümünde ayrıntılı.

## Sürüm numaraları

Kubernetes sürümleri `1.37.3` gibi üç parçalıdır:

| Parça | Örnek | Anlamı |
|---|---|---|
| Major | `1` | Büyük değişiklik. 2015'ten beri hep 1 |
| Minor | `37` | Yeni özellikler. Yılda yaklaşık 3 kez, 4 ayda bir çıkar |
| Patch | `3` | Hata ve güvenlik düzeltmeleri. Genelde ayda bir |

- Aynı anda **en son 3 minor sürüm** desteklenir.
- Her minor sürüm yaklaşık **14 ay** destek alır (12 ay + 2 ay geçiş).

Pratikteki anlamı: bir kümenin desteksiz kalmaması için yılda en az bir kez güncellenmesi gerekir. Bu repodaki örnekler Kubernetes **v1.37** ile denendi.

---

Sonraki: [02 - Mimari](../02-mimari/)
