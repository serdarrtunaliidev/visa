# Agent.md — Visa Alias Directory Entegrasyon Rehberi

> **Bu dosya iki amaçla kullanılır:**
> 1. **Referans dokümanı:** Visa Alias Directory entegrasyonuyla ilgili tüm bilgi
>    (domain, endpoint'ler, iş kuralları, mimari, güvenlik, test).
> 2. **AI coding assistant (Copilot, Cursor, Claude Code) talimat dosyası:**
>    Kod yazarken uyulması gereken kurallar.
>
> Her kod önerisinden önce bu dosyayı oku. Çelişki durumunda **bu dosya önceliklidir.**
>
> **Versiyon:** 1.0 · **Son güncelleme:** 2026-11 · **Visa API:** Alias Directory 2.0

---

## İÇİNDEKİLER

1. [Domain Bilgisi](#1-domain-bilgisi)
2. [Endpoint Kataloğu](#2-endpoint-kataloğu)
3. [İş Kuralları (Zorunlu)](#3-iş-kuralları-zorunlu)
4. [Mimari ve Orkestrasyon](#4-mimari-ve-orkestrasyon)
5. [Kodlama Standartları](#5-kodlama-standartları)
6. [Güvenlik Kuralları](#6-güvenlik-kuralları)
7. [Hata Yönetimi ve Retry](#7-hata-yönetimi-ve-retry)
8. [Test Standartları](#8-test-standartları)
9. [Referans Kod Şablonları](#9-referans-kod-şablonları)
10. [Yasaklar ve Kontrol Listesi](#10-yasaklar-ve-kontrol-listesi)
11. [Şirket Standartları (Doldurulacak)](#11-şirket-standartları-doldurulacak)

---

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

## 4.4. DB Şeması (`visa_alias_records`)

```sql
CREATE TABLE visa_alias_records (
    id                  BIGSERIAL PRIMARY KEY,
    request_id          UUID NOT NULL UNIQUE,      -- idempotency anahtarı
    alias_value         VARCHAR(255) NOT NULL,
    alias_type          VARCHAR(10) NOT NULL,      -- PHONE / EMAIL
    account_number_hash VARCHAR(64) NOT NULL,      -- PAN hash (asla düz PAN)
    account_last_four   VARCHAR(4),
    consent_payload     JSONB,
    status              VARCHAR(20) NOT NULL,      -- PENDING, ALIAS_CREATED, PC_ADDED, FAILED
    visa_alias_id       UUID,
    visa_pc_id          UUID,
    error_code          VARCHAR(50),
    error_message       VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX uk_alias ON visa_alias_records (alias_type, alias_value);
```

## 4.5. Visa API Altyapı Bileşenleri

Visa'nın örnek projesinden alınması gereken altyapı sınıfları:

| Bileşen | Görevi |
|---|---|
| `VisaAPIRestTemplateProvider` | mTLS + SSLContext ile `RestTemplate` bean |
| `VisaApiClient` | Base HTTP client (Basic Auth ekler) |
| `MLEApiClient` | MLE (mesaj şifreleme) destekli client |
| `EncryptionUtil` | JWE şifreleme/çözme (RSA-OAEP-256 + A128GCM) |
| `VisaHttpBasicAuthProvider` | Basic Auth header |
| `FileUtils` | Sertifika dosyası yolu çözümleme |
| `EncryptedResponse` | MLE yanıt sarmalayıcı `{ "encData": "..." }` |

**Konfigürasyon (`application.yml`):**

```yaml
visa:
  baseUri: ${VISA_BASE_URI:https://sandbox.api.visa.com/}
  username: ${VISA_USERNAME}
  password: ${VISA_PASSWORD}
  keystore:
    path: ${VISA_KEYSTORE_PATH}
    password: ${VISA_KEYSTORE_PASSWORD}
  mle:
    keyId: ${VISA_MLE_KEY_ID}
    serverPublicCertificate: ${VISA_MLE_SERVER_CERT_PATH}
    privateKey: ${VISA_MLE_PRIVATE_KEY_PATH}
```

---

# 5. Kodlama Standartları

## 5.1. Teknoloji

- **Java 17+** (record, sealed class, pattern matching)
- **Spring Boot 3.x**
- **Jakarta EE** namespace (javax değil)
- **Lombok**: `@Getter`, `@Builder`, `@RequiredArgsConstructor` kullan;
  `@Data` **kullanma** (entity'lerde equals/hashCode sorunları)

## 5.2. Naming Conventions

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

## 5.3. Validation

- Jakarta Bean Validation (`@NotNull`, `@NotBlank`, `@Size`, `@Pattern`)
- Controller'da `@Valid`
- İş kuralı validasyonları Service'te
- Visa'ya gitmeden **önce** validate → gereksiz çağrı önle

```java
public record MobileAliasRequest(
    @NotBlank @Pattern(regexp = "[0-9a-fA-F-]{36}") String requestId,
    @NotBlank String aliasValue,
    @NotBlank @Pattern(regexp = "PHONE|EMAIL") String aliasType,
    @NotBlank String cardType,
    @NotBlank String accountNumber,
    @NotNull @Valid MobileConsent consent
) {}
```

## 5.4. Exception Hiyerarşisi

```
RuntimeException
 └── DomainException (abstract)
      ├── AliasNotFoundException
      ├── AliasOrchestrationException
      ├── VisaClientException        (Visa'dan dönen 4xx/5xx)
      └── VisaGatewayException       (Visa gateway hataları)
```

- `@RestControllerAdvice` ile merkezi hata yönetimi
- Hata yanıtı: `{ "errorCode": "...", "errorMessages": [...] }`
- **Stack trace mobilize dönmez**

## 5.5. Logging Kuralları

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
// PAN loglanmaz; sadece son 4 hane
log.debug("Kart son 4 hane: {}",
    accountNumber.substring(accountNumber.length() - 4));
```

---

# 6. Güvenlik Kuralları

## 6.1. MLE (Mesaj Şifreleme) — ZORUNLU

- Hassas veri içeren **tüm** Visa çağrılarında MLE kullanılır
- `VisaAPIRequest.withMLERequest(true)` **her zaman** true olmalı
- Algoritma: **RSA-OAEP-256 + A128GCM** (Visa'nın standardı)
- İki çift anahtar gerekir:
  - Visa server public cert (şifreleme için)
  - Bizim client private key (şifre çözme için)

## 6.2. İki Yönlü SSL (mTLS)

- `RestTemplate` bir keystore (`.jks`) ile yapılandırılır
- Keystore yolu ve şifresi **ortam değişkeni** olmalı
- **Asla** git'e commit edilmez; `.gitignore`'a eklenir
- `application.properties`'e gerçek sır yazılmaz

## 6.3. Sır Yönetimi

**Yasak:**
- API key, şifre, keystore şifresi koda gömme
- `application.properties`'e gerçek sır yazma
- Loglarda sır gösterme

**Zorunlu:**
- Tüm sırlar **ortam değişkeni** veya **Vault/AWS Secrets Manager**
- `.env.example` dosyası ile placeholder
- CI/CD'de sır enjeksiyonu

## 6.4. PCI-DSS / KVKK / GDPR

- PAN **asla** DB'ye yazılmaz → sadece hash + son 4 hane
- `userDetails.userName` **PII içermez** → anonim ID
- Log rotasyonu ve saklama süreleri şirket politikasına göre
- Silme talebi → `deleteAlias` + DB'den hard delete

## 6.5. MLE Tercih Sebebi

Sample projede her yerde `.withMLERequest(true)` kullanılıyor. **Siz de
kullanmalısınız.** Kart numarası gibi hassas veriler MLE olmadan Visa'ya
gönderilirse:
- Visa tarafından reddedilebilir
- PCI-DSS ihlali olur

---

# 7. Hata Yönetimi ve Retry

## 7.1. Retry Stratejisi

| Hata | Retry? | Strateji |
|---|---|---|
| 4xx (400, 404, 422) | ❌ | Düzelt, retry etme |
| 429 | ⚠️ | `Retry-After` header |
| 5xx (500, 503) | ⚠️ | Exponential backoff + jitter (max 3) |
| Timeout | ⚠️ | Backoff (max 3) |
| `VAD-1040` (alias exists) | ⚠️ | Race recovery |

**Backoff formülü:** `delay = min(base * 2^attempt, maxDelay) * random(0.5..1.5)`

## 7.2. Hata Yönetimi Pattern

```java
try {
    return visaClient.createAlias(request);
} catch (VisaClientException e) {
    // 4xx: iş hatası → FAILED olarak kaydet, kullanıcıya dön
    // 5xx/timeout: teknik hata → PENDING olarak bırak, retry job
    log.error("Visa çağrısı başarısız requestId={} status={} code={}",
              requestId, e.getStatusCode(), e.getErrorCode(), e);
    throw new AliasOrchestrationException("Alias oluşturulamadı", e);
}
```

## 7.3. Race Condition Recovery

```java
try {
    return createAlias(record, req);
} catch (VisaClientException e) {
    if (isAliasAlreadyExists(e)) {  // VAD-1000 veya VAD-1040
        return recoverFromExistingAlias(record, req);
    }
    throw e;
}
```

## 7.4. Merkezi Hata Yönetimi

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(VisaClientException.class)
    public ResponseEntity<MobileAliasResponse> handleVisa(VisaClientException e) {
        return ResponseEntity.badRequest().body(
            MobileAliasResponse.failed(
                extractRequestId(e), e.getErrorCode(), e.getErrorMessages()));
    }
}
```

---

# 8. Test Standartları

## 8.1. Test Piramidi

- **Unit test** (JUnit 5 + Mockito): Service ve domain logic
- **Integration test** (`@SpringBootTest` + WireMock): Client katmanı
- **Contract test**: Visa API sözleşmesine karşı
- **Sandbox test**: Gerçek Visa sandbox ile manuel/E2E

## 8.2. Zorunlu Test Case'leri

**Orkestrasyon servisi için:**
1. ✅ Yeni alias oluşturma → `ALIAS_CREATED`
2. ✅ Mevcut alias'a kart ekleme → `PC_ADDED`
3. ✅ Aynı `requestId` iki kez → `ALREADY_PROCESSED`, Visa'ya 1 kez gidilir
4. ✅ Race condition (Create Alias sırasında `VAD-1040`) → recovery
5. ✅ Visa 500 → retry policy doğru mu
6. ✅ Visa 400 → `FAILED` döner, retry yok
7. ✅ PAN loglanmıyor mu? (log capture testi)

## 8.3. Test Naming

```java
@Test
void givenNewAlias_whenHandle_thenReturnsAliasCreated() { ... }

@Test
void givenExistingAlias_whenHandle_thenAddsPaymentCredential() { ... }

@Test
void givenDuplicateRequestId_whenHandle_thenReturnsAlreadyProcessed() { ... }
```

---

# 9. Referans Kod Şablonları

## 9.1. Controller DTO'ları

### `MobileAliasRequest`

```java
package com.[şirket].[proje].dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;

public record MobileAliasRequest(
    @NotBlank @Pattern(regexp = "[0-9a-fA-F-]{36}") String requestId,
    @NotBlank String aliasValue,
    @NotBlank @Pattern(regexp = "PHONE|EMAIL") String aliasType,
    @NotBlank String cardType,
    @NotBlank String accountNumber,
    @NotNull @Valid MobileConsent consent
) {}

public record MobileConsent(
    @NotBlank String presenter,
    @NotBlank String version,
    @NotBlank @Pattern(regexp = "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}\\.\\d{3}Z$")
    String validFromDateTime
) {}
```

### `MobileAliasStatus`

```java
package com.[şirket].[proje].dto;

public enum MobileAliasStatus {
    ALIAS_CREATED,
    PAYMENT_CREDENTIAL_ADDED,
    ALREADY_PROCESSED,
    FAILED;

    public boolean isSuccess() {
        return this != FAILED;
    }
}
```

### `MobileAliasResponse`

```java
package com.[şirket].[proje].dto;

import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.Instant;
import java.util.List;

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
    public static MobileAliasResponse aliasCreated(
            String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.ALIAS_CREATED,
                aid, pcid, null, null, Instant.now());
    }

    public static MobileAliasResponse paymentCredentialAdded(
            String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.PAYMENT_CREDENTIAL_ADDED,
                aid, pcid, null, null, Instant.now());
    }

    public static MobileAliasResponse alreadyProcessed(
            String rid, String aid, String pcid) {
        return new MobileAliasResponse(rid, MobileAliasStatus.ALREADY_PROCESSED,
                aid, pcid, null, null, Instant.now());
    }

    public static MobileAliasResponse failed(
            String rid, String code, List<String> msgs) {
        return new MobileAliasResponse(rid, MobileAliasStatus.FAILED,
                null, null, code, msgs, Instant.now());
    }

    public boolean isSuccess() {
        return status != null && status.isSuccess();
    }
}
```

## 9.2. Entity — `VisaAliasRecord`

```java
package com.[şirket].[proje].domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;
import java.util.List;

@Entity
@Table(name = "visa_alias_records",
       uniqueConstraints = {
           @UniqueConstraint(name = "uk_request_id", columnNames = "request_id"),
           @UniqueConstraint(name = "uk_alias",
                             columnNames = {"alias_type", "alias_value"})
       })
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class VisaAliasRecord {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "request_id", nullable = false, length = 36)
    private String requestId;

    @Column(name = "alias_value", nullable = false)
    private String aliasValue;

    @Column(name = "alias_type", nullable = false, length = 10)
    private String aliasType;

    @Column(name = "account_number_hash", nullable = false, length = 64)
    private String accountNumberHash;

    @Column(name = "account_last_four", length = 4)
    private String accountLastFour;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private RecordStatus status;

    @Column(name = "visa_alias_id", length = 36)
    private String visaAliasId;

    @Column(name = "visa_pc_id", length = 36)
    private String visaPcId;

    @Column(name = "error_code", length = 50)
    private String errorCode;

    @Column(name = "error_message", length = 500)
    private String errorMessage;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    // Factory
    public static VisaAliasRecord pending(
            String requestId, String aliasValue, String aliasType,
            String accountNumberHash, String accountLastFour) {
        var r = new VisaAliasRecord();
        r.requestId = requestId;
        r.aliasValue = aliasValue;
        r.aliasType = aliasType;
        r.accountNumberHash = accountNumberHash;
        r.accountLastFour = accountLastFour;
        r.status = RecordStatus.PENDING;
        r.createdAt = Instant.now();
        r.updatedAt = Instant.now();
        return r;
    }

    // State transitions
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

## 9.3. Repository

```java
public interface VisaAliasRecordRepository
        extends JpaRepository<VisaAliasRecord, Long> {

    Optional<VisaAliasRecord> findByRequestId(String requestId);

    Optional<VisaAliasRecord> findByAliasTypeAndAliasValue(
            String aliasType, String aliasValue);
}
```

## 9.4. Orkestrasyon Servisi

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class MobileAliasOrchestrationService {

    private final VisaAliasClient visaClient;
    private final VisaAliasRecordRepository dbRepository;
    private final AliasRequestMapper mapper;
    private final SensitiveDataHasher hasher;

    @Transactional
    public MobileAliasResponse handle(MobileAliasRequest req) {
        // 1) İdempotency
        return dbRepository.findByRequestId(req.requestId())
                .map(existing -> toResponse(existing))
                .orElseGet(() -> process(req));
    }

    private MobileAliasResponse process(MobileAliasRequest req) {
        String aliasValue = normalizeAliasValue(req.aliasValue(), req.aliasType());

        VisaAliasRecord record = dbRepository.save(VisaAliasRecord.pending(
                req.requestId(), aliasValue, req.aliasType(),
                hasher.hash(req.accountNumber()),
                last4(req.accountNumber())));

        // DB'de alias zaten var mı?
        return dbRepository.findByAliasTypeAndAliasValue(
                        req.aliasType(), aliasValue)
                .filter(r -> r.getVisaAliasId() != null)
                .map(existing -> {
                    record.linkExistingAlias(existing.getVisaAliasId());
                    dbRepository.save(record);
                    return addPaymentCredential(record, req);
                })
                .orElseGet(() -> createAliasOrRecover(record, req));
    }

    private MobileAliasResponse addPaymentCredential(
            VisaAliasRecord record, MobileAliasRequest req) {
        try {
            var resp = visaClient.createPaymentCredential(
                    record.getVisaAliasId(),
                    mapper.toPaymentCredential(req));
            record.markPaymentCredentialAdded(resp.id());
            dbRepository.save(record);
            log.info("Kart eklendi requestId={} aliasId={} pcId={}",
                    req.requestId(), record.getVisaAliasId(), resp.id());
            return MobileAliasResponse.paymentCredentialAdded(
                    req.requestId(), record.getVisaAliasId(), resp.id());
        } catch (VisaClientException e) {
            record.markFailed(e.getErrorCode(), e.getErrorMessages());
            dbRepository.save(record);
            return MobileAliasResponse.failed(
                    req.requestId(), e.getErrorCode(), e.getErrorMessages());
        }
    }

    private MobileAliasResponse createAliasOrRecover(
            VisaAliasRecord record, MobileAliasRequest req) {
        try {
            var resp = visaClient.createAlias(mapper.toCreateAliasRequest(req));
            record.markAliasCreated(resp.id(),
                    resp.paymentCredentials().get(0).id());
            dbRepository.save(record);
            log.info("Alias oluşturuldu requestId={} aliasId={}",
                    req.requestId(), resp.id());
            return MobileAliasResponse.aliasCreated(
                    req.requestId(), resp.id(),
                    resp.paymentCredentials().get(0).id());
        } catch (VisaClientException e) {
            if (isAliasAlreadyExists(e)) {
                return recoverFromExistingAlias(record, req);
            }
            record.markFailed(e.getErrorCode(), e.getErrorMessages());
            dbRepository.save(record);
            return MobileAliasResponse.failed(
                    req.requestId(), e.getErrorCode(), e.getErrorMessages());
        }
    }

    private MobileAliasResponse recoverFromExistingAlias(
            VisaAliasRecord record, MobileAliasRequest req) {
        String aliasId = visaClient.findAliasIdByValue(record.getAliasValue());
        if (aliasId == null) {
            var err = List.of("Alias var denildi ama ID alınamadı");
            record.markFailed("ORCH-001", err);
            dbRepository.save(record);
            return MobileAliasResponse.failed(
                    req.requestId(), "ORCH-001", err);
        }
        record.linkExistingAlias(aliasId);
        dbRepository.save(record);
        return addPaymentCredential(record, req);
    }

    private boolean isAliasAlreadyExists(VisaClientException e) {
        return "VAD-1000".equals(e.getErrorCode())
            || "VAD-1040".equals(e.getErrorCode());
    }

    private MobileAliasResponse toResponse(VisaAliasRecord r) {
        return switch (r.getStatus()) {
            case ALIAS_CREATED -> MobileAliasResponse.aliasCreated(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
            case PC_ADDED -> MobileAliasResponse.paymentCredentialAdded(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
            case FAILED -> MobileAliasResponse.failed(
                    r.getRequestId(), r.getErrorCode(),
                    List.of(r.getErrorMessage() == null ? "" : r.getErrorMessage()));
            case PENDING -> MobileAliasResponse.alreadyProcessed(
                    r.getRequestId(), r.getVisaAliasId(), r.getVisaPcId());
        };
    }

    private String normalizeAliasValue(String value, String type) {
        if ("PHONE".equals(type) && value.startsWith("+")) {
            return value.substring(1);
        }
        return value;
    }

    private String last4(String s) {
        return s == null || s.length() < 4 ? null : s.substring(s.length() - 4);
    }
}
```

## 9.5. Controller

```java
@RestController
@RequestMapping("/mobile/aliases")
@RequiredArgsConstructor
public class MobileAliasController {

    private final MobileAliasOrchestrationService orchestration;

    @PostMapping
    public ResponseEntity<MobileAliasResponse> handle(
            @Valid @RequestBody MobileAliasRequest request) {
        MobileAliasResponse response = orchestration.handle(request);
        HttpStatus status = switch (response.status()) {
            case ALIAS_CREATED -> HttpStatus.CREATED;
            case PAYMENT_CREDENTIAL_ADDED, ALREADY_PROCESSED -> HttpStatus.OK;
            case FAILED -> HttpStatus.BAD_REQUEST;
        };
        return ResponseEntity.status(status).body(response);
    }
}
```

## 9.6. Visa Client (İskelet)

```java
@Component
@RequiredArgsConstructor
public class VisaAliasClient {

    private static final String BASE_PATH = "/aliasdirectory/v1";

    private final VisaApiClient apiClient;
    private final ExceptionHandler exceptionHandler;

    public CreateAliasResponse createAlias(CreateAliasRequest body) {
        var apiReq = new VisaAPIRequest<CreateAliasResponse>(
                BASE_PATH + "/aliases", HttpMethod.POST,
                new HttpHeaders(),
                asList(MediaType.APPLICATION_JSON),
                MediaType.APPLICATION_JSON)
            .withBody(body)
            .withMLERequest(true)   // ZORUNLU
            .withResponseTypeRef(
                new ParameterizedTypeReference<>() {},
                CreateAliasResponse.class);
        return apiClient.executeVisaAPIRequest(apiReq,
                exceptionHandler.customErrorCodeExceptionHandler(errorCodeMap()));
    }

    public PaymentCredentialResponse createPaymentCredential(
            String aliasId, PaymentCredentialRequest body) {
        var apiReq = new VisaAPIRequest<PaymentCredentialResponse>(
                BASE_PATH + "/aliases/" + aliasId + "/paymentCredentials",
                HttpMethod.POST,
                new HttpHeaders(),
                asList(MediaType.APPLICATION_JSON),
                MediaType.APPLICATION_JSON)
            .withBody(body)
            .withMLERequest(true)
            .withResponseTypeRef(
                new ParameterizedTypeReference<>() {},
                PaymentCredentialResponse.class);
        return apiClient.executeVisaAPIRequest(apiReq,
                exceptionHandler.customErrorCodeExceptionHandler(errorCodeMap()));
    }

    public String findAliasIdByValue(String aliasValue) {
        try {
            var body = new AliasIdRequest(aliasValue);
            var apiReq = new VisaAPIRequest<AliasIdResponse>(
                    BASE_PATH + "/aliases/aliasId", HttpMethod.POST,
                    new HttpHeaders(),
                    asList(MediaType.APPLICATION_JSON),
                    MediaType.APPLICATION_JSON)
                .withBody(body)
                .withMLERequest(true)
                .withResponseTypeRef(
                    new ParameterizedTypeReference<>() {},
                    AliasIdResponse.class);
            var resp = apiClient.executeVisaAPIRequest(apiReq,
                    exceptionHandler.customErrorCodeExceptionHandler(errorCodeMap()));
            return resp.id();
        } catch (VisaClientException e) {
            if (e.getStatusCode() == HttpStatus.NOT_FOUND) return null;
            throw e;
        }
    }

    private static Map<String, String> errorCodeMap() {
        return Map.of(
            "VAD-1000", "Data mismatch",
            "VAD-1040", "Original alias bulunamadı",
            "VAD-1120", "Internal error, retry",
            "VAD-1130", "Internal error, retry",
            "VAD-1260", "Default hesap silindi",
            "default",  "İşlem başarısız"
        );
    }
}
```

## 9.7. `application.yml` ve `.env.example`

```yaml
# application.yml
visa:
  baseUri: ${VISA_BASE_URI:https://sandbox.api.visa.com/}
  username: ${VISA_USERNAME}
  password: ${VISA_PASSWORD}
  keystore:
    path: ${VISA_KEYSTORE_PATH}
    password: ${VISA_KEYSTORE_PASSWORD}
  mle:
    keyId: ${VISA_MLE_KEY_ID}
    serverPublicCertificate: ${VISA_MLE_SERVER_CERT_PATH}
    privateKey: ${VISA_MLE_PRIVATE_KEY_PATH}

logging:
  level:
    com.[şirket].proje: INFO
    org.apache.hc.client5: WARN
```

```bash
# .env.example
VISA_BASE_URI=https://sandbox.api.visa.com/
VISA_USERNAME=
VISA_PASSWORD=
VISA_KEYSTORE_PATH=classpath:certs/visa_keystore.jks
VISA_KEYSTORE_PASSWORD=
VISA_MLE_KEY_ID=
VISA_MLE_SERVER_CERT_PATH=classpath:certs/visa_server_cert.pem
VISA_MLE_PRIVATE_KEY_PATH=classpath:certs/visa_client_private_key.pem
```

---

# 10. Yasaklar ve Kontrol Listesi

## 10.1. Yasaklar (Anti-Patterns)

Agent **asla** aşağıdakileri yapmaz:

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
13. ❌ `Thread.sleep()` retry için (backoff kütüphanesi kullan)
14. ❌ Test edilmemiş hata yolu bırakma
15. ❌ Bulk `getAll` (pagination'sız)
16. ❌ `PUT /paymentCredentials/{id}` öncesi GET yapmama (tam replace!)
17. ❌ Alias `DISABLED` iken `resolve` çağırma
18. ❌ `+` ile telefon numarası gönderme

## 10.2. Her Kod Önerisinde Kontrol Listesi

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

## 10.3. Sık Sorulan Sorular

**S: Mobil ekip `+905551234567` gönderirse?**
C: `normalizeAliasValue()` ile `+` temizle; E.164 standardı `+` içermez.

**S: Alias `DISABLED` ise ne yapmalıyız?**
C: Yeni kart eklenebilir ama `resolve` çalışmaz. İş kararı gerekli; logla
ve devam et.

**S: Aynı alias'a aynı kartı 2 kez eklemek isterse?**
C: Visa `CVC-30010` döner → `FAILED` yanıtla, mobilize "kart zaten ekli" mesajı ver.

**S: Visa 500 döndü, retry etmeli miyim?**
C: Evet, exponential backoff + jitter (max 3). Idempotency için aynı
`requestId` kullan.

**S: `externalId` zorunlu mu?**
C: Teknik olarak hayır, ama idempotency için zorunlu kabul et.
Mobil `requestId`'yi `externalId` olarak gönder.

**S: Batch processing kullanmalı mıyım?**
C: Sadece mobil tekli istek senaryosunda **hayır**. Bulk migration için
ileride değerlendirilebilir.

**S: `PUT /paymentCredentials/{id}` neden tehlikeli?**
C: Tam replace yapar. Gönderilmeyen alanlar silinir. Önce GET, sonra PUT.

---

# 11. Şirket Standartları (Doldurulacak)

> **Agent'a:** Aşağıdaki bölümleri şirket standartlarıyla doldur. Boş
> bırakılan alanlar için genel en iyi pratikler geçerlidir.

## 11.1. Mimari

- **Katman isimlendirme:** `[örn. controller → facade → service → repository]`
- **DTO dönüşümü:** `[örn. MapStruct kullanılıyor]`
- **Transaction yönetimi:** `[örn. @Transactional Service katmanında]`
- **Event yayınlama:** `[örn. Kafka/RabbitMQ varsa]`
- **Caching:** `[örn. Redis, key formatı]`

## 11.2. Kodlama

- **Paket yapısı:** `com.[şirket].[proje].[katman]...`
- **Lombok:** `[izin verilen/verilmeyen anotasyonlar]`
- **Log formatı:** `[JSON / pattern / MDC]`
- **Checkstyle/Spotless:** `[kurallar]`
- **Record vs POJO:** `[hangi durumda hangisi]`
- **Null handling:** `[Optional / null / @Nullable]`

## 11.3. Güvenlik

- **Sır yönetimi:** `[Vault / AWS / Azure Key Vault]`
- **Sertifika rotasyonu:** `[sıklık / otomasyon]`
- **Log maskeleme:** `[pattern]`
- **Güvenlik code review:** `[zorunlu mu]`

## 11.4. Hata Yönetimi

- **Hata yanıt formatı:** `[RFC 7807 / özel format]`
- **Retry kütüphanesi:** `[Resilience4j / Spring Retry]`
- **Dead letter queue:** `[başarısız mesajlar nereye]`
- **Alerting:** `[Slack/PagerDuty]`

## 11.5. Test

- **Coverage eşiği:** `[örn. %80]`
- **Test framework:** `[JUnit 5 / TestNG]`
- **Mock kütüphanesi:** `[Mockito / MockK]`
- **WireMock / Testcontainers:** `[kullanılıyor mu]`

---

## 📌 Son Not

Bu dosya **yaşayan bir dokümandır**. Visa API değiştiğinde, şirket standartları
güncellendiğinde veya yeni pattern'ler benimsendiğinde güncellenmelidir.

- **Versiyonlama:** git history ile takip edin.
- **Sahip:** Entegrasyon ekibi lideri.
- **Gözden geçirme:** Her Visa API sürüm değişikliğinde ve 6 ayda bir.

**Visa Alias Directory 2.0 referansları:**
- API Reference: `https://developer.visa.com/api-reference`
- Getting Started: `https://developer.visa.com/docs/alias-directory-service/docs-how-to`
- Encryption Guide: `https://developer.visa.com/pages/encryption_guide`
- Destek: `developer@visa.com`