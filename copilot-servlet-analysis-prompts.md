# Copilot Servlet Uygulaması Analiz Promptları (Java + HTML/JSP + JS)

Java servlet uygulamasında akış üç katmana dağılır: Java (servlet), HTML/JSP ve JS. Önce katmanlar ayrı ayrı, sonra birlikte analiz edilir.

## Bağlam İpucu

Chat'te `#file:` ile ilgili tüm dosyaları birlikte verin (servlet, JSP/HTML, JS). Hangi dosyaların ilişkili olduğunu bilmiyorsanız önce `@workspace` ile bulun.

---

## 1) İlgili Dosyaları Bulma

```text
@workspace CardListServlet sınıfıyla ilişkili tüm dosyaları bul:
- Bu servlet'e istek gönderen JSP/HTML sayfaları ve JS dosyaları
- web.xml veya @WebServlet ile tanımlı URL eşlemesi
- Bu servletin çağırdığı service/DAO sınıfları
Her dosya için tek cümleyle rolünü yaz.
```

---

## 2) Java (Servlet) Analizi

```text
#file:CardListServlet.java dosyasını analiz et:
1. Sorumluluğu nedir, hangi URL/HTTP metotlarını (doGet/doPost) karşılıyor?
2. Request parametreleri nelerdir, nasıl okunuyor ve doğrulanıyor?
3. Request/session/application scope'a hangi attribute'lar konuyor?
4. Hangi service/DAO çağrılıyor, response olarak ne dönüyor (forward, redirect, JSON)?
5. Instance değişkeni kullanımı var mı? Servlet tek instance olduğu için thread-safe mi?
6. Exception ve null durumları nasıl yönetiliyor?
Metot adı veya satır numarası vererek açıkla.
```

---

## 3) HTML/JSP Analizi

```text
#file:cardList.jsp dosyasını analiz et:
1. Sayfa servletten hangi verileri alıyor (attribute, EL, scriptlet)?
2. Hangi form/link/butonlar hangi servlet URL'lerine ne parametreyle istek atıyor?
3. Butonların id'leri ve disabled/hidden durumları neye göre belirleniyor?
4. İş mantığı sayfaya sızmış mı (scriptlet içinde koşullar, hesaplamalar)?
5. Kullanıcı girdisi çıktıya escape edilmeden basılıyor mu (XSS riski)?
```

---

## 4) JS Analizi

```text
#file:cardList.js dosyasını analiz et:
1. Hangi DOM elemanlarına hangi event'leri bağlıyor?
2. Hangi servlet URL'lerine AJAX/fetch isteği atıyor, hangi parametrelerle, dönen cevabı nasıl işliyor?
3. DOM'u (buton durumu, liste içeriği) hangi koşullara göre değiştiriyor?
4. Global değişken veya önceki seçimden kalan state var mı?
5. Hata durumları (başarısız istek, boş cevap) yönetiliyor mu?
```

---

## 5) Üç Katmanı Birleştiren Akış Analizi

```text
#file:CardListServlet.java #file:cardList.jsp #file:cardList.js
Kullanıcı bir karta tıkladığında uçtan uca akışı adım adım çıkar:
Tıklama (JS/HTML) -> istek (URL, parametreler) -> servlet -> service/DAO -> response -> ekranın güncellenmesi.
Her adım için dosya ve satır/metot ver.
Katmanlar arasında uyumsuzluk var mı kontrol et (parametre adı farkı, beklenen ile dönen veri formatı farkı, null durumları).
Sequence diagram olarak da ver (Mermaid).
```

---

## 6) Risk Taraması (Tüm Katmanlar)

```text
Bu üç dosyayı birlikte, çalışma zamanında hata çıkarabilecek yerler açısından incele.
Servlet thread-safety, session state, null parametre, XSS, CSRF, çift tıklama/tekrar gönderim,
JS tarafında state'in önceki karttan kalması gibi konulara bak.
Çıktıyı tablo ver: Dosya/Satır | Risk | Neden hata çıkarır | Önerilen düzeltme.
Sadece kodda görünen riskleri yaz.
```

---

## 7) Hassas Veri Kontrolü (Banka Projeleri)

```text
Kart numarası ve diğer hassas veriler (token, PII) hangi katmanda maskeli, hangisinde açık?
Servlet, JSP ve JS dosyalarını birlikte kontrol et. Loglara, URL parametrelerine, HTML'e
ve JS değişkenlerine maskesiz yazılan yerleri satır numarasıyla listele.
```

---



## 7) Hassas Veri Kontrolü (Banka Projeleri)

```text
#file:KartController.java dosyasını analiz et:
1. Sorumluluğu nedir? Hangi endpoint/metotları, hangi URL ve HTTP metotlarıyla karşılıyor?
2. Girdi parametreleri nelerdir, nasıl okunup doğrulanıyor?
3. Hangi service/DAO çağrılıyor, response olarak ne dönüyor (JSON, DTO, view)?
4. Kart durumuna veya case'e göre karar veren iş kuralları neler?
5. Null, exception ve boş liste durumları nasıl yönetiliyor?
6. Singleton sınıfta paylaşılan state (instance değişkeni) var mı?
Metot adı veya satır numarası vererek açıkla.
```

#file:kartDetay.html dosyasını analiz et:
1. Sayfada hangi bölümler, form ve butonlar var? Her butonun id'si ve işi nedir?
2. Butonların varsayılan durumu (disabled/hidden) nedir, HTML'de sabit mi yoksa dinamik mi?
3. Hangi elemanlar hangi JS fonksiyonlarına veya URL'lere bağlı (onclick, form action, data-attribute)?
4. Veri ekrana nasıl basılıyor (template ifadesi, JS ile doldurma)?
5. Kullanıcı verisi escape edilmeden basılıyor mu (XSS riski)?
6. Kart numarası gibi hassas veri maskesiz gösteriliyor mu?

#file:kartDetay.js dosyasını analiz et:
1. Hangi DOM elemanlarına hangi event'ler bağlı?
2. Hangi URL'lere (fetch/AJAX) hangi parametrelerle istek atıyor, cevabı nasıl işliyor?
3. DOM'u (buton aktif/pasif, liste, alanlar) hangi koşullara göre değiştiriyor?
4. Global değişken veya önceki seçimden kalan state var mı?
5. Hata durumları (başarısız istek, boş cevap) yönetiliyor mu?
6. Kart numarası, token gibi hassas veri console.log'a veya URL'e yazılıyor mu?

---

## Verimi Artıran Noktalar

- 5. prompt en değerlisidir, çünkü katmanlar arası uyumsuzluklar tek tek dosya analizinde görünmez.
- Mermaid çıktısı GitHub'daki `.md` dosyalarında doğrudan diyagram olarak görünür.
- Büyük dosyaları metot metot (kodu seçerek) analiz ettirmek daha derin sonuç verir.
