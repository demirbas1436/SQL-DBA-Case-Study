# 1. Performans Düşüşünün Olası Nedenleri

(What could be the reasons for the performance degradation?)

## 1.1. Veri Miktarının Aşırı Artması

-   Günlük: **25.000 satır**
-   Yıllık: **\~9 milyon satır**
-   5 yılda: **\~45 milyon satır**

Bu kadar büyük bir tablo, indeksleme ve sorgu performansını ciddi
şekilde etkiler.

## 1.2. Yetersiz İndeksleme

-   Sadece **Id** alanı birincil anahtar olarak indekslenmiş.
-   Ancak sorgular çoğunlukla **HastaId** ve **IslemTarihi** üzerinden
    çalışıyor.
-   Bu alanlarda indeks bulunmaması, sorguların yavaşlamasına yol açar.

## 1.3. Fragmentasyon ve Disk I/O Sorunları

-   Sürekli **INSERT / UPDATE / DELETE** işlemleri, veri sayfalarının
    parçalanmasına (fragmentation) sebep olur.\
    (Veritabanında fragmentasyon, kayıtların veya verilerin fiziksel olarak diskte/tabloda düzensiz bir şekilde dağılması anlamına gelir. Bu da zamanla performans kaybına yol açar.
   -    Neden Oluşur?
     -   Çok sık INSERT / UPDATE / DELETE yapılması
     -   Tablo ve indekslerin sürekli büyüyüp küçülmesi
     -    Yoğun transaction trafiği
-   Sonuçları:
    -   Sorguların yavaşlaması
    -   Daha fazla disk okuma (I/O) ihtiyacı
    -   Bellek ve CPU kullanımında artış

## 1.4. Sorgu Optimizasyonunun Eksikliği

-   Karmaşık veya optimize edilmemiş sorgular, büyük veri setlerinde
    ciddi performans sorunları doğurur.

## 1.5. Veri Arşivleme Eksikliği

-   Tüm verilerin tek tabloda tutulması, **geçmiş yıllara ait sorguların
    da aynı tablo üzerinde yoğun şekilde çalışmasına** neden olur.
-   Bu da hem performansı hem de bakım maliyetini artırır.

------------------------------------------------------------------------

# 2. Sürdürülebilirlik İçin İyileştirme Önerileri

(What improvements would you suggest for better sustainability?)

## 2.1. Arşivleme ve Bölme (Partitioning)

-   Büyük tablolar için eski kayıtlar arşivlenebilir veya tablo
    bölünebilir.

``` sql
-- 2 yıldan eski kayıtları arşiv tabloya taşı
INSERT INTO HastaIslemLog_Arsiv (HastaId, IslemTarihi, IslemKodu, Aciklama)
SELECT HastaId, IslemTarihi, IslemKodu, Aciklama
FROM HastaIslemLog
WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());

-- Ardından eski kayıtları ana tablodan sil
DELETE FROM HastaIslemLog
WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());
```

## 2.2. İndeksleme (Indexing)

``` sql
-- HastaId, IslemTarihi ve IslemKodu alanlarına indeks ekle
CREATE NONCLUSTERED INDEX IX_HastaId ON HastaIslemLog(HastaId);
CREATE NONCLUSTERED INDEX IX_IslemTarihi ON HastaIslemLog(IslemTarihi);
CREATE NONCLUSTERED INDEX IX_IslemKodu ON HastaIslemLog(IslemKodu);
```

**Açıklama:**\
- `NONCLUSTERED INDEX`, verileri fiziksel olarak sıralamaz, sadece arama
rehberi gibi çalışır.
- Örneğin:

``` sql
SELECT * FROM HastaIslemLog WHERE HastaId = 12345;
```

Bu sorgu, `IX_HastaId` sayesinde çok daha hızlı çalışır.

## 2.3. Veri Boyutunu Azaltma

-   `Aciklama` alanı uzun ve her zaman dolu olmayabilir.
-   Bu alan ayrı bir tabloya taşınarak performans artırılabilir.

``` sql
CREATE TABLE HastaIslemAciklama (
    Id INT PRIMARY KEY,
    Aciklama NVARCHAR(500)
);
```

## 2.4. Zaman Bazlı Görünümler (Views)

-   Kullanıcılar genellikle **son 1 ay / son 1 yıl** verilerini görmek
    ister.

``` sql
CREATE VIEW HastaIslemLog_Son30Gun AS
SELECT *
FROM HastaIslemLog
WHERE IslemTarihi >= DATEADD(DAY, -30, GETDATE());
```

## 2.5. Bakım ve İstatistik Güncelleme

-   SQL Server'da düzenli bakım performansı artırır.

``` sql
-- Tüm indeksleri yeniden oluştur
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';

-- İstatistikleri güncelle
EXEC sp_updatestats;
```

``` sql

CREATE TABLE HastaIslemLog (
    Id INT IDENTITY(1,1) PRIMARY KEY,                  
    HastaId INT NOT NULL,                              
    IslemTarihi DATETIME NOT NULL DEFAULT GETDATE(),   
    IslemKodu NVARCHAR(20) NOT NULL,                  
    Aciklama NVARCHAR(500) NULL                        
);

-- 1. İndeksleri oluşturmak
CREATE NONCLUSTERED INDEX IX_HastaId 
ON HastaIslemLog(HastaId);

CREATE NONCLUSTERED INDEX IX_IslemTarihi 
ON HastaIslemLog(IslemTarihi);

CREATE NONCLUSTERED INDEX IX_IslemKodu 
ON HastaIslemLog(IslemKodu);

-- 2. Arşiv tablosunu oluşturmak
CREATE TABLE HastaIslemLog_Arsiv (
    Id INT PRIMARY KEY,                                
    HastaId INT NOT NULL,
    IslemTarihi DATETIME NOT NULL,
    IslemKodu NVARCHAR(20) NOT NULL,
    Aciklama NVARCHAR(500) NULL
);

-- 3. 2 yıldan eski kayıtları arşivlemek
INSERT INTO HastaIslemLog_Arsiv (Id, HastaId, IslemTarihi, IslemKodu, Aciklama)
SELECT Id, HastaId, IslemTarihi, IslemKodu, Aciklama
FROM HastaIslemLog
WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());

-- Arşivlenen kayıtları ana tablodan silmek
DELETE FROM HastaIslemLog
WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());

-- 4. Son 30 güne ait kayıtlar için görünüm oluşturmak
CREATE VIEW HastaIslemLog_Son30Gun AS
SELECT *
FROM HastaIslemLog
WHERE IslemTarihi >= DATEADD(DAY, -30, GETDATE());

-- 5. Bakım komutları (opsiyonel ama önerilebilir)
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
EXEC sp_updatestats;
```

------------------------------------------------------------------------

# 3. Bu Yapının 5 Yıl Boyunca Kullanılması Doğru Muydu?

(Do you think using the table in this way for 5 years was the correct
approach?)

## Kısa Vadede: **Evet**

-   Başlangıçta bu yapı işlevsel ve yeterli olabilirdi.

## Uzun Vadede: **Hayır**

-   Veri hacmi arttıkça, tek tablo yaklaşımı **sürdürülebilir olmaktan
    çıkar**.\
-   Tek tabloda tüm verilerin tutulması, hem performans hem de bakım
    maliyetini yükseltir.

## Daha Doğru Yaklaşım

-   **Modüler, arşivlenebilir ve indekslenebilir** bir yapı
    kullanılmalıydı.\
-   Böylece sistem büyüyen veri hacmine rağmen daha uzun süre sağlıklı
    çalışabilirdi.
