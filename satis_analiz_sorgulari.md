# SQL Sorguları ile Satış Analizi

## 1. Her Yıl ve Her Ürün İçin Toplam Satış Tutarı ve Toplam Adet

(Write a query that returns, per year and per product, the total sales
amount (Fiyat \* Adet) and total quantity)

``` sql
SELECT 
  YEAR(S.SatisTarihi) AS Yil,
  U.UrunAdi,
  SUM(S.Adet) AS ToplamAdet,
  SUM(S.Adet * U.Fiyat) AS ToplamSatisTutari
FROM 
  Satis S
JOIN 
  Urun U ON S.UrunID = U.UrunID
GROUP BY 
  YEAR(S.SatisTarihi), U.UrunAdi
ORDER BY 
  Yil, ToplamSatisTutari DESC;
```

------------------------------------------------------------------------

## 2. Her Yıl İçin En Yüksek Satış Tutarına Sahip Ürün

(For each year, identify the product with the highest sales amount.)

``` sql
WITH YillikSatis AS (
  SELECT 
    YEAR(S.SatisTarihi) AS Yil,
    U.UrunAdi,
    SUM(S.Adet * U.Fiyat) AS ToplamSatisTutari
  FROM 
    Satis S
  JOIN 
    Urun U ON S.UrunID = U.UrunID
  GROUP BY 
    YEAR(S.SatisTarihi), U.UrunAdi
)
SELECT 
  Yil,
  UrunAdi,
  ToplamSatisTutari
FROM 
  YillikSatis YS
WHERE 
  ToplamSatisTutari = (
    SELECT MAX(ToplamSatisTutari)
    FROM YillikSatis
    WHERE Yil = YS.Yil
  )
ORDER BY 
  Yil;
```

------------------------------------------------------------------------

## 3. Hiç Satılmamış Ürünleri Listelemek

(Write a query to list products that were never sold.)

``` sql
SELECT 
  U.UrunID,
  U.UrunAdi,
  U.Fiyat
FROM 
  Urun U
LEFT JOIN 
  Satis S ON U.UrunID = S.UrunID
WHERE 
  S.SatisID IS NULL;
```
