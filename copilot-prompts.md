# Copilot Kod Analiz Promptları

Spring Boot (Java) projelerinde sınıf analizi ve hata riski tespiti için kullanılan Copilot Chat promptları.

## Kullanım İpucu

Analiz edilecek sınıfı IDE'de açın. Chat'te `#file` veya `@workspace` ile ya da kodu seçerek bağlam verin. Bağlam ne kadar net olursa cevap o kadar isabetli olur.

---

## 1) Sınıf Analizi

```text
#file:OrderService.java dosyasını analiz et:
1. Sınıfın sorumluluğu nedir, Single Responsibility ihlali var mı?
2. Bağımlılıkları (injection edilenler) ve bunların kullanımı doğru mu?
3. Public metotların girdi/çıktı sözleşmesi ve transaction sınırları nedir?
4. Spring Boot açısından uygunsuz kullanım var mı (field injection, @Transactional yanlış yerde, vb.)?
Her maddeyi satır numarası veya metot adı vererek açıkla.
```

---

## 2) Hata Çıkarabilecek Yerleri Bulma

```text
Seçili kodu bir code reviewer gibi incele ve çalışma zamanında hata çıkarabilecek yerleri bul:
- Olası NullPointerException (null dönebilecek çağrılar, Optional kullanımı)
- Yakalanmayan veya yutulan exception'lar
- Transaction/rollback riskleri (checked exception'da rollback olmaması vb.)
- Thread-safety sorunları (singleton bean içinde paylaşılan state)
- Kaynak sızıntıları (stream, connection, HTTP client)
- Dış servis çağrılarında timeout/retry eksikliği
- Sınır durumları (boş liste, negatif değer, null parametre)

Çıktıyı tablo olarak ver: Satır/Metot | Risk | Neden hata çıkarır | Önerilen düzeltme.
Sadece bu kodda gerçekten görünen riskleri yaz, genel tavsiye verme.
```

---

## 3) Test ile Doğrulatma

```text
Yukarıda bulduğun her risk için, hatayı tetikleyecek bir JUnit 5 + Mockito test senaryosu yaz.
```

---

## 4) Banka Projeleri İçin Ek Kontroller

```text
Ayrıca şunları kontrol et:
- Loglarda hassas veri (kart no, token, PII) yazdırılıyor mu?
- Para tutarlarında double/float yerine BigDecimal kullanılmış mı?
```

---

## 5) Token Prompt
```text
# Copilot Rules
- Stack: Java 17, Spring Boot.
- Answer in Turkish, concisely.
- Explain only non-obvious logic.
- Analyze only supplied context.
- Do not search unrelated files.
- Follow existing project conventions.
- Prefer minimal code changes.
- Show only changed methods when code is requested.
- Never invent project APIs or business rules.
- State uncertainty instead of guessing.
- Max 5 bullets unless more detail is requested.?
```

---


## Verimi Artıran Noktalar

- "Sadece bu kodda görünen riskleri yaz" gibi bir kısıt, Copilot'un genel geçer maddeler sıralamasını engeller.
- Büyük sınıfları metot metot (kodu seçerek) analiz ettirmek daha derin sonuç verir.
- `/fix` ve `/tests` slash komutları seçili kod üzerinde hızlı sonuç verir.

#file:KartController.java #file:kartDetay.html #file:kartDetay.js
Kullanıcı bir karta tıkladığında uçtan uca akışı adım adım çıkar:
Tıklama (HTML/JS) -> istek (URL, parametreler) -> Java endpoint -> service/DAO -> response -> ekranın güncellenmesi.
Her adım için dosya ve satır/metot ver.
Katmanlar arası uyumsuzluk var mı kontrol et (parametre adı farkı, beklenen ile dönen veri formatı farkı, null durumları).
Sequence diagram olarak da ver (Mermaid).
