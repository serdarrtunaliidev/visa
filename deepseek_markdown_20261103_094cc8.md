# Visa Alias Directory — Kapsamlı Entegrasyon Rehberi

> **Bu dosya üç amaçla kullanılır:**
> 1. **Referans dokümanı:** Domain bilgisi, endpoint'ler, iş kuralları
> 2. **Adım adım implementasyon rehberi:** Spring Boot banka projesinde sıfırdan entegrasyon
> 3. **AI coding assistant talimatı:** Copilot/Cursor/Claude Code için kurallar
>
> **Versiyon:** 1.0 · **Visa API:** Alias Directory 2.0 · **Hedef:** Java 17 / Spring Boot 3.x
>
> Kod örnekleri iskelet niteliğindedir; bankanın mevcut mimarisi, exception
> hiyerarşisi, mapper yaklaşımı ve sır yönetimi standartlarına göre uyarlayın.

---

## 📋 İÇİNDEKİLER

**BÖLÜM A — Referans Bilgisi**
1. [Domain Bilgisi](#1-domain-bilgisi)
2. [Endpoint Kataloğu](#2-endpoint-kataloğu)
3. [İş Kuralları (Zorunlu)](#3-iş-kuralları-zorunlu)
4. [Mimari ve Orkestrasyon](#4-mimari-ve-orkestrasyon)

**BÖLÜM B — Adım Adım İmplementasyon**
5. [Adım 0: Ön Gereksinimler](#5-adım-0-ön-gereksinimler)
6. [Adım 1: Proje İskeleti ve Bağımlılıklar](#6-adım-1-proje-iskeleti-ve-bağımlılıklar)
7. [Adım 2: Konfigürasyon ve Sır Yönetimi](#7-adım-2-konfigürasyon-ve-sır-yönetimi)
8. [Adım 3: Sertifika ve MLE Anahtarları](#8-adım-3-sertifika-ve-mle-anahtarları)
9. [Adım 4: SSL/RestTemplate Altyapısı](#9-adım-4-sslresttemplate-altyapısı)
10. [Adım 5: MLE Katmanı](#10-adım-5-mle-katmanı)
11. [Adım 6: Visa API Modelleri](#11-adım-6-visa-api-modelleri)
12. [Adım 7: Visa Client Katmanı](#12-adım-7-visa-client-katmanı)
13. [Adım 8: Veritabanı Katmanı](#13-adım-8-veritabanı-katmanı)
14. [Adım 9: Orkestrasyon Servisi](#14-adım-9-orkestrasyon-servisi)
15. [Adım 10: Controller ve DTO'lar](#15-adım-10-controller-ve-dtolar)
16. [Adım 11: Hata Yönetimi](#16-adım-11-hata-yönetimi)
17. [Adım 12: Test Stratejisi](#17-adım-12-test-stratejisi)

**BÖLÜM C — Standartlar ve Operasyon**
18. [Kodlama Standartları](#18-kodlama-standartları)
19. [Güvenlik Kuralları](#19-güvenlik-kuralları)
20. [Yasaklar ve Kontrol Listesi](#20-yasaklar-ve-kontrol-listesi)
21. [Canlıya Geçiş](#21-canlıya-geçiş)
22. [Sorun Giderme](#22-sorun-giderme)
23. [Şirket Standartları (Doldurulacak)](#23-şirket-standartları-doldurulacak)

---

# BÖLÜM A — REFERANS BİLGİSİ

# 1. Domain Bilgisi

## 1.1. Visa Alias Directory Nedir?

Visa Alias Directory, hassas ödeme bilgilerinin (kart numarası, banka hesabı)
yerine **"alias"** adı verilen tanımlayıcıların (telefon numarası, e-posta)
kullanılmasını sağlayan bir dizin servisidir.

**Temel fayda:** Kullanıcı "Ahmet'e 100 TL gönder" der → sistem Ahmet'in
telefon numarasını (alias) kullanır → Visa arka planda Ahmet'in kartını/banka
hesabını çözer → push payment başlatılır. Kullanıcı hiçbir zaman kart numarası
görmez/girmez.

## 1.2. Temel Kavramlar

| Kavram | Tanım | Sorumluluk |
|---|---|---|
| **Alias** | Telefon/e-posta gibi müşteri tanımlayıcı | `aliasValue` + `aliasType` |
| **Profile** | Alias sahibinin adı/adresi | `firstName`, `lastName`, `contactInfo[]` |
| **Consent** | Rıza bilgisi | `presenter`, `version`, `validFromDateTime`, `expiryDateTime` |
| **Payment Credential** | Kart veya banka hesabı | `CARD` veya `BANK` tipinde |
| **External ID** | Müşteri tarafının verdiği ID | İdempotency anahtarı (`requestId`) |
| **Participant** | Alias Directory'ye katılan kurum | Bizim kurumumuz |
| **Program** | Participant grubunun davranışını belirleyen konfigürasyon | Bölgeye özel kurallar |

**Kritik:** Aynı alias'ın (ör. John'un telefonu) **birden fazla profili** olabilir
— her banka için bir profil. Biz tek participant senaryosunda olduğumuz için
`alias_value` bizim için unique, ama ileride çoklu participant'a geçilirse
`(alias_value, participant_id)` unique olmalı.

## 1.3. Alias Directory 1.0 vs 2.0

| Özellik | 1.0 | 2.0 (Bizim kullandığımız) |
|---|---|---|
| Ödeme aracı tipleri | Sadece kart | **Kart + Banka hesabı** |
| Veri yerelleştirme | Yok | **Deployment flexibility** |
| Raporlama | Temel | **Gelişmiş Reports API'leri** |
| Toplu işlem | Yok | **Batch Processing API'leri** |
| Remote directory | Yok | **Directory Network Manager** |

> 1.0 kullanmayın. 2.0 endpoint'lerini kullanın.

## 1.4. Visa Ortamları

| Ortam | Base URL | Kullanım |
|---|---|---|
| **Sandbox** | `https://sandbox.api.visa.com` | Geliştirme, test |
| **Certification** | `https://cert.api.visa.com` | Canlı öncesi sertifikasyon |
| **Production** | `https://api.visa.com` | Canlı |

Base URL **asla koda gömülmez** → `application.yml` + ortam değişkeni.

## 1.5. Alias Value Formatları

- **PHONE**: ITU-T E.164, **`+` işareti OLMADAN**. Örnekler:
  - ABD: `1650xxxxxxx`
  - Türkiye: `905xxxxxxxxx`
  - İngiltere: `4478xxxxxxxx`
  - **Mobil ekip `+` gönderirse `normalizeAliasValue()` ile temizle.**
- **EMAIL**: Standart e-posta formatı.
- **PAYNAME**: `+personalidentifier.domain` formatı (Visa+ için, gelecekte).

## 1.6. Yaşam Döngüsü ve Durumlar

```
PENDING ─── ALIAS_CREATED ─── PC_ADDED
   │
   └─── FAILED (retry ile tekrar denenebilir)
```

| Visa Alias Status | Anlam |
|---|---|
| `ACTIVE` | Lifecycle + Resolution çalışır |
| `DISABLED` | Resolution çalışmaz; tekrar ACTIVE yapılabilir |
| `BLOCKED` | Fraud/şüpheli davranış — otomatik atanır |
| `EXPIRED` | Süre aşımı — tekrar ACTIVE yapılabilir |
| `DELETED` | **Geri alınamaz** — aynı alias yeniden oluşturulmalı |

**Kritik cascade kuralları:**
- Alias `DISABLED` olursa → tüm bağlı credential'lar da `DISABLED` olur
- Alias silinirse → bağlı tüm credential'lar silinir
- Alias `ACTIVE` yapılırsa → credential'lar da `ACTIVE` olur

## 1.7. Ödeme Aracı Zorunlu Alanları

**CARD (Create Alias / Create Payment Credential):**
- `accountNumber` — kart numarası (PAN veya token)
- `accountNumberType` — `PAN` / `TOKEN` / `NIUBIZ` (varsayılan `PAN`)
- `cardType` — ör. "Visa Platinum"
- `expirationDate` — `YYYY-MM` (ör. `2030-01`)
- `issuerName` — kartı veren banka
- `nameOnCard` — kart üzerindeki isim
- `billingAddress.country` — **zorunlu** (ISO-3166 Alpha-3)
- `billingAddress` diğer alanları opsiyonel

**BANK:**
- `accountNumber`, `accountName`
- `accountNumberType` — `IBAN` / `DEFAULT` / `TOKEN`
- `bankName`, `countryCode`, `currencyCode`
- `address.country` — **zorunlu**
- `BIC`, `bankCode`, `bankCodeType`, `branchCode`, `accountType` — koşullu

## 1.8. Hata Kodları

| Kod | Anlam | Retry? |
|---|---|---|
| **400** | Validasyon hatası / eksik alan | ❌ |
| **404** | Kaynak bulunamadı | ❌ |
| **422** | İş kuralı ihlali | ❌ |
| **429** | Rate limit | ⚠️ `Retry-After` |
| **500 / 503** | Sunucu hatası | ⚠️ Backoff + jitter |
| **VAD-1000** | Data mismatch | ❌ |
| **VAD-1040** | Original alias bulunamadı (race condition olabilir) | ⚠️ Recovery |
| **VAD-1120/1130** | Internal error | ⚠️ Retry |
| **VAD-1260** | Default hesap silindi | ❌ |
| **CVC-30010** | Kart zaten ekli | ❌ |

---

# 2. Endpoint Kataloğu

## 2.1. Kullanacağımız Endpoint'ler (Alias Directory 2.0)

| Operasyon | Metot | Path | Yanıt |
|---|---|---|---|
| **Create Alias** | POST | `/aliasdirectory/v1/aliases` | 200 + `aliasId`, `paymentCredentials[]` |
| **Get Alias** | GET | `/aliasdirectory/v1/aliases/{aliasId}` | 200 + alias detayı |
| **Update Alias** | PUT | `/aliasdirectory/v1/aliases/{aliasId}` | **204** (tam replace) |
| **Delete Alias** | DELETE | `/aliasdirectory/v1/aliases/{aliasId}` | **204** (cascade) |
| **Update Alias Status** | PUT | `/aliasdirectory/v1/aliases/{aliasId}/status` | **204** |
| **Find Alias ID by Value** | POST | `/aliasdirectory/v1/aliases/aliasId` | 200 + `id` (404 = yok) |
| **Get by External ID** | POST | `/aliasdirectory/v1/aliases/getByExternalId` | 200 + `id` |
| **Get All Payment Credentials** | GET | `/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials` | 200 + array |
| **Create Payment Credential** | POST | `/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials` | 200 + `id` |
| **Get Payment Credential** | GET | `/aliasdirectory/v1/paymentCredentials/{id}` | 200 |
| **Update Payment Credential** | PUT | `/aliasdirectory/v1/paymentCredentials/{id}` | **204** (tam replace) |
| **Delete Payment Credential** | DELETE | `/aliasdirectory/v1/paymentCredentials/{id}` | **204** |
| **Alias Resolve** | POST | `/aliasdirectory/v1/aliases/resolve` | 200 + profil + credential'lar |
| **Alias Inquiry** | POST | `/aliasdirectory/v1/aliases/inquiry` | 200 + var/yok özeti |

**KULLANMADIKLARIMIZ:** Alias Metadata API'leri, Batch Processing, Reports,
Webhooks. Bunlar ayrı kullanıcı segmenti içindir.

## 2.2. Path Parametreleri — Dikkat!

| Parametre | Nereden Alınır | Kullanıldığı Yerler |
|---|---|---|
| **`aliasId`** | Create Alias yanıtı `id` / `findAliasIdByValue` / `getByExternalId` | Get/Update/Delete Alias, Create/GetAll Payment Credentials |
| **`paymentCredentialId`** | Create Payment Credential yanıtı `id` / `getAllPaymentCredentials` | Get/Update/Delete Payment Credential |

> `aliasId` ile `paymentCredentialId` **karıştırılmamalı**.

## 2.3. Sık Kullanılan Header'lar

| Header | Nerede | Açıklama |
|---|---|---|
| `X-Correlation-Id` | Tüm isteklerde | İzleme için UUID üret |
| `keyId` | MLE istekleri | MLE anahtar ID'si |
| `On-Behalf-Of` | TSP senaryosunda | Participant ID |

## 2.4. cURL Örneği

```bash
curl -X POST "https://sandbox.api.visa.com/aliasdirectory/v1/aliases" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -H "X-Correlation-Id: $(uuidgen)" \
  --cert visa_cert.pem --key visa_key.pem \
  -d '{
    "aliasType": "PHONE",
    "aliasValue": "905551234567",
    "consent": { "presenter": "Bank A", "version": "1.0",
                 "validFromDateTime": "2026-01-01T00:00:00.000Z" },
    "profile": { "firstName": "John", "lastName": "D" },
    "paymentCredentials": [{
      "type": "CARD",
      "accountNumber": "4111111145551142",
      "cardType": "Visa Platinum",
      "issuerName": "Bank A",
      "nameOnCard": "John D",
      "expirationDate": "2030-01",
      "billingAddress": { "country": "USA" }
    }]
  }'
```

---

# 3. İş Kuralları (Zorunlu)

## 3.1. Create Alias — "Her Tipten En Fazla 1"

Create Alias isteğinde **`paymentCredentials` dizisine her tipten en fazla bir
tane** eklenebilir:
- ✅ 1 kart + 1 banka hesabı
- ❌ 2 kart + 1 banka hesabı
- ❌ 2 banka hesabı

İkinci ve sonraki aynı tip araçlar için **`Create Payment Credential`** endpoint'i
kullanılır.

## 3.2. CEMEA/EU İsim Kuralı

`nameOnCard` ve `accountName` alanları için **soyadının sadece ilk harfi + nokta**:
- "John Smith" → `"John S."`
- "Ayşe Yılmaz" → `"Ayşe Y."`

> Bu kural CEMEA ve EU pazarlarında zorunludur. İhlal → 422.

## 3.3. PUT = Tam Replace

`PUT /paymentCredentials/{id}` ve `PUT /aliases/{id}` **tam replace** yapar.
Gönderilmeyen alanlar **silinir**.

**Doğru pattern:**
```
1. GET /paymentCredentials/{id} → tüm alanları al
2. Yerelde değiştir
3. PUT /paymentCredentials/{id} → tüm alanları geri gönder
```

## 3.4. Cascade Silme

- `DELETE /aliases/{aliasId}` → tüm credential'lar + additional alias'lar silinir
- `DELETE /additionalAliases/{id}` → sadece o alias silinir
- `DELETE /paymentCredentials/{id}` → sadece o credential silinir

## 3.5. Status Cascade

- Alias `DISABLED` → tüm credential'lar `DISABLED`
- Alias `ACTIVE` → tüm credential'lar `ACTIVE`
- DB'de bu türetmeyi otomatik yap

## 3.6. Idempotency — `externalId` Kullanımı

- Mobil ekip her istek için **UUID `requestId`** üretir
- Bu ID:
  - DB'de `request_id` (unique)
  - Visa'da `externalId` olarak gönderilir
- Aynı `requestId` tekrar gelirse → mevcut sonucu döndür (`ALREADY_PROCESSED`)

## 3.7. Race Condition — Recovery

İki paralel istek aynı alias'ı oluşturmaya çalışırsa:
1. İlk istek `createAlias` ile başarılı olur
2. İkinci istek `VAD-1000` veya `VAD-1040` alır
3. İkinci istek bu hatayı yakalar → `findAliasIdByValue` ile ID'yi bulur →
   `createPaymentCredential`'a geçer

```
1. Lookup (findAliasIdByValue)
   ├── 200 → aliasId al → Create Payment Credential
   └── 404 → Create Alias'ı dene
              ├── 200 → başarılı
              └── VAD-1040/VAD-1000 → Lookup'ı tekrar → Create Payment Credential
```

## 3.8. Kullanıcı Adı PII Kuralı

`userDetails.userName` alanı **PII içermez**. Anonim ID kullanın
(ör. `usr_12345`). Bu, KVKK/GDPR uyumu için de gereklidir.

---

# 4. Mimari ve Orkestrasyon

## 4.1. Ana Akış

```
[MOBİL İSTEK]
     │
     ▼
[CONTROLLER] → doğrulama → [ORCHESTRATION SERVICE]
                              │
                              ├─ 1. DB'de requestId var mı? ─── EVET → idempotent yanıt
                              │
                              ├─ 2. PENDING kaydet
                              │
                              ├─ 3. DB'de visa_alias_id var mı?
                              │       ├─ VAR → createPaymentCredential(aliasId, card)
                              │       └─ YOK → createAlias(...)
                              │
                              ├─ 4. Başarılı → ALIAS_CREATED / PC_ADDED
                              └─ 5. Başarısız → FAILED (retry için sakla)
```

## 4.2. Katmanlar

```
controller/  → HTTP, doğrulama, status eşleme
service/     → İş mantığı, transaction, orkestrasyon
client/      → Visa HTTP çağrıları (VisaAliasClient)
repository/  → DB erişimi (Spring Data JPA)
domain/      → Entity'ler
dto/         → Request/Response modelleri (API sözleşmesi)
```

**Kurallar:**
- Controller **asla** `VisaAliasClient`'ı doğrudan çağırmaz → Service üzerinden
- Client **asla** DB'ye yazmaz → sadece HTTP
- Entity **asla** Controller'a dönmez → DTO kullan
- Controller'da `@Transactional` **olmaz** → Service'te olur
- `Visa` API modeli (request/response) ile `domain` modeli **ayrı** olmalı

## 4.3. Model Ayrımı (Kritik!)

```java
// 1) Controller katmanı (mobil ekibin gördüğü)
record MobileAliasRequest(...) {}
record MobileAliasResponse(...) {}

// 2) Visa API katmanı (Visa'nın beklediği)
class CreateAliasRequest { ... }
class CreateAliasResponse { ... }

// 3) Entity katmanı (DB)
@Entity class VisaAliasRecord { ... }
```

---

# BÖLÜM B — ADIM ADIM İMPLEMENTASYON

# 5. Adım 0: Ön Gereksinimler

Başlamadan önce şunların hazır olduğundan emin olun:

### 5.1. Visa Developer Erişimi
- [ ] [Visa Developer](https://developer.visa.com) hesabı
- [ ] **Alias Directory Service** ürünü eklenmiş proje
- [ ] Erişim talebi onaylanmış ("Request Access" → Approved)
- [ ] Sandbox kimlik bilgileri indirilmiş

### 5.2. İndirilmesi Gerekenler

| Dosya | Nereden | Nerede Kullanılacak |
|---|---|---|
| `visa_keystore.jks` (veya `.p12`) | Credentials → Two-Way SSL | mTLS kimlik doğrulama |
| `VisaRootCA.pem`, `VisaIntermediateCA.pem` | Credentials → Two-Way SSL | CA trust store |
| `visa_server_cert.pem` | Credentials → MLE | Server public cert (şifreleme) |
| MLE private key `.pem` | Credentials → MLE | Client private key (şifre çözme) |
| `keyId` | Credentials → MLE | Header'da gönderilecek |
| `username` / `password` | Credentials → Two-Way SSL | Basic Auth |

> ⚠️ Bu dosyalar **asla git'e commit edilmez**. `.gitignore`'a ekleyin.

### 5.3. Yazılım Ortamı
- [ ] Java 17+
- [ ] Spring Boot 3.x
- [ ] Maven veya Gradle
- [ ] PostgreSQL / Oracle / MySQL (bankanın kullandığı DB)
- [ ] Postman / Insomnia (test için)
- [ ] IntelliJ IDEA veya Cursor (kod için)

### 5.4. Ekip İletişimi
- [ ] Mobil ekipten **istek sözleşmesi** (hangi alanlar gelecek)
- [ ] Güvenlik ekibinden **sır yönetimi** yaklaşımı (Vault/AWS/K8s Secrets)
- [ ] DevOps ekibinden **sertifika mounting** yöntemi
- [ ] DBA'dan **şema oluşturma** izni

---

# 6. Adım 1: Proje İskeleti ve Bağımlılıklar

## 6.1. Paket Yapısı

```
com.banka.visa.alias/
├── config/                    # Yapılandırma sınıfları
│   ├── VisaProperties.java
│   └── RestTemplateConfig.java
├── client/                    # Visa HTTP istemcisi
│   ├── VisaApiClient.java      # Base
│   ├── MleApiClient.java       # MLE'li
│   └── VisaAliasClient.java    # Domain-specific
├── crypto/
│   └── MleEncryptionUtil.java
├── dto/                       # Visa API modelleri
│   ├── request/
│   │   ├── CreateAliasRequest.java
│   │   ├── Consent.java
│   │   ├── Profile.java
│   │   ├── PaymentCredential.java
│   │   └── CardCredential.java
│   └── response/
│       ├── CreateAliasResponse.java
│       └── AliasIdResponse.java
├── domain/
│   └── VisaAliasRecord.java
├── repository/
│   └── VisaAliasRecordRepository.java
├── service/
│   ├── MobileAliasOrchestrationService.java
│   └── mapper/AliasRequestMapper.java
├── web/
│   ├── MobileAliasController.java
│   └── dto/
│       ├── MobileAliasRequest.java
│       ├── MobileAliasResponse.java
│       └── MobileAliasStatus.java
├── exception/
│   ├── VisaClientException.java
│   ├── AliasOrchestrationException.java
│   └── GlobalExceptionHandler.java
└── util/
    ├── SensitiveDataHasher.java
    └── AliasNormalizer.java
```

> **Kural:** `dto` (Visa'nın beklediği) ile `web.dto` (mobilin gördüğü) **ayrı** olmalı. Entity hiçbir zaman Controller'a dönmez.

## 6.2. `pom.xml` Bağımlılıkları

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.httpcomponents.client5</groupId>
        <artifactId>httpclient5</artifactId>
    </dependency>
    <dependency>
        <groupId>com.nimbusds</groupId>
        <artifactId>nimbus-jose-jwt</artifactId>
        <version>9.37.3</version>
    </dependency>
    <dependency>
        <groupId>org.bouncycastle</groupId>
        <artifactId>bcprov-jdk18on</artifactId>
        <version>1.77</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.wiremock</groupId>
        <artifactId>wiremock-standalone</artifactId>
        <version>3.9.1</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 6.3. `application.yml` İskeleti

```yaml
spring:
  application:
    name: visa-alias-service
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true

visa:
  base-uri: ${VISA_BASE_URI:https://sandbox.api.visa.com/}
  username: ${VISA_USERNAME}
  password: ${VISA_PASSWORD}
  keystore:
    path: ${VISA_KEYSTORE_PATH}
    password: ${VISA_KEYSTORE_PASSWORD}
  mle:
    enabled: true
    key-id: ${VISA_MLE_KEY_ID}
    server-public-cert: ${VISA_MLE_SERVER_CERT_PATH}
    client-private-key: ${VISA_MLE_CLIENT_KEY_PATH}

security:
  hash:
    pepper: ${HASH_PEPPER}

logging:
  level:
    com.banka.visa: INFO
    org.apache.hc.client5: WARN
```

## 6.4. `.env.example` (Git'e commit edilir, gerçek değer içermez)

```bash
VISA_BASE_URI=https://sandbox.api.visa.com/
VISA_USERNAME=
VISA_PASSWORD=
VISA_KEYSTORE_PATH=/etc/visa/keystore.jks
VISA_KEYSTORE_PASSWORD=
VISA_MLE_KEY_ID=
VISA_MLE_SERVER_CERT_PATH=/etc/visa/server-cert.pem
VISA_MLE_CLIENT_KEY_PATH=/etc/visa/client-key.pem
HASH_PEPPER=
DB_URL=jdbc:postgresql://localhost:5432/visa_alias
DB_USERNAME=
DB_PASSWORD=
```

## 6.5. `.gitignore` (Zorunlu)

```gitignore
target/
build/
*.class
*.jar
.idea/
*.iml
.vscode/
.env
.env.*
!.env.example
*.jks
*.p12
*.pfx
*.pem
*.key
secrets/
certs/
*.log
logs/
```

## ✅ Doğrulama
- [ ] `mvn clean compile` hatasız geçiyor
- [ ] `application.yml` gerçek sır içermiyor (sadece `${...}` placeholder)
- [ ] `.env` ve `.env.*` `.gitignore`'da

---

# 7. Adım 2: Konfigürasyon ve Sır Yönetimi

## 7.1. Konfigürasyon Sınıfı

```java
@Configuration
@ConfigurationProperties(prefix = "visa")
@Getter
@Setter
public class VisaProperties {
    private String baseUri;
    private String username;
    private String password;
    private Keystore keystore = new Keystore();
    private Mle mle = new Mle();

    @Getter @Setter
    public static class Keystore {
        private String path;
        private String password;
    }

    @Getter @Setter
    public static class Mle {
        private boolean enabled = true;
        private String keyId;
        private String serverPublicCert;
        private String clientPrivateKey;
    }
}
```

Ana sınıfta: `@ConfigurationPropertiesScan` veya `@EnableConfigurationProperties(VisaProperties.class)`.

## 7.2. Sır Yönetimi Kararı (Şirket Standardı)

Bankanın tercihine göre:

| Yaklaşım | Artı | Eksi |
|---|---|---|
| **K8s Secrets** | Standart, basit | Rotasyon manuel |
| **HashiCorp Vault** | Dinamik sır, audit | Altyapı gerekir |
| **AWS/Azure Secrets Manager** | Managed | Cloud bağımlı |
| **Ortam değişkeni** (basit) | Hızlı | Rotasyon zor |

**Öneri:** Vault veya K8s Secrets. `application.yml`'de sadece `${VISA_USERNAME}` gibi placeholder'lar olmalı.

## 7.3. Sertifika Yönetimi

- Sertifikalar **konteyner imajına gömülmez**
- **Volume mount** veya **init container** ile `/etc/visa/` altına yerleştirilir
- Dosya izinleri `600`, sahibi uygulama kullanıcısı
- Rotasyon: Visa sertifikaları genelde **1 yılda** bir yenilenir; takvim hatırlatıcısı kurun

## ✅ Doğrulama
- [ ] `VisaProperties` doğru bind ediliyor
- [ ] Sır kaynağı (Vault/K8s) test ortamında çalışıyor
- [ ] `git grep "VISA_PASSWORD="` → sadece `.env.example` dönmeli

---

# 8. Adım 3: Sertifika ve MLE Anahtarları

## 8.1. mTLS Keystore Hazırlığı

```bash
# Eğer .p12 indirdiyseniz, JKS'ye çevirin (banka standardı JKS ise)
keytool -importkeystore \
  -srckeystore visa.p12 -srcstoretype PKCS12 \
  -destkeystore visa_keystore.jks -deststoretype JKS \
  -srcstorepass <P12_PASS> -deststorepass <JKS_PASS>

# Visa CA'larını truststore'a ekleyin (bazı ortamlarda gerekir)
keytool -import -alias visa-root -keystore visa_truststore.jks \
  -file VisaRootCA.pem -storepass <PASS>
keytool -import -alias visa-intermediate -keystore visa_truststore.jks \
  -file VisaIntermediateCA.pem -storepass <PASS>
```

## 8.2. MLE Anahtarları

- **Server public cert** (`visa_server_cert.pem`): Visa'nın, bizim isteğimizi şifrelemek için bize verdiği public key
- **Client private key** (`.pem`): Bizim, Visa'nın şifrelediği yanıtı çözmek için kullandığımız private key

**Dikkat:** Client private key **asla** paylaşılmaz, loglanmaz, git'e girmez.

## 8.3. Dosya Yerleşimi (Canlıda)

```
/etc/visa/
├── keystore.jks              (600, app user)
├── keystore.pass             (600, app user) — VEYA Vault'tan
├── server-cert.pem           (644)
├── client-key.pem            (600, app user)
└── key-id.txt                (644) — VEYA Vault'tan
```

## ✅ Doğrulama
- [ ] `keytool -list -keystore visa_keystore.jks` → 1 private key entry görünüyor
- [ ] `openssl x509 -in visa_server_cert.pem -text -noout` → geçerli tarih aralığı
- [ ] `openssl rsa -in client-key.pem -check` → geçerli private key

---

# 9. Adım 4: SSL/RestTemplate Altyapısı

Bu adımda, Visa'ya **iki yönlü SSL** ile bağlanabilen bir `RestTemplate` bean'i oluşturacağız.

## 9.1. `RestTemplateConfig`

```java
@Configuration
@RequiredArgsConstructor
public class RestTemplateConfig {

    private final VisaProperties props;

    @Bean
    public RestTemplate visaRestTemplate() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("JKS");
        try (InputStream is = new FileInputStream(props.getKeystore().getPath())) {
            keyStore.load(is, props.getKeystore().getPassword().toCharArray());
        }

        SSLContext sslContext = SSLContexts.custom()
                .loadKeyMaterial(keyStore, props.getKeystore().getPassword().toCharArray())
                .loadTrustMaterial(keyStore, null)
                .build();

        SSLConnectionSocketFactory sslFactory =
                new SSLConnectionSocketFactory(sslContext, NoopHostnameVerifier.INSTANCE);

        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectTimeout(Timeout.ofSeconds(10))
                .setResponseTimeout(Timeout.ofSeconds(30))
                .build();

        PoolingHttpClientConnectionManager cm =
                PoolingHttpClientConnectionManagerBuilder.create()
                        .setSSLSocketFactory(sslFactory)
                        .setMaxConnTotal(50)
                        .setMaxConnPerRoute(20)
                        .build();

        CloseableHttpClient httpClient = HttpClients.custom()
                .setDefaultRequestConfig(requestConfig)
                .setConnectionManager(cm)
                .build();

        HttpComponentsClientHttpRequestFactory factory =
                new HttpComponentsClientHttpRequestFactory(httpClient);
        return new RestTemplate(factory);
    }
}
```

## 9.2. Base HTTP Client

```java
public class VisaApiClient {

    protected final RestTemplate restTemplate;
    protected final ObjectMapper objectMapper;
    protected final VisaProperties props;

    public VisaApiClient(RestTemplate restTemplate, ObjectMapper objectMapper,
                        VisaProperties props) {
        this.restTemplate = restTemplate;
        this.objectMapper = objectMapper;
        this.props = props;
    }

    protected HttpHeaders buildHeaders() {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.setAccept(List.of(MediaType.APPLICATION_JSON));
        headers.add("X-Correlation-Id", UUID.randomUUID().toString());

        String auth = props.getUsername() + ":" + props.getPassword();
        headers.add(HttpHeaders.AUTHORIZATION, "Basic " +
                Base64.getEncoder().encodeToString(auth.getBytes(StandardCharsets.UTF_8)));
        return headers;
    }
}
```

## ✅ Doğrulama
- [ ] `curl --cert ... --key ...` benzeri bir test çalışıyor
- [ ] Timeout'lar makul (connect 10sn, read 30sn)
- [ ] Connection pool ayarları yapılmış

---

# 10. Adım 5: MLE Katmanı

**Kritik:** Kart numarası gibi hassas veriler MLE olmadan gönderilemez.

## 10.1. MLE Encryption Utility

```java
@Component
@RequiredArgsConstructor
public class MleEncryptionUtil {

    private final VisaProperties props;

    /** İsteği JWE formatında şifrele: {"encData":"..."} */
    public String encrypt(Object payload) throws Exception {
        String json = new ObjectMapper().writeValueAsString(payload);

        JWEHeader header = new JWEHeader.Builder(
                JWEAlgorithm.RSA_OAEP_256, EncryptionMethod.A128GCM)
                .keyID(props.getMle().getKeyId())
                .customParam("iat", System.currentTimeMillis())
                .build();

        JWEObject jwe = new JWEObject(header, new Payload(json));
        jwe.encrypt(new RSAEncrypter(loadPublicKey(props.getMle().getServerPublicCert())));

        return "{\"encData\":\"" + jwe.serialize() + "\"}";
    }

    /** Yanıtı JWE'den çöz */
    public <T> T decrypt(String encData, Class<T> type) throws Exception {
        JWEObject jwe = JWEObject.parse(encData);
        jwe.decrypt(new RSADecrypter(loadPrivateKey(props.getMle().getClientPrivateKey())));
        String json = jwe.getPayload().toString();
        return new ObjectMapper().readValue(json, type);
    }

    private RSAPublicKey loadPublicKey(String path) throws Exception {
        String pem = Files.readString(Path.of(path));
        String base64 = pem.replace("-----BEGIN CERTIFICATE-----", "")
                           .replace("-----END CERTIFICATE-----", "")
                           .replaceAll("\\s", "");
        byte[] der = Base64.getDecoder().decode(base64);
        Certificate cert = CertificateFactory.getInstance("X.509")
                .generateCertificate(new ByteArrayInputStream(der));
        return (RSAPublicKey) cert.getPublicKey();
    }

    private PrivateKey loadPrivateKey(String path) throws Exception {
        String pem = Files.readString(Path.of(path))
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replace("-----BEGIN RSA PRIVATE KEY-----", "")
                .replace("-----END RSA PRIVATE KEY-----", "")
                .replaceAll("\\s", "");
        byte[] der = Base64.getDecoder().decode(pem);
        return KeyFactory.getInstance("RSA")
                .generatePrivate(new PKCS8EncodedKeySpec(der));
    }
}
```

## 10.2. MLE'li Client

```java
public class MleApiClient extends VisaApiClient {

    private final MleEncryptionUtil mle;

    public MleApiClient(RestTemplate restTemplate, ObjectMapper objectMapper,
                        VisaProperties props, MleEncryptionUtil mle) {
        super(restTemplate, objectMapper, props);
        this.mle = mle;
    }

    public <T> T executeMle(HttpMethod method, String path, Object body,
                            Class<T> responseType) {
        try {
            String encryptedBody = body == null ? null : mle.encrypt(body);
            String url = props.getBaseUri() + path;
            HttpEntity<Object> entity = new HttpEntity<>(encryptedBody, buildHeaders());

            ResponseEntity<String> resp = restTemplate.exchange(
                    url, method, entity, String.class);

            if (resp.getBody() == null || resp.getBody().isBlank()) {
                return null;  // 204 No Content
            }

            JsonNode node = objectMapper.readTree(resp.getBody());
            if (node.has("encData")) {
                return mle.decrypt(node.get("encData").asText(), responseType);
            }
            return objectMapper.treeToValue(node, responseType);

        } catch (HttpClientErrorException e) {
            throw VisaClientException.from(e);
        } catch (Exception e) {
            throw new RuntimeException("MLE çağrısı başarısız", e);
        }
    }
}
```

## ✅ Doğrulama
- [ ] Sandbox'ta `POST /aliases` çağrısı 200 dönüyor
- [ ] `encData` içeren bir yanıt decrypt edilip parse ediliyor
- [ ] Private key loglarda görünmüyor

---

# 11. Adım 6: Visa API Modelleri

Visa'nın beklediği JSON yapısını **birebir** karşılayan DTO'lar. Lombok kullanın ama `@Data` **kullanmayın**.

## 11.1. İstek Modelleri

```java
// CreateAliasRequest.java
@Getter @Setter
public class CreateAliasRequest {
    @NotNull private String aliasType;
    @NotNull private String aliasValue;
    private String externalId;
    @Valid @NotNull private Consent consent;
    @Valid private Profile profile;
    @Valid @NotNull private List<PaymentCredential> paymentCredentials;
}

// Consent.java
@Getter @Setter
public class Consent {
    @NotNull private String presenter;
    @NotNull private String version;
    @NotNull
    @Pattern(regexp = "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z$")
    private String validFromDateTime;
    private String expiryDateTime;
    private List<String> intermediaries;
}

// Profile.java
@Getter @Setter
public class Profile {
    @NotNull private String firstName;
    @NotNull private String lastName;
    private String middleName;
    private String dateOfBirth;
    private List<ContactInfo> contactInfo;
}

// PaymentCredential.java (abstract, tip bazlı)
@Getter @Setter
@JsonTypeInfo(use = NAME, include = EXISTING_PROPERTY, property = "type", visible = true)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CardCredential.class, name = "CARD"),
    @JsonSubTypes.Type(value = BankCredential.class, name = "BANK")
})
public abstract class PaymentCredential {
    @NotNull private String type;
}

// CardCredential.java
@Getter @Setter
public class CardCredential extends PaymentCredential {
    @NotNull private String accountNumber;
    @NotNull private String cardType;
    @NotNull private String issuerName;
    @NotNull private String nameOnCard;
    @NotNull
    @Pattern(regexp = "^\\d{4}-(0[1-9]|1[0-2])$")
    private String expirationDate;
    @Valid @NotNull private BillingAddress billingAddress;
    private String externalId;
    private List<PreferredFor> preferredFor;
}

// BillingAddress.java
@Getter @Setter
public class BillingAddress {
    @NotNull private String country;
    private String addressLine1;
    private String addressLine2;
    private String city;
    private String postalCode;
    private String state;
}
```

## 11.2. Yanıt Modelleri

```java
@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class CreateAliasResponse {
    private String id;
    private String externalId;
    private List<PaymentCredentialSummary> paymentCredentials;
}

@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class PaymentCredentialSummary {
    private String id;
    private String type;
    private String externalId;
}

@Getter @Setter
@JsonIgnoreProperties(ignoreUnknown = true)
public class AliasIdResponse {
    private String id;
}
```

## ✅ Doğrulama
- [ ] Tüm modeller `@JsonIgnoreProperties(ignoreUnknown = true)`
- [ ] Jackson polymorphism çalışıyor (`type` ile CARD/BANK ayrımı)
- [ ] Unit test: JSON ↔ DTO dönüşümü test edildi

---

# 12. Adım 7: Visa Client Katmanı

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class VisaAliasClient {

    private static final String BASE = "/aliasdirectory/v1";

    private final MleApiClient client;

    public CreateAliasResponse createAlias(CreateAliasRequest request) {
        log.info("Visa createAlias başladı externalId={}", request.getExternalId());
        return client.executeMle(HttpMethod.POST, BASE + "/aliases",
                request, CreateAliasResponse.class);
    }

    public PaymentCredentialSummary createPaymentCredential(
            String aliasId, PaymentCredential request) {
        log.info("Visa createPaymentCredential başladı aliasId={}", aliasId);
        return client.executeMle(HttpMethod.POST,
                BASE + "/aliases/" + aliasId + "/paymentCredentials",
                request, PaymentCredentialSummary.class);
    }

    public AliasIdResponse findAliasIdByValue(String aliasValue) {
        try {
            Map<String, String> body = Map.of("aliasValue", aliasValue);
            return client.executeMle(HttpMethod.POST,
                    BASE + "/aliases/aliasId", body, AliasIdResponse.class);
        } catch (VisaClientException e) {
            if (e.getHttpStatus() == 404) return null;
            throw e;
        }
    }

    public void deleteAlias(String aliasId) {
        client.executeMle(HttpMethod.DELETE,
                BASE + "/aliases/" + aliasId, null, Void.class);
    }
}
```

## ✅ Doğrulama
- [ ] Sandbox'ta her metot manuel test edildi
- [ ] `X-Correlation-Id` header'da gönderiliyor
- [ ] `findAliasIdByValue` 404 → `null` dönüyor

---

# 13. Adım 8: Veritabanı Katmanı

## 13.1. Flyway Migration (`V1__create_visa_alias_records.sql`)

```sql
CREATE TABLE visa_alias_records (
    id                  BIGSERIAL PRIMARY KEY,
    request_id          VARCHAR(36) NOT NULL,
    alias_value         VARCHAR(255) NOT NULL,
    alias_type          VARCHAR(10) NOT NULL,
    account_number_hash VARCHAR(64) NOT NULL,
    account_last_four   VARCHAR(4),
    consent_json        JSONB,
    status              VARCHAR(20) NOT NULL,
    visa_alias_id       VARCHAR(36),
    visa_pc_id          VARCHAR(36),
    error_code          VARCHAR(50),
    error_message       VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX uk_request_id ON visa_alias_records (request_id);
CREATE UNIQUE INDEX uk_alias ON visa_alias_records (alias_type, alias_value);
CREATE INDEX idx_status ON visa_alias_records (status);
```

## 13.2. Entity

```java
@Entity
@Table(name = "visa_alias_records")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class VisaAliasRecord {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "request_id", nullable = false, unique = true)
    private String requestId;

    @Column(name = "alias_value", nullable = false)
    private String aliasValue;

    @Column(name = "alias_type", nullable = false)
    private String aliasType;

    @Column(name = "account_number_hash", nullable = false)
    private String accountNumberHash;

    @Column(name = "account_last_four")
    private String accountLastFour;

    @Column(name = "consent_json", columnDefinition = "jsonb")
    private String consentJson;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RecordStatus status;

    @Column(name = "visa_alias_id")
    private String visaAliasId;

    @Column(name = "visa_pc_id")
    private String visaPcId;

    @Column(name = "error_code")
    private String errorCode;

    @Column(name = "error_message")
    private String errorMessage;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    public static VisaAliasRecord pending(String requestId, String aliasType,
                                          String aliasValue, String hash,
                                          String last4, String consentJson) {
        VisaAliasRecord r = new VisaAliasRecord();
        r.requestId = requestId;
        r.aliasType = aliasType;
        r.aliasValue = aliasValue;
        r.accountNumberHash = hash;
        r.accountLastFour = last4;
        r.consentJson = consentJson;
        r.status = RecordStatus.PENDING;
        r.createdAt = Instant.now();
        r.updatedAt = Instant.now();
        return r;
    }

    public void markAliasCreated(String aliasId, String pcId) {
        this.visaAliasId = aliasId;
        this.visaPcId = pcId;
        this.status = RecordStatus.ALIAS_CREATED;
        this.updatedAt = Instant.now();
    }

    public void markPaymentCredentialAdded(String pcId) {
        this.visaPcId = pcId;
        this.status = RecordStatus.PC_ADDED;
        this.updatedAt = Instant.now();
    }

    public void linkExistingAlias(String aliasId) {
        this.visaAliasId = aliasId;
        this.updatedAt = Instant.now();
    }

    public void markFailed(String code, List<String> messages) {
        this.errorCode = code;
        this.errorMessage = messages == null ? null : String.join("; ", messages);
        this.status = RecordStatus.FAILED;
        this.updatedAt = Instant.now();
    }
}

enum RecordStatus { PENDING, ALIAS_CREATED, PC_ADDED, FAILED }
```

## 13.3. Repository

```java
public interface VisaAliasRecordRepository extends JpaRepository<VisaAliasRecord, Long> {
    Optional<VisaAliasRecord> findByRequestId(String requestId);
    Optional<VisaAliasRecord> findByAliasTypeAndAliasValue(String type, String value);
    List<VisaAliasRecord> findByStatus(RecordStatus status);
}
```

## 13.4. SensitiveDataHasher

```java
@Component
public class SensitiveDataHasher {

    @Value("${security.hash.pepper}")
    private String pepper;

    public String hash(String pan) {
        try {
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            md.update(pepper.getBytes(StandardCharsets.UTF_8));
            byte[] digest = md.digest(pan.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(digest);
        } catch (Exception e) {
            throw new IllegalStateException("Hash hatası", e);
        }
    }
}
```

## ✅ Doğrulama
- [ ] Flyway migration uygulandı (`flyway_schema_history`)
- [ ] `uk_request_id` unique constraint çalışıyor
- [ ] PAN hiçbir DB kolonunda düz metin değil

---

# 14. Adım 9: Orkestrasyon Servisi

İşin kalbi. Mobil isteği DB'ye yazıp Visa'ya yönlendiren katman.

## 14.1. `MobileAliasOrchestrationService`

```java
@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class MobileAliasOrchestrationService {

    private final VisaAliasClient visaClient;
    private final VisaAliasRecordRepository db;
    private final AliasRequestMapper mapper;
    private final SensitiveDataHasher hasher;
    private final AliasNormalizer normalizer;
    private final ObjectMapper objectMapper;

    public MobileAliasResponse handle(MobileAliasRequest req) {
        return db.findByRequestId(req.requestId())
                .map(this::toResponseFromEntity)
                .orElseGet(() -> process(req));
    }

    private MobileAliasResponse process(MobileAliasRequest req) {
        String aliasValue = normalizer.normalizeAliasValue(req.aliasValue(), req.aliasType());

        VisaAliasRecord record = db.save(VisaAliasRecord.pending(
                req.requestId(), req.aliasType(), aliasValue,
                hasher.hash(req.accountNumber()),
                last4(req.accountNumber()),
                toJson(req.consent())));

        Optional<VisaAliasRecord> existing =
                db.findByAliasTypeAndAliasValue(req.aliasType(), aliasValue)
                  .filter(r -> r.getVisaAliasId() != null);

        if (existing.isPresent()) {
            record.linkExistingAlias(existing.get().getVisaAliasId());
            db.save(record);
            return addPaymentCredential(record, req);
        }
        return createAliasOrRecover(record, req);
    }

    private MobileAliasResponse createAliasOrRecover(VisaAliasRecord record, MobileAliasRequest req) {
        try {
            CreateAliasRequest body = mapper.toCreateAliasRequest(req, record.getAliasValue());
            CreateAliasResponse resp = visaClient.createAlias(body);

            String pcId = resp.getPaymentCredentials().get(0).getId();
            record.markAliasCreated(resp.getId(), pcId);
            db.save(record);

            log.info("Alias oluşturuldu requestId={} aliasId={}",
                    req.requestId(), resp.getId());
            return MobileAliasResponse.aliasCreated(req.requestId(), resp.getId(), pcId);

        } catch (VisaClientException e) {
            if (isAliasAlreadyExists(e)) {
                return recoverFromExistingAlias(record, req);
            }
            record.markFailed(e.getErrorCode(), e.getErrorMessages());
            db.save(record);
            return MobileAliasResponse.failed(req.requestId(),
                    e.getErrorCode(), e.getErrorMessages());
        }
    }

    private MobileAliasResponse addPaymentCredential(VisaAliasRecord record, MobileAliasRequest req) {
        try {
            PaymentCredential pc = mapper.toPaymentCredential(req);
            PaymentCredentialSummary resp = visaClient.createPaymentCredential(
                    record.getVisaAliasId(), pc);

            record.markPaymentCredentialAdded(resp.getId());
            db.save(record);

            log.info("Kart eklendi requestId={} aliasId={} pcId={}",
                    req.requestId(), record.getVisaAliasId(), resp.getId());
            return MobileAliasResponse.paymentCredentialAdded(
                    req.requestId(), record.getVisaAliasId(), resp.getId());

        } catch (VisaClientException e) {
            record.markFailed(e.getErrorCode(), e.getErrorMessages());
            db.save(record);
            return MobileAliasResponse.failed(req.requestId(),
                    e.getErrorCode(), e.getErrorMessages());
        }
    }

    private MobileAliasResponse recoverFromExistingAlias(VisaAliasRecord record, MobileAliasRequest req) {
        AliasIdResponse aliasResp;
        try {
            aliasResp = visaClient.findAliasIdByValue(record.getAliasValue());
        } catch (Exception e) {
            log.error("Alias recovery başarısız requestId={}", req.requestId(), e);
            aliasResp = null;
        }

        if (aliasResp == null || aliasResp.getId() == null) {
            List<String> err = List.of("Alias var denildi ama ID alınamadı");
            record.markFailed("ORCH-001", err);
            db.save(record);
            return MobileAliasResponse.failed(req.requestId(), "ORCH-001", err);
        }

        record.linkExistingAlias(aliasResp.getId());
        db.save(record);
        return addPaymentCredential(record, req);
    }

    private boolean isAliasAlreadyExists(VisaClientException e) {
        return "VAD-1000".equals(e.getErrorCode())
            || "VAD-1040".equals(e.getErrorCode());
    }

    private MobileAliasResponse toResponseFromEntity(VisaAliasRecord r) {
        return switch (r.getStatus()) {
            case ALIAS_CREATED -> MobileAliasResponse.aliasCreated(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
            case PC_ADDED -> MobileAliasResponse.paymentCredentialAdded(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
            case FAILED -> MobileAliasResponse.failed(
                    r.getRequestId(), r.getErrorCode(),
                    r.getErrorMessage() == null ? List.of() : List.of(r.getErrorMessage()));
            case PENDING -> MobileAliasResponse.alreadyProcessed(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
        };
    }

    private String last4(String s) {
        return s == null || s.length() < 4 ? null : s.substring(s.length() - 4);
    }

    private String toJson(Object o) {
        try { return objectMapper.writeValueAsString(o); }
        catch (Exception e) { throw new RuntimeException(e); }
    }
}
```

## 14.2. `AliasNormalizer`

```java
@Component
public class AliasNormalizer {
    public String normalizeAliasValue(String value, String type) {
        if (value == null) return null;
        String trimmed = value.trim();
        if ("PHONE".equalsIgnoreCase(type) && trimmed.startsWith("+")) {
            return trimmed.substring(1);
        }
        return trimmed;
    }
}
```

## 14.3. `AliasRequestMapper`

```java
@Component
public class AliasRequestMapper {

    public CreateAliasRequest toCreateAliasRequest(MobileAliasRequest req, String normalizedAlias) {
        Profile profile = new Profile();
        profile.setFirstName(req.firstName());
        profile.setLastName(formatLastName(req.lastName()));

        Consent consent = new Consent();
        consent.setPresenter(req.consent().presenter());
        consent.setVersion(req.consent().version());
        consent.setValidFromDateTime(req.consent().validFromDateTime());

        CardCredential card = new CardCredential();
        card.setType("CARD");
        card.setAccountNumber(req.accountNumber());
        card.setCardType(req.cardType());
        card.setExpirationDate(req.expirationDate());
        card.setIssuerName(req.issuerName());
        card.setNameOnCard(req.nameOnCard());

        BillingAddress addr = new BillingAddress();
        addr.setCountry(req.billingCountry());
        card.setBillingAddress(addr);
        card.setExternalId(req.requestId());

        CreateAliasRequest body = new CreateAliasRequest();
        body.setAliasType(req.aliasType());
        body.setAliasValue(normalizedAlias);
        body.setExternalId(req.requestId());
        body.setConsent(consent);
        body.setProfile(profile);
        body.setPaymentCredentials(List.of(card));
        return body;
    }

    public PaymentCredential toPaymentCredential(MobileAliasRequest req) {
        CardCredential card = new CardCredential();
        card.setType("CARD");
        card.setAccountNumber(req.accountNumber());
        card.setCardType(req.cardType());
        card.setExpirationDate(req.expirationDate());
        card.setIssuerName(req.issuerName());
        card.setNameOnCard(req.nameOnCard());
        BillingAddress addr = new BillingAddress();
        addr.setCountry(req.billingCountry());
        card.setBillingAddress(addr);
        card.setExternalId(req.requestId());
        return card;
    }

    private String formatLastName(String lastName) {
        if (lastName == null || lastName.isBlank()) return lastName;
        return lastName.substring(0, 1).toUpperCase() + ".";
    }
}
```

## ✅ Doğrulama
- [ ] Happy path: yeni alias → `ALIAS_CREATED`
- [ ] Happy path: mevcut alias'a kart → `PC_ADDED`
- [ ] Aynı requestId iki kez → `ALREADY_PROCESSED`, Visa'ya 1 kez gidiyor
- [ ] `VAD-1040` → recovery çalışıyor
- [ ] PAN hiçbir log satırında yok

---

# 15. Adım 10: Controller ve DTO'lar

## 15.1. Mobil DTO'lar

```java
public record MobileAliasRequest(
    @NotBlank @Pattern(regexp = "[0-9a-fA-F-]{36}") String requestId,
    @NotBlank String aliasValue,
    @NotBlank @Pattern(regexp = "PHONE|EMAIL") String aliasType,
    @NotBlank String cardType,
    @NotBlank String accountNumber,
    @NotBlank String expirationDate,
    @NotBlank String issuerName,
    @NotBlank String nameOnCard,
    @NotBlank String billingCountry,
    @NotBlank String firstName,
    @NotBlank String lastName,
    @NotNull @Valid MobileConsent consent
) {}

public record MobileConsent(
    @NotBlank String presenter,
    @NotBlank String version,
    @NotBlank @Pattern(regexp = "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z$")
    String validFromDateTime
) {}
```

## 15.2. Response

```java
public enum MobileAliasStatus {
    ALIAS_CREATED, PAYMENT_CREDENTIAL_ADDED, ALREADY_PROCESSED, FAILED
}

@JsonInclude(JsonInclude.Include.NON_NULL)
public record MobileAliasResponse(
    String requestId,
    MobileAliasStatus status,
    String aliasId,
    String paymentCredentialId,
    String errorCode,
    List<String> errorMessages,
    Instant processedAt
) {
    public static MobileAliasResponse aliasCreated(String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.ALIAS_CREATED,
                aid, pcid, null, null, Instant.now());
    }
    public static MobileAliasResponse paymentCredentialAdded(String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.PAYMENT_CREDENTIAL_ADDED,
                aid, pcid, null, null, Instant.now());
    }
    public static MobileAliasResponse alreadyProcessed(String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.ALREADY_PROCESSED,
                aid, pcid, null, null, Instant.now());
    }
    public static MobileAliasResponse failed(String rid, String code, List<String> msgs) {
        return new MobileAliasResponse(rid, MobileAliasStatus.FAILED,
                null, null, code, msgs, Instant.now());
    }
}
```

## 15.3. Controller

```java
@RestController
@RequestMapping("/api/mobile/aliases")
@RequiredArgsConstructor
public class MobileAliasController {

    private final MobileAliasOrchestrationService orchestration;

    @PostMapping
    public ResponseEntity<MobileAliasResponse> handle(
            @Valid @RequestBody MobileAliasRequest req,
            @RequestHeader(value = "X-Api-Key") String apiKey) {

        MobileAliasResponse response = orchestration.handle(req);

        HttpStatus status = switch (response.status()) {
            case ALIAS_CREATED -> HttpStatus.CREATED;
            case PAYMENT_CREDENTIAL_ADDED, ALREADY_PROCESSED -> HttpStatus.OK;
            case FAILED -> HttpStatus.BAD_REQUEST;
        };
        return ResponseEntity.status(status).body(response);
    }
}
```

## ✅ Doğrulama
- [ ] Postman ile happy path çalışıyor
- [ ] Validation hataları 400 dönüyor
- [ ] HTTP status doğru (201/200/400)

---

# 16. Adım 11: Hata Yönetimi

## 16.1. `VisaClientException`

```java
@Getter
public class VisaClientException extends RuntimeException {
    private final int httpStatus;
    private final String errorCode;
    private final List<String> errorMessages;

    public VisaClientException(int httpStatus, String code, List<String> messages) {
        super("Visa hatası: " + httpStatus + " " + code);
        this.httpStatus = httpStatus;
        this.errorCode = code;
        this.errorMessages = messages;
    }

    public static VisaClientException from(HttpClientErrorException e) {
        try {
            JsonNode n = new ObjectMapper().readTree(e.getResponseBodyAsString());
            String code = n.has("errorCode") ? n.get("errorCode").asText() : "UNKNOWN";
            List<String> msgs = new ArrayList<>();
            if (n.has("errorMessages")) {
                n.get("errorMessages").forEach(m -> msgs.add(m.asText()));
            }
            return new VisaClientException(e.getStatusCode().value(), code, msgs);
        } catch (Exception ex) {
            return new VisaClientException(e.getStatusCode().value(), "PARSE_ERROR",
                    List.of(e.getMessage()));
        }
    }
}
```

## 16.2. GlobalExceptionHandler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, Object> body = new HashMap<>();
        body.put("errorCode", "VALIDATION_ERROR");
        body.put("errorMessages", ex.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .toList());
        return ResponseEntity.badRequest().body(body);
    }

    @ExceptionHandler(VisaClientException.class)
    public ResponseEntity<Map<String, Object>> handleVisa(VisaClientException ex) {
        log.warn("Visa hatası code={} status={}", ex.getErrorCode(), ex.getHttpStatus());
        return ResponseEntity.status(ex.getHttpStatus()).body(Map.of(
                "errorCode", ex.getErrorCode(),
                "errorMessages", ex.getErrorMessages()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<Map<String, Object>> handleGeneric(Exception ex) {
        log.error("Beklenmeyen hata", ex);
        return ResponseEntity.status(500).body(Map.of(
                "errorCode", "INTERNAL_ERROR",
                "errorMessages", List.of("Beklenmeyen bir hata oluştu")));
    }
}
```

> **Kural:** `ex.printStackTrace()` yok. Stack trace mobilize dönmez. PAN/şifre loglanmaz.

---

# 17. Adım 12: Test Stratejisi

## 17.1. Unit Test — Orkestrasyon

```java
@ExtendWith(MockitoExtension.class)
class MobileAliasOrchestrationServiceTest {

    @Mock VisaAliasClient visaClient;
    @Mock VisaAliasRecordRepository db;
    @Mock AliasRequestMapper mapper;
    @Mock SensitiveDataHasher hasher;
    @Mock AliasNormalizer normalizer;
    @Mock ObjectMapper objectMapper;
    @InjectMocks MobileAliasOrchestrationService service;

    @Test
    void givenNewAlias_whenHandle_thenAliasCreated() {
        var req = /* ... */;
        when(db.findByRequestId(any())).thenReturn(Optional.empty());
        when(normalizer.normalizeAliasValue(any(), any())).thenReturn("905551234567");
        when(hasher.hash(any())).thenReturn("HASH");
        when(db.save(any())).thenAnswer(inv -> inv.getArgument(0));
        when(db.findByAliasTypeAndAliasValue(any(), any())).thenReturn(Optional.empty());

        var visaResp = new CreateAliasResponse();
        visaResp.setId("alias-1");
        var pc = new PaymentCredentialSummary();
        pc.setId("pc-1");
        visaResp.setPaymentCredentials(List.of(pc));
        when(visaClient.createAlias(any())).thenReturn(visaResp);

        var resp = service.handle(req);

        assertThat(resp.status()).isEqualTo(MobileAliasStatus.ALIAS_CREATED);
        assertThat(resp.aliasId()).isEqualTo("alias-1");
        verify(visaClient, times(1)).createAlias(any());
    }

    @Test
    void givenSameRequestId_whenHandle_thenAlreadyProcessed() {
        var existing = VisaAliasRecord.pending("req-1", "PHONE", "90555", "H", "1142", "{}");
        existing.markAliasCreated("alias-1", "pc-1");
        when(db.findByRequestId("req-1")).thenReturn(Optional.of(existing));

        var resp = service.handle(/* req with same requestId */);

        assertThat(resp.status()).isEqualTo(MobileAliasStatus.ALREADY_PROCESSED);
        verifyNoInteractions(visaClient);
    }

    @Test
    void givenAliasAlreadyExists_whenCreateAlias_thenRecovers() {
        // when(visaClient.createAlias(...)).thenThrow(VAD-1040);
        // when(visaClient.findAliasIdByValue(...)).thenReturn("alias-existing");
        // when(visaClient.createPaymentCredential(...)).thenReturn(pc);
        // → response PAYMENT_CREDENTIAL_ADDED
    }
}
```

## 17.2. Integration Test — WireMock

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureMockMvc
class MobileAliasControllerIT {

    @Autowired MockMvc mockMvc;
    @RegisterExtension
    static WireMockExtension visa = WireMockExtension.newInstance().build();

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("visa.base-uri", visa::baseUrl);
    }

    @Test
    void whenValidRequest_thenCreated() throws Exception {
        visa.stubFor(post("/aliasdirectory/v1/aliases")
                .willReturn(okJson("{\"id\":\"alias-1\",\"paymentCredentials\":[{\"id\":\"pc-1\"}]}")));

        mockMvc.perform(post("/api/mobile/aliases")
                .contentType(APPLICATION_JSON)
                .header("X-Api-Key", "test")
                .content(/* JSON */))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("ALIAS_CREATED"));
    }
}
```

## 17.3. Log Capture Testi (PAN Kontrolü)

```java
@Test
void whenHandle_thenPanNotInLogs() {
    ListAppender<ILoggingEvent> appender = attachAppender();
    service.handle(/* req with PAN 4111111145551142 */);
    String logs = appender.list.stream()
            .map(ILoggingEvent::getFormattedMessage)
            .collect(Collectors.joining("\n"));
    assertThat(logs).doesNotContain("4111111145551142");
    assertThat(logs).contains("1142");
}
```

## 17.4. Zorunlu Test Case'leri

1. ✅ Yeni alias oluşturma → `ALIAS_CREATED`
2. ✅ Mevcut alias'a kart ekleme → `PC_ADDED`
3. ✅ Aynı `requestId` iki kez → `ALREADY_PROCESSED`, Visa'ya 1 kez gidilir
4. ✅ Race condition (Create Alias sırasında `VAD-1040`) → recovery
5. ✅ Visa 500 → retry policy doğru mu
6. ✅ Visa 400 → `FAILED` döner, retry yok
7. ✅ PAN loglanmıyor mu? (log capture testi)

## ✅ Doğrulama
- [ ] Unit test coverage > %70 (service için > %85)
- [ ] 7 zorunlu senaryo test edildi
- [ ] PAN log testi geçiyor

---

# BÖLÜM C — STANDARTLAR VE OPERASYON

# 18. Kodlama Standartları

## 18.1. Teknoloji

- **Java 17+** (record, sealed class, pattern matching)
- **Spring Boot 3.x**
- **Jakarta EE** namespace (javax değil)
- **Lombok**: `@Getter`, `@Builder`, `@RequiredArgsConstructor` kullan;
  `@Data` **kullanma** (entity'lerde equals/hashCode sorunları)

## 18.2. Naming Conventions

| Tür | Kural | Örnek |
|---|---|---|
| Controller | `*Controller` | `MobileAliasController` |
| Service | `*Service` | `MobileAliasOrchestrationService` |
| Visa HTTP client | `Visa*Client` | `VisaAliasClient` |
| Request DTO | `*Request` | `CreateAliasRequest` |
| Response DTO | `*Response` | `MobileAliasResponse` |
| Entity | Tekil isim | `VisaAliasRecord` |
| Repository | `*Repository` | `VisaAliasRecordRepository` |
| Exception | `*Exception` | `AliasOrchestrationException` |

## 18.3. Validation

- Jakarta Bean Validation (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`)
- Controller'da `@Valid`
- İş kuralı validasyonları Service'te
- Visa'ya gitmeden **önce** validate → gereksiz çağrı önle

## 18.4. Exception Hiyerarşisi

```
RuntimeException
 └── DomainException (abstract)
      ├── AliasNotFoundException
      ├── AliasOrchestrationException
      ├── VisaClientException        (Visa'dan dönen 4xx/5xx)
      └── VisaGatewayException       (Visa gateway hataları)
```

## 18.5. Logging Kuralları

**Yasak (asla loglanmaz):**
- PAN (tam kart numarası)
- CVV
- Şifre, token, private key
- Visa kimlik bilgileri

**Zorunlu (her log satırında):**
- `requestId` (korelasyon)
- `X-Correlation-Id` (Visa isteklerinde)
- Operation adı (`createAlias`, `createPaymentCredential`)

**Doğru pattern:**
```java
log.info("Visa alias create başladı requestId={} aliasType={}",
         requestId, aliasType);
log.debug("Kart son 4 hane: {}",
    accountNumber.substring(accountNumber.length() - 4));
```

---

# 19. Güvenlik Kuralları

## 19.1. MLE (Mesaj Şifreleme) — ZORUNLU

- Hassas veri içeren **tüm** Visa çağrılarında MLE kullanılır
- `withMLERequest(true)` **her zaman** true olmalı
- Algoritma: **RSA-OAEP-256 + A128GCM**
- İki çift anahtar: Visa server public cert + bizim client private key

## 19.2. İki Yönlü SSL (mTLS)

- `RestTemplate` bir keystore (`.jks`) ile yapılandırılır
- Keystore yolu ve şifresi **ortam değişkeni** olmalı
- **Asla** git'e commit edilmez

## 19.3. Sır Yönetimi

**Yasak:**
- API key, şifre, keystore şifresi koda gömme
- `application.properties`'e gerçek sır yazma
- Loglarda sır gösterme

**Zorunlu:**
- Tüm sırlar **ortam değişkeni** veya **Vault/AWS Secrets Manager**
- `.env.example` dosyası ile placeholder
- CI/CD'de sır enjeksiyonu

## 19.4. PCI-DSS / KVKK / GDPR

- PAN **asla** DB'ye yazılmaz → sadece hash + son 4 hane
- `userDetails.userName` **PII içermez** → anonim ID
- Log rotasyonu ve saklama süreleri şirket politikasına göre
- Silme talebi → `deleteAlias` + DB'den hard delete

## 19.5. Güvenlik Kontrol Listesi

### Sır ve Sertifika
- [ ] `VISA_PASSWORD`, keystore şifresi, MLE private key: **Vault/K8s Secrets**
- [ ] `application.yml`'de sadece `${...}` placeholder
- [ ] `.env`, `*.jks`, `*.pem`, `*.key` **`.gitignore`'da**
- [ ] `git grep -iE "(password|secret|BEGIN.*PRIVATE)"` → temiz

### Veri Güvenliği
- [ ] PAN DB'de sadece **hash + son 4 hane**
- [ ] PAN **hiçbir** log satırında yok
- [ ] Exception mesajlarında PAN yok
- [ ] `userDetails.userName` PII içermiyor

### API Güvenliği
- [ ] Mobil endpoint'te `X-Api-Key` zorunlu
- [ ] Tüm Visa çağrıları **HTTPS + mTLS**
- [ ] MLE tüm hassas endpoint'lerde aktif
- [ ] `X-Correlation-Id` her istekte üretiliyor

### Uyumluluk
- [ ] KVKK/GDPR: silme talebi → `deleteAlias` + DB hard delete
- [ ] Saklama süreleri tanımlı (log, DB)
- [ ] Denetim izi (audit trail) var mı?

---

# 20. Yasaklar ve Kontrol Listesi

## 20.1. Yasaklar (Anti-Patterns)

Agent / geliştirici **asla** aşağıdakileri yapmaz:

1. ❌ PAN'ı loglama veya exception mesajına ekleme
2. ❌ Sırları koda / properties'e gömme
3. ❌ `withMLERequest(false)` (hassas endpoint'lerde)
4. ❌ Controller'da `@Transactional`
5. ❌ Visa entity'sini doğrudan mobilize dönme
6. ❌ `e.printStackTrace()` (SLF4J kullan)
7. ❌ `catch (Exception e) {}` (boş catch)
8. ❌ Hard-coded URL / şifre / keyId
9. ❌ `@Data` entity'lerde
10. ❌ Retry olmadan 5xx/timeout yutma
11. ❌ Idempotency kontrolü olmadan POST çağrısı
12. ❌ Alias var/yok kontrolünü sadece Visa'ya bırakma (DB'de de tut)
13. ❌ `Thread.sleep()` retry için
14. ❌ Test edilmemiş hata yolu bırakma
15. ❌ Bulk `getAll` (pagination'sız)
16. ❌ `PUT /paymentCredentials/{id}` öncesi GET yapmama (tam replace!)
17. ❌ Alias `DISABLED` iken `resolve` çağırma
18. ❌ `+` ile telefon numarası gönderme

## 20.2. Her Kod Önerisinde Kontrol Listesi

### Fonksiyonel
- [ ] Alias var/yok senaryosu DB'den mi kontrol ediliyor?
- [ ] `requestId` idempotency için kullanılıyor mu?
- [ ] Race condition (VAD-1040) recovery var mı?
- [ ] State machine doğru geçiş yapıyor mu?
- [ ] Mobil ekibe `MobileAliasResponse` dönüyor mu?

### Güvenlik
- [ ] `withMLERequest(true)` var mı?
- [ ] PAN hiçbir yerde loglanmıyor mu?
- [ ] Sırlar ortam değişkeninden mi?
- [ ] `userDetails.userName` PII içermiyor mu?
- [ ] `@JsonInclude(NON_NULL)` response'ta var mı?

### Hata Yönetimi
- [ ] 4xx → retry yok, `FAILED` dönüyor mu?
- [ ] 5xx/timeout → retry + backoff var mı?
- [ ] Merkezi `@ControllerAdvice` var mı?
- [ ] Exception mesajında sır/PAN yok mu?

### Kod Kalitesi
- [ ] Controller → Service → Client → Repository katmanları doğru mu?
- [ ] DTO ↔ Entity ↔ API model ayrımı yapılmış mı?
- [ ] Validation `@Valid` ile mi?
- [ ] Test yazılmış mı (happy path + 2 hata senaryosu)?
- [ ] Naming convention'lara uyuldu mu?

## 20.3. Sık Sorulan Sorular

**S: Mobil ekip `+905551234567` gönderirse?**
C: `normalizeAliasValue()` ile `+` temizle; E.164 standardı `+` içermez.

**S: Alias `DISABLED` ise ne yapmalıyız?**
C: Yeni kart eklenebilir ama `resolve` çalışmaz. İş kararı gerekli; logla ve devam et.

**S: Aynı alias'a aynı kartı 2 kez eklemek isterse?**
C: Visa `CVC-30010` döner → `FAILED` yanıtla, mobilize "kart zaten ekli" mesajı ver.

**S: Visa 500 döndü, retry etmeli miyim?**
C: Evet, exponential backoff + jitter (max 3). Idempotency için aynı `requestId` kullan.

**S: `externalId` zorunlu mu?**
C: Teknik olarak hayır, ama idempotency için zorunlu kabul et. Mobil `requestId`'yi `externalId` olarak gönder.

**S: Batch processing kullanmalı mıyım?**
C: Sadece mobil tekli istek senaryosunda **hayır**. Bulk migration için ileride değerlendirilebilir.

**S: `PUT /paymentCredentials/{id}` neden tehlikeli?**
C: Tam replace yapar. Gönderilmeyen alanlar silinir. Önce GET, sonra PUT.

---

# 21. Canlıya Geçiş

## 21.1. Sandbox → Certification
- [ ] Tüm zorunlu test case'leri geçti
- [ ] Visa Developer → Certification ortamına geçiş başvurusu yapıldı
- [ ] Certification kimlik bilgileri ve sertifikaları alındı
- [ ] `VISA_BASE_URI=https://cert.api.visa.com/` ile tekrar test

## 21.2. Certification → Production
- [ ] Visa'nın verdiği sertifikasyon test case'leri tamamlandı
- [ ] **Sözleşme** imzalandı
- [ ] Production kimlik bilgileri ve sertifikaları alındı
- [ ] **Pilot faz** (sınırlı kullanıcıyla canlı test) — şiddetle önerilir
- [ ] Monitoring & alerting kuruldu:
  - Visa hata oranı > %1 → alarm
  - p99 latency > 3sn → alarm
  - `FAILED` kayıt sayısı → günlük rapor
- [ ] Runbook hazır (Visa down olursa ne yapılacak)
- [ ] **Müşteri destek** bilgilendirildi

## 21.3. Post-Launch
- [ ] İlk 24 saat: hata logları izlenir
- [ ] İlk 1 hafta: performans metrikleri
- [ ] İlk 1 ay: iş metrikleri (kaç alias oluşturuldu, kaç transfer)
- [ ] Sertifika rotasyon takvimi (1 yıl sonrası)

## 21.4. Tahmini Takvim

| Aşama | Süre |
|---|---|
| Geliştirme (Adım 5-17) | ~2-3 hafta |
| Sandbox E2E test | ~2 gün |
| Certification geçişi | ~1 hafta |
| Production + pilot | ~2-4 hafta |
| **Toplam** | **~5-7 hafta** |

---

# 22. Sorun Giderme

| Belirti | Olası Neden | Çözüm |
|---|---|---|
| `SSLHandshakeException` | Keystore yolu/şifresi yanlış | `keytool -list -keystore` ile doğrula |
| `403 Permission denied` | Basic Auth yanlış / IP allowlist | Credentials'ı kontrol et, Visa'ya IP bildir |
| `401 Unauthorized` | Kimlik bilgisi eksik | `visa.username`/`password` set mi? |
| MLE decrypt hatası | Client private key yanlış | `.pem` içeriğini kontrol et |
| `VAD-1000` | Alias zaten var | `findAliasIdByValue` → `createPaymentCredential` |
| `VAD-1040` | Original alias yok | Race recovery çalışmalı |
| `CVC-30010` | Kart zaten ekli | İş kararı: idempotent yanıt |
| Timeout | Ağ/proxy | `connectTimeout=10s`, `readTimeout=30s` |
| 429 Too Many Requests | Rate limit | `Retry-After` header'a uy |

### Debug İpuçları
- **`X-Correlation-Id`'yi logla** → Visa destek bu ID ile araştırır
- **Postman'de MLE'siz test edin** (sandbox'ta bazen MLE opsiyonel) → sorun MLE'de mi anlaşılır
- **`org.apache.hc.client5: DEBUG`** log seviyesini aç → SSL handshake detayları
- **Visa Developer portalı → API Test Console** → endpoint'i orada deneyin

---

# 23. Şirket Standartları (Doldurulacak)

> **Agent'a:** Aşağıdaki bölümleri şirket standartlarıyla doldur. Boş bırakılan
> alanlar için genel en iyi pratikler geçerlidir.

## 23.1. Mimari
- **Katman isimlendirme:** `[örn. controller → facade → service → repository]`
- **DTO dönüşümü:** `[örn. MapStruct kullanılıyor]`
- **Transaction yönetimi:** `[örn. @Transactional Service katmanında]`
- **Event yayınlama:** `[örn. Kafka/RabbitMQ varsa]`
- **Caching:** `[örn. Redis, key formatı]`

## 23.2. Kodlama
- **Paket yapısı:** `com.[şirket].[proje].[katman]...`
- **Lombok:** `[izin verilen/verilmeyen anotasyonlar]`
- **Log formatı:** `[JSON / pattern / MDC]`
- **Checkstyle/Spotless:** `[kurallar]`
- **Record vs POJO:** `[hangi durumda hangisi]`
- **Null handling:** `[Optional / null / @Nullable]`

## 23.3. Güvenlik
- **Sır yönetimi:** `[Vault / AWS / Azure Key Vault]`
- **Sertifika rotasyonu:** `[sıklık / otomasyon]`
- **Log maskeleme:** `[pattern]`
- **Güvenlik code review:** `[zorunlu mu]`

## 23.4. Hata Yönetimi
- **Hata yanıt formatı:** `[RFC 7807 / özel format]`
- **Retry kütüphanesi:** `[Resilience4j / Spring Retry]`
- **Dead letter queue:** `[başarısız mesajlar nereye]`
- **Alerting:** `[Slack/PagerDuty]`

## 23.5. Test
- **Coverage eşiği:** `[örn. %80]`
- **Test framework:** `[JUnit 5 / TestNG]`
- **Mock kütüphanesi:** `[Mockito / MockK]`
- **WireMock / Testcontainers:** `[kullanılıyor mu]`

---

## 📌 Son Not

Bu dosya **yaşayan bir dokümandır**. Visa API değiştiğinde, banka standartları
güncellendiğinde veya yeni pattern'ler benimsendiğinde güncellenmelidir.

- **Versiyonlama:** git history ile takip edin.
- **Sahip:** Entegrasyon ekibi lideri.
- **Gözden geçirme:** Her Visa API sürüm değişikliğinde ve 6 ayda bir.

**Visa Alias Directory 2.0 referansları:**
- API Reference: `https://developer.visa.com/api-reference`
- Getting Started: `https://developer.visa.com/docs/alias-directory-service/docs-how-to`
- Encryption Guide: `https://developer.visa.com/pages/encryption_guide`
- Destek: `developer@visa.com`