- Java 17, Spring Boot 3.x. Bu repo bank-app, visa-sample yalnızca referanstır, onu DEĞİŞTİRME.
- visa-sample kodunu kopyalama, mimarisini referans alıp uyarlayarak yeniden yaz.
- Parola, anahtar, kullanıcı adı, PAN, IBAN, telefon koda/teste/loga/yoruma ASLA yazma.
- Hostname doğrulamasını kapatma, istek/yanıt gövdesini loglama, e.printStackTrace yok.
- Config tek bootstrap.yml (local/test/prod), .env yok, sırlar ortam değişkeninden.
  bank-app'in mevcut config standardı farklıysa DUR ve bana sor.
- OpenSSL, BouncyCastle ekleme. JWE için nimbus-jose-jwt.
- Gerçek Visa/DataPower'a ağ çağrısı yapan test yazma, WireMock kullan.
- Her adımdan sonra mvn -q compile çalıştır, hataları düzelt, değişen dosyaları listele.


------



Hiçbir dosyayı değiştirme, yalnızca analiz et.
1) #folder:bank-app: paket yapısı, config yaklaşımı, HTTP istemcisi (RestTemplate/WebClient/Feign),
   hata yönetimi, loglama/maskeleme, test altyapısı, Java ve Spring Boot sürümü, müşteri/hesap/kart
   ile ilgili alan sınıfları.
2) #folder:visa-sample: Visa'ya HTTP isteği atan tüm yerleri bul. Her biri için sınıf.metot, Visa
   endpoint'i (metot + path), MLE kullanıyor mu, hangi model sınıfları.
3) Çıktı: "Visa çağrısı -> bank-app'te hangi iş akışından tetiklenmeli -> hedef sınıf/paket" tablosu.
   Emin olmadıklarını "belirsiz" diye işaretle ve bana sor.

-----


Prompt 0'daki bulgulara göre bank-app içinde `visa` paketi oluştur. Yalnızca altyapı:
- Config: VisaProperties (baseUri, username/password, tls keystore/truststore, mle keyId /
  serverPublicCert / client key, http timeout'ları, isteğe bağlı proxy). Eksik ayarda açılışta düş.
- mTLS'li HTTP istemcisi: hostname doğrulaması açık, keystore truststore olarak kullanılmasın,
  JVM geneli SSL ayarı yapma, connect ve read timeout tanımlı.
- MleCrypto: RSA-OAEP-256 + A128GCM, header'da kid=keyId ve iat (ms). İstek gövdesi
  {"encData":"<JWE>"}, "keyId" HTTP başlığı zorunlu. Anahtar PKCS#1 ve PKCS#8 PEM'den okunsun.
- MleApiClient: Basic Auth + MLE, null/boş alanları gönderme, hata gövdesinden errorCode çıkar,
  durum kodunu kendin değerlendir (istisna fırlatan error handler kullanma). Gövdeyi loglama.
#folder:visa-sample sadece mimari referans. Kopyalama, yeniden yaz.



VisaAliasClient yaz. Endpoint listesi için #folder:visa-sample/src/main/java/.../service/v2'ye
bak (yalnızca Visa'yı çağıran kısımlar). Her metot: createAlias, getAlias, updateAlias,
deleteAlias, updateAliasStatus, createPaymentCredential, getPaymentCredential,
updatePaymentCredential, deletePaymentCredential, aliasInquiry, aliasResolve,
findAliasIdByValue. DTO'ları ve zorunlu alanları #file:docs/visa-api-reference.md'ye göre yaz
(Visa'nın zorunlu işaretledikleri doğrulamada zorunlu olsun, ekstra kural koyma).



Prompt 0'daki tabloya göre bank-app'in mevcut iş akışlarında Visa çağrısını bir facade
(ör. AliasEnrollmentService) üzerinden bağla. Banka müşteri/hesap/kart alanlarını Visa DTO'larına
eşleyen mapper yaz. externalId ile idempotency sağla. Mevcut akışların davranışını bozma, yeni
çağrıyı ayrı bir adımda ekle ve hata durumunda mevcut akışı nasıl etkileyeceğini açıkla.


MleCrypto için gidiş-dönüş, yanlış anahtar ve boş metin testleri yaz. MleApiClient için WireMock
testleri yaz: keyId ve Authorization başlıkları, gövdenin encData olması, Visa hata yanıtı
(errorCode) ve DataPower tarzı HTML hata yanıtı. Gerçek ağ çağrısı yapma.
