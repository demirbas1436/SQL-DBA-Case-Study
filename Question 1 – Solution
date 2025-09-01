## 1. Performans Düşüşünün Olası Nedenleri (What could be the reasons for the performance degradation?):
# Veri Miktarının Aşırı Artması:
  Günde 25.000 satır × 365 gün × 5 yıl ≈ 45 milyon satır. Bu kadar büyük bir tablo, indeksleme ve sorgu performansını ciddi şekilde etkileyebilir.

# Yetersiz İndeksleme:
  Sadece Id alanı birincil anahtar olarak indekslenmiş. Ancak sorgular genellikle HastaId, IslemTarihi gibi alanlara göre yapılır. Bu alanlarda indeks olmaması sorguları yavaşlatır.

# Fragmentasyon ve Disk I/O: 
  Sürekli veri eklenmesi ve olası silmeler, veri sayfalarında parçalanmaya (fragmentation) yol açar. Bu da disk erişimini yavaşlatır.
  (Veritabanında fragmentasyon, kayıtların veya verilerin fiziksel olarak diskte/tabloda düzensiz bir şekilde dağılması anlamına gelir.
   Bu da zamanla performans kaybına yol açar.
   Neden Oluşur?
    1. Çok sık INSERT / UPDATE / DELETE yapılması
    2. Tablo ve indekslerin sürekli büyüyüp küçülmesi
    3. Yoğun transaction trafiği
   Sonuçları
    1. Sorguların yavaşlaması
    2. Daha fazla disk okuma (I/O)
    3. Bellek ve CPU kullanımında artış)

# Sorgu Optimizasyonunun Eksikliği:
  Karmaşık veya optimize edilmemiş sorgular, büyük veri üzerinde ciddi performans sorunlarına neden olur.

# Veri Arşivleme Eksikliği:
  Tüm verilerin tek tabloda tutulması, geçmişe yönelik sorguların da aynı yoğunlukta çalışmasına neden olur.

## Sürdürülebilirlik İçin İyileştirme Önerileri (What improvements would you suggest for better sustainability?)

# Arşivleme ve Bölme (Partitioning)
  Tablo çok büyüdüğü için geçmiş verileri arşivlemek veya bölmek mantıklı olur.
  1. Arşivleme için örnek sorgu:
  ```sql
    -- 2 yıldan eski kayıtları arşiv tabloya taşı
    INSERT INTO HastaIslemLog_Arsiv (HastaId, IslemTarihi, IslemKodu, Aciklama)
    SELECT HastaId, IslemTarihi, IslemKodu, Aciklama
    FROM HastaIslemLog
    WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE());
    
    -- Ardından eski kayıtları ana tablodan sil
    DELETE FROM HastaIslemLog
    WHERE IslemTarihi < DATEADD(YEAR, -2, GETDATE())```

  2. İndeksleme (Indexing)
  ```sql
    -- HastaId ve IslemTarihi alanlarına indeks ekle
    CREATE NONCLUSTERED INDEX IX_HastaId ON HastaIslemLog(HastaId);
    CREATE NONCLUSTERED INDEX IX_IslemTarihi ON HastaIslemLog(IslemTarihi);```
    (```sql
        CREATE NONCLUSTERED INDEX ```  SQL Server’da indeks oluşturma komutudur.

      NONCLUSTERED INDEX, tablonun verilerini fiziksel olarak sıralamaz, sadece ayrı bir yapı oluşturur.
      Yani, indeks aslında bir arama rehberi gibi çalışır: belirli bir sütuna göre daha hızlı arama/sıralama yapabilirsin.

      IX_HastaId indeksi
      ```sql
         CREATE NONCLUSTERED INDEX IX_HastaId ON HastaIslemLog(HastaId); ```
      HastaIslemLog tablosundaki HastaId sütununa indeks ekler.
      Bu, genelde belirli bir hastaya ait kayıtları filtrelemek için kullanışlıdır:
      ```sql
      SELECT * FROM HastaIslemLog WHERE HastaId = 12345; ```sql
      Bu sorgu, indeks sayesinde çok daha hızlı çalışır.)

  3. Veri Boyutunu Azaltma
    Aciklama alanı çok uzun olabilir. Eğer bu alan her zaman dolu değilse, ayrı bir tabloya taşınabilir.
    ```sql
       -- Yeni açıklama tablosu
      CREATE TABLE HastaIslemAciklama (
          Id INT PRIMARY KEY,
          Aciklama NVARCHAR(500)
      ); ```sql
      
      -- HastaIslemLog tablosundan açıklamayı çıkarılır
      -- ve sadece Id ile ilişkilendir.

  4. Zaman Bazlı Görünümler (Views)
    Kullanıcılar genellikle son 1 ay veya 1 yıl gibi verileri görmek ister. Bu durumda zaman bazlı görünümler oluşturulabilir.
    ```sql
       CREATE VIEW HastaIslemLog_Son1Ay AS
       SELECT *
       FROM HastaIslemLog
       WHERE IslemTarihi >= DATEADD(MONTH, -1, GETDATE()); ```sql

  5. Bakım ve İstatistik Güncelleme
    SQL Server’da indekslerin yeniden oluşturulması ve istatistiklerin güncellenmesi performansı artırır.
    ```sql
       -- Tüm indeksleri yeniden oluşturur
      EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
      
      -- İstatistikleri güncelle
      EXEC sp_updatestats; ```sql


```sql

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

-- 4. Son 30 güne ait kayıtlar için görünüm oluşturmak.
CREATE VIEW HastaIslemLog_Son30Gun AS
SELECT *
FROM HastaIslemLog
WHERE IslemTarihi >= DATEADD(DAY, -30, GETDATE());

-- 7. Bakım komutları (opsiyonel ama önerilebilir.)
EXEC sp_MSforeachtable 'ALTER INDEX ALL ON ? REBUILD';
EXEC sp_updatestats;
```

## 3. Bu Yapının 5 Yıl Boyunca Kullanılması Doğru Muydu? (Do you think using the table in this way for 5 years was the correct approach? Why or why not?)
  # Kısa Vadede Evet, Uzun Vadede Hayır:
    Başlangıçta bu yapı yeterli olabilir. Ancak veri hacmi büyüdükçe, bu yaklaşım sürdürülebilir olmaktan çıkabilir.
  # Tek Tablo Yaklaşımı Risklidir:
    Tüm işlemlerin tek bir tabloda tutulması, hem performans hem de bakım açısından zorluk yaratır. Modüler ve arşivlenebilir yapı tercih edilebilir.
