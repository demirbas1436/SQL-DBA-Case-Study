# Bu Sorgudan Kaynaklanabilecek Performans Sorunları

(What performance problems might arise from this query?)

## 1. İndeks Kullanılamaz Durumda

-   `LOWER(AdSoyad)` ifadesi, sütun üzerinde tanımlı indeksin
    kullanılmasını engeller.\
-   Fonksiyon kullanımı, SQL Server'ın indeksleri atlamasına neden olur.

## 2. LIKE '%...%' Kalıbı

-   Joker karakterin başta olması (`%ahmet%`) SQL'in indeksleri
    kullanmasını engeller.\
-   Bu durumda tam tablo taraması (**full table scan**) yapılır.

## 3. YEAR(KayitTarihi) Fonksiyonu

-   `YEAR(KayitTarihi)` ifadesi de tarih sütunu üzerindeki indekslerin
    devre dışı kalmasına neden olur.\
-   Fonksiyonlar indeks dostu değildir.

## 4. SELECT \* Kullanımı

-   Tüm sütunların çağrılması, gereksiz veri taşınmasına ve bellek
    kullanımına yol açar.\
-   Özellikle büyük tabloda ciddi performans kaybı yaratır.

------------------------------------------------------------------------

# Sorgu ve/veya Tablo Yapısını Optimize Etme Önerileri

(How would you optimize this query and/or the table structure?)

## Mevcut Sorgunun Sorunları

``` sql
SELECT * 
FROM HastaKayit 
WHERE LOWER(AdSoyad) LIKE '%ahmet%' 
  AND YEAR(KayitTarihi) = 2024;
```

**Sorunlar:**\
- `LOWER(AdSoyad)` ve `YEAR(KayitTarihi)` fonksiyonları → indeksleri
devre dışı bırakır.\
- `LIKE '%ahmet%'` → wildcard başta olduğu için indeks kullanılamaz.\
- `SELECT *` → gereksiz veri taşır, performansı düşürür.

------------------------------------------------------------------------

## Optimizasyon İçin Olması Gereken Sorgular

### 🔹 1. Fonksiyonları kaldırarak indeks dostu hale getirme

``` sql
-- AdSoyad alanını doğrudan filtreleyebilmek için normalize edilmiş bir kolon ekleyelim
ALTER TABLE HastaKayit ADD AdSoyadNormalized AS LOWER(AdSoyad) PERSISTED;

-- Bu alana indeks ekleyelim
CREATE NONCLUSTERED INDEX IX_AdSoyadNormalized ON HastaKayit(AdSoyadNormalized);

-- KayitTarihi için yıl bilgisi ayrı bir kolon olarak eklensin
ALTER TABLE HastaKayit ADD KayitYili AS YEAR(KayitTarihi) PERSISTED;

-- Bu alana da indeks ekleyelim
CREATE NONCLUSTERED INDEX IX_KayitYili ON HastaKayit(KayitYili);

-- Artık sorgu şu şekilde yazılabilir:
SELECT AdSoyad, KayitTarihi, ... -- sadece gerekli alanlar
FROM HastaKayit
WHERE AdSoyadNormalized LIKE '%ahmet%' 
  AND KayitYili = 2024;
```

Bu yapı sayesinde hem LIKE hem YEAR fonksiyonları önceden hesaplanmış
alanlar üzerinden çalışır ve indeks kullanılabilir hale gelir.

------------------------------------------------------------------------

### 🔹 2. Full-Text Search Alternatifi

Eğer `AdSoyad` üzerinde sıkça metin araması yapılıyorsa, **Full-Text
Index** kullanmak çok daha verimli olur.

``` sql
-- Full-text özelliğini aktif et
CREATE FULLTEXT CATALOG HastaKayitCatalog AS DEFAULT;

-- Full-text indeks oluştur
CREATE FULLTEXT INDEX ON HastaKayit(AdSoyad) 
    KEY INDEX PK_HastaKayit ON HastaKayitCatalog;

-- Sorgu artık şöyle olabilir:
SELECT AdSoyad, KayitTarihi, ...
FROM HastaKayit
WHERE CONTAINS(AdSoyad, 'ahmet') 
  AND KayitTarihi BETWEEN '2024-01-01' AND '2024-12-31';
```

------------------------------------------------------------------------

### 🔹 3. Tarih Aralığı Kullanımı

`YEAR(KayitTarihi) = 2024` yerine doğrudan **tarih aralığı** kullanmak
indeks kullanımını kolaylaştırır:

``` sql
SELECT AdSoyad, KayitTarihi, ...
FROM HastaKayit
WHERE AdSoyad LIKE '%ahmet%' 
  AND KayitTarihi >= '2024-01-01' 
  AND KayitTarihi < '2025-01-01';
```

------------------------------------------------------------------------

### 🔹 4. Gereksiz Veri Taşımamak

``` sql
-- SELECT * yerine sadece gerekli alanları seç
SELECT AdSoyad, KayitTarihi, HastaId
FROM HastaKayit
WHERE ...
```

Bu sorgular ve yapısal değişiklikler, hem veritabanı performansını
artırır, hem de uygulama tarafında daha hızlı sonuçlar alınmasını
sağlar.

------------------------------------------------------------------------

# Uygulama Tarafında Yapılabilecek İyileştirmeler

(Are there any improvements that could be made on the application side?)

-   **Arama Kutusu İçin Otomatik Tamamlama**: Kullanıcılar yazarken
    öneriler sunulursa, tam metin arama yerine daha hedefli sorgular
    yapılabilir.\
-   **Ön Bellekleme (Caching)**: Sık kullanılan sorguların sonuçları
    uygulama tarafında belli sürelerle ön belleğe alınabilir.\
-   **Arama Parametrelerinin Kısıtlanması**: Kullanıcılara tarih
    aralığı, isim baş harfi gibi filtreler sunularak sorgular daha dar
    kapsamlı hale getirilebilir.\
-   **Arama Loglarının Analizi**: Kullanıcıların en çok hangi isimleri
    veya tarihleri aradığı analiz edilerek, bu alanlara özel indeksler
    veya optimizasyonlar yapılabilir.
