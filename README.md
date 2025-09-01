#  Pusula Talent Academy 2025 - SQL & DBA Case Study

Bu repo, **Pusula Talent Academy 2025 SQL & DBA Case Study** kapsamında
verilen sorulara hazırlanmış çözümleri içermektedir.\
Amaç, **veritabanı tasarımı, performans analizi, indeks stratejileri ve
T-SQL sorgulama yetkinliklerini** göstermek ve problem çözme
becerilerini ortaya koymaktır.

------------------------------------------------------------------------

## 🚀 İçindekiler

1.  [Soru 1 - Performans ve Ölçeklenebilirlik
    Analizi](#soru-1---performans-ve-ölçeklenebilirlik-analizi)
2.  [Soru 2 - İndeks Stratejisi ve Sorgu
    Optimizasyonu](#soru-2---indeks-stratejisi-ve-sorgu-optimizasyonu)
3.  [Soru 3 - T-SQL Query Challenge (Satış
    Analizi)](#soru-3---t-sql-query-challenge-satış-analizi)
4.  [Teknolojiler](#teknolojiler)
5.  [Sonuç](#sonuç)

------------------------------------------------------------------------

##  Soru 1 - Performans ve Ölçeklenebilirlik Analizi

**Senaryo:**
- `HastaIslemLog` tablosu 5 yıl boyunca günlük \~25.000 satır veri
almış.
- 5 yıl sonunda toplam \~45 milyon satır oluşmuş.
- Kullanıcılar geçmiş kayıtlara erişimde ciddi performans sorunları
yaşamış.

**Çözüm Adımları:**
- Performans kaybının nedenlerini analiz ettim: yetersiz indeksleme,
fragmentasyon, optimize edilmemiş sorgular, arşivleme eksikliği.\
- İyileştirme önerileri:
- **Arşivleme ve Partitioning** (eski verileri ayrı tabloda tutmak)
- **Yeni indeksler** (HastaId, IslemTarihi, IslemKodu)
- **Zaman bazlı görünümler (Views)**
- **Veri boyutu azaltma** (Aciklama kolonunu ayrı tabloya taşımak)
- **Bakım komutları** (index rebuild, update statistics)

------------------------------------------------------------------------

##  Soru 2 - İndeks Stratejisi ve Sorgu Optimizasyonu

**Senaryo:**\
Sık kullanılan sorgu:

``` sql
SELECT * 
FROM HastaKayit 
WHERE LOWER(AdSoyad) LIKE '%ahmet%' 
  AND YEAR(KayitTarihi) = 2024;
```

**Sorunlar:**
- `LOWER()` ve `YEAR()` fonksiyonları → indeks kullanımını engelliyor.
- `LIKE '%ahmet%'` → full table scan'e yol açıyor.
- `SELECT *` → gereksiz veri taşınıyor.

**Çözüm Önerileri:**
1. **Fonksiyonların kaldırılması**: Persisted computed column ve
indeksler ile `AdSoyadNormalized`, `KayitYili` eklemek.
2. **Full-Text Search**: Metin aramaları için full-text index
kullanımı.
3. **Tarih aralığı**: `YEAR()` yerine `BETWEEN` kullanarak indeks dostu
sorgular yazmak.
4. **SELECT \* yerine gerekli alanlar**: Sadece ihtiyaç duyulan
kolonları çağırmak.
5. **Uygulama tarafında iyileştirmeler**: Arama kutusu için otomatik
tamamlama, caching, arama parametrelerini daraltma.

------------------------------------------------------------------------

##  Soru 3 - T-SQL Query Challenge (Satış Analizi)

**Senaryo:**
- Tablo: `Urun`, `Satis`
- Görevler: Yıllık toplam satış ve adet, en çok satan ürün, hiç
satılmayan ürünler.

**Çözümler:**

 Yıllık toplam satış ve adet:

``` sql
SELECT 
  YEAR(S.SatisTarihi) AS Yil,
  U.UrunAdi,
  SUM(S.Adet) AS ToplamAdet,
  SUM(S.Adet * U.Fiyat) AS ToplamSatisTutari
FROM Satis S
JOIN Urun U ON S.UrunID = U.UrunID
GROUP BY YEAR(S.SatisTarihi), U.UrunAdi
ORDER BY Yil, ToplamSatisTutari DESC;
```

 En çok satan ürün (her yıl için):

``` sql
WITH YillikSatis AS (
  SELECT YEAR(S.SatisTarihi) AS Yil, U.UrunAdi,
         SUM(S.Adet * U.Fiyat) AS ToplamSatisTutari
  FROM Satis S
  JOIN Urun U ON S.UrunID = U.UrunID
  GROUP BY YEAR(S.SatisTarihi), U.UrunAdi
)
SELECT Yil, UrunAdi, ToplamSatisTutari
FROM YillikSatis YS
WHERE ToplamSatisTutari = (
  SELECT MAX(ToplamSatisTutari) 
  FROM YillikSatis WHERE Yil = YS.Yil
)
ORDER BY Yil;
```

 Hiç satılmamış ürünler:

``` sql
SELECT U.UrunID, U.UrunAdi, U.Fiyat
FROM Urun U
LEFT JOIN Satis S ON U.UrunID = S.UrunID
WHERE S.SatisID IS NULL;
```

------------------------------------------------------------------------

##  Teknolojiler

-   **Microsoft SQL Server (T-SQL)**
-   İndeks stratejileri ve sorgu optimizasyonu
-   Veritabanı bakım komutları
-   Full-Text Search

------------------------------------------------------------------------

##  Sonuç

Bu case study kapsamında:
- **Performans sorunlarını analiz ettim**
- **Sürdürülebilir çözümler önerdim**
- **Sorgu optimizasyonu ve indeksleme stratejilerini geliştirdim**
- **T-SQL üzerinde analitik rapor sorguları yazdım**

------------------------------------------------------------------------

 Bu README, proje dosyalarının (Markdown raporları) genel özetini
sunar.
