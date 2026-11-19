# Copilot Buton Aktif/Pasif Yönetimi Promptları

Java servlet uygulamasında kart detay ekranındaki butonların neye göre pasif olduğunu analiz etmek ve belirli bir case'e göre aktif/pasif yapmak için kullanılan Copilot Chat promptları.

Önce mevcut pasiflik mantığı çıkarılır, sonra yeni case kuralı eklenir. Aksi halde zaten pasif olan butonlar yeni kuralla yanlışlıkla aktif edilebilir.

---

## 1) Butonlar Şu An Neye Göre Pasif?

```text
#file:CardDetailServlet.java ve ilgili JSP/HTML/JS dosyalarını incele.
Kart detay ekranındaki her buton için şunları çıkar:
1. Butonun id/name'i ve ekranda ne işe yaradığı
2. Varsayılan durumu (aktif/pasif)
3. Pasif ise hangi koşula göre pasif (kart durumu, kullanıcı yetkisi, HTML'deki sabit
   disabled attribute'u, JS ile sonradan pasif yapılması, servlet'in request'e koyduğu
   bir flag vb.)
4. Bu koşulun kodda nerede belirlendiği (dosya, metot, satır)

Çıktıyı tablo olarak ver:
Buton | Varsayılan durum | Pasif olma koşulu | Koşulun belirlendiği yer

Sadece kodda görünenlere dayan, bulamadığını "BULUNAMADI" diye işaretle.
```

---

## 2) Case'e Göre Aktif/Pasif Mantığını Ekleme

`[CASE TANIMI]` ve buton adlarını projenize göre doldurun.

```text
Kart detay ekranına şu kuralı ekle:
Seçilen kart [CASE TANIMI: örn. durumu "BLOKELİ" ise / kart tipi "KREDİ" ise]
olduğunda [BUTON1, BUTON2] pasif olsun.

Gereksinimler:
1. Mevcut pasiflik mantığını bozma. Zaten pasif olan butonlar, yeni case'in dışında
   da pasif kalsın.
2. Buton durumu her kart seçildiğinde sıfırdan hesaplansın:
   son durum = mevcut kuralla pasif OR yeni case'le pasif.
   Önceki karttan kalan durum yeni karta taşınmasın. Case'e uymayan karta geçilince
   butonlar eski (mevcut kurala göre) haline dönsün.
3. Case kontrolü tek bir metotta toplansın (örn. isActionAllowed(card, action)),
   if'ler JSP/JS içine dağılmasın.
4. Aynı kontrol servlet tarafında işlem yapılırken de uygulansın. Buton pasif olsa
   bile istek doğrudan gönderilirse işlem reddedilsin.
5. Değişen dosyaları ve metotları listele, öncesi/sonrası farkı göster.
```

---

## 3) Doğrulama (Opsiyonel)

```text
Bu değişiklik için şu senaryoları kapsayan test yaz:
- Case'e uyan kart seçilince ilgili butonlar pasif
- Ardından case'e uymayan kart seçilince butonlar eski haline dönüyor
- Zaten pasif olan buton case dışında da pasif kalıyor
- Pasif butonun işlemi servlet'e doğrudan gönderilince reddediliyor
```

---

## Dikkat Edilecek Noktalar

- Butonları JS'te `disabled = true/false` ile açıp kapatıyorsanız, "geri açma" adımında zaten pasif olması gereken butonları da açabilirsiniz. Bu yüzden 2. maddedeki "sıfırdan hesapla" kuralı önemlidir.
- Pasif butonlar yalnızca arayüz önlemidir. Asıl kontrol servlet tarafında olmalıdır.
- Case ne kadar net yazılırsa Copilot o kadar doğru kod üretir.
