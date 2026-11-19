# Copilot Business Review Promptları

Kodu iş (business) açısından inceleyip iş kurallarını ve business case'leri çıkarmak için kullanılan Copilot Chat promptları.

---

## 1) İş Kuralları ve Business Case Çıkarma

```text
#file:OrderService.java dosyasını iş (business) açısından incele:

1. İş kuralları: Kodda uygulanan her iş kuralını düz Türkçe cümlelerle listele
   (örn. "Bakiye yetersizse işlem reddedilir"). Her kural için metot adı/satır ver.
2. Business case'ler: Bu kodun karşıladığı tüm senaryoları çıkar:
   - Mutlu yol (başarılı akış)
   - Alternatif akışlar
   - Hata/red senaryoları
   - Sınır değerler (limit, tutar, tarih, boş/null veri)
3. Eksik veya belirsiz kurallar: Kodda karşılanmayan ama iş açısından beklenebilecek
   durumları yaz (örn. aynı işlemin tekrar gönderilmesi, kısmi başarı, iptal/iade,
   eşzamanlı istekler, zaman aşımı sonrası durum).
4. Tutarsızlıklar: Kod ile iş mantığı arasında çelişen veya şüpheli yerleri belirt
   (yanlış koşul, ters mantık, tutulmayan validasyon).
5. Çıktıyı tablo olarak ver:
   Case No | Senaryo | Girdi/Ön koşul | Beklenen sonuç | Kodda karşılanıyor mu? (Evet/Hayır/Kısmen) | Metot/Satır

Sadece bu kodda görünenlere dayan, varsayım yaparsan "VARSAYIM" diye işaretle.
```

---

## 2) Case'leri Teste Çevirme

```text
Yukarıdaki tablodaki "Hayır" ve "Kısmen" olan her case için JUnit 5 + Mockito test yaz.
Test adları senaryoyu anlatsın (örn. bakiyeYetersizse_islemReddedilir).
```

---

## 3) Banka Projeleri İçin Ek Kontrol

```text
Ayrıca idempotency, tutar/para birimi tutarlılığı, limit ve yetki kontrollerini ayrıca değerlendir.
```
