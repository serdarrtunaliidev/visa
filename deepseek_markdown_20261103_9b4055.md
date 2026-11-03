# Visa Alias Directory — DataPower API Gateway ile Spring Boot Entegrasyon Rehberi

> **Versiyon:** 1.0 · **Visa API:** Alias Directory 2.0 · **Hedef:** Java 17 / Spring Boot 3.x
>
> **Mimari özeti:**
> ```
> Spring Boot  →  DataPower API Gateway  →  Visa Alias Directory
>    │                    │                          │
>    │  mTLS              │                          │
>    │  Basic Auth        │  mTLS                    │
>    │  MLE (encData)     │  Basic Auth (pass)       │
>    │                    │  encData (pass)          │
>    └───────────────────►└─────────────────────────►│
> ```
>
> **Spring Boot üstlenir:** mTLS, Basic Auth, MLE (şifreleme/çözme), iş mantığı, DB, orkestrasyon, idempotency.
> **DataPower üstlenir:** `/visa-alias` prefix strip + Visa'ya path forward, kendi mTLS'i ile Visa'ya bağlanma. **MLE'ye ve Basic Auth header'ına dokunmaz** (pass-through).
>
> **Spring Boot'ta tek değişiklik:** `visa.base-uri` DataPower'ı gösterir. **Java kodu aynı kalır.**

---

## 📋 İÇİNDEKİLER

**BÖLÜM A — Referans Bilgisi**
1. [Domain Bilgisi](#1-domain-bilgisi)
2. [Endpoint Kataloğu](#2-endpoint-kataloğu)
3. [İş Kuralları](#3-iş-kuralları)
4. [Mimari ve Orkestrasyon](#4-mimari-ve-orkestrasyon)

**BÖLÜM B — Spring Boot İmplementasyonu**
5. [Adım 0: Ön Gereksinimler](#5-adım-0-ön-gereksinimler)
6. [Adım 1: Proje İskeleti](#6-adım-1-proje-iskeleti)
7. [Adım 2: Konfigürasyon](#7-adım-2-konfigürasyon)
8. [Adım 3: Sertifika ve MLE Anahtarları](#8-adım-3-sertifika-ve-mle-anahtarları)
9. [Adım 4: SSL/RestTemplate (mTLS)](#9-adım-4-sslresttemplate-mtls)
10. [Adım 5: MLE Katmanı](#10-adım-5-mle-katmanı)
11. [Adım 6: Visa API Modelleri](#11-adım-6-visa-api-modelleri)
12. [Adım 7: Visa Client](#12-adım-7-visa-client)
13. [Adım 8: Veritabanı](#13-adım-8-veritabanı)
14. [Adım 9: Orkestrasyon](#14-adım-9-orkestrasyon)
15. [Adım 10: Controller ve DTO'lar](#15-adım-10-controller-ve-dtolar)
16. [Adım 11: Hata Yönetimi](#16-adım-11-hata-yönetimi)
17. [Adım 12: Test](#17-adım-12-test)

**BÖLÜM C — DataPower Yapılandırması**
18. [DataPower Path Mapping](#18-datapower-path-mapping)
19. [DataPower'ın Yapması Gerekenler](#19-datapowerın-yapması-gerekenler)
20. [Test ve Doğrulama](#20-test-ve-doğrulama)

**BÖLÜM D — Standartlar ve Operasyon**
21. [Kodlama Standartları](#21-kodlama-standartları)
22. [Güvenlik Kuralları](#22-güvenlik-kuralları)
23. [Yasaklar ve Kontrol Listesi](#23-yasaklar-ve-kontrol-listesi)
24. [Canlıya Geçiş](#24-canlıya-geçiş)
25. [Sorun Giderme](#25-sorun-giderme)
26. [Şirket Standartları](#26-şirket-standartları)

---

# BÖLÜM A — REFERANS BİLGİSİ

# 1. Domain Bilgisi

## 1.1. Visa Alias Directory Nedir?

Hassas ödeme bilgilerinin (kart numarası, banka hesabı) yerine **"alias"** adı verilen tanımlayıcıları (telefon, e-posta) kullanan bir dizin servisidir.

**Senaryo:** Kullanıcı "Ahmet'e 100 TL gönder" der → sistem Ahmet'in telefonunu (alias) kullanır → Visa arka planda Ahmet'in kartını/hesabını çözer → push payment başlatılır. Kullanıcı kart numarası görmez.

## 1.2. Temel Kavramlar

| Kavram | Tanım | Sorumluluk |
|---|---|---|
| **Alias** | Telefon/e-posta gibi müşteri tanımlayıcı | `aliasValue` + `aliasType` |
| **Profile** | Alias sahibinin adı/adresi | `firstName`, `lastName`, `contactInfo[]` |
| **Consent** | Rıza bilgisi | `presenter`, `version`, `validFromDateTime`, `expiryDateTime` |
| **Payment Credential** | Kart veya banka hesabı | `CARD` veya `BANK` |
| **External ID** | Müşteri tarafı ID | İdempotency anahtarı (`requestId`) |
| **Participant** | Alias Directory'ye katılan kurum | Bizim kurumumuz |

## 1.3. Alias Directory 1.0 vs 2.0

| Özellik | 1.0 | 2.0 (Bizim) |
|---|---|---|
| Ödeme aracı tipleri | Sadece kart | Kart + Banka hesabı |
| Veri yerelleştirme | Yok | Var |
| Raporlama | Temel | Gelişmiş |
| Toplu işlem | Yok | Batch Processing |
| Remote directory | Yok | Directory Network Manager |

> 1.0 kullanmayın. 2.0 endpoint'lerini kullanın.

## 1.4. Visa Ortamları

| Ortam | Base URL |
|---|---|
| **Sandbox** | `https://sandbox.api.visa.com` |
| **Certification** | `https://cert.api.visa.com` |
| **Production** | `https://api.visa.com` |

> **DataPower kullanıldığında** bu URL'ler **DataPower'ın backend URL'i** olur. Spring Boot sadece DataPower'ın adresini bilir.

## 1.5. Alias Value Formatları

- **PHONE:** ITU-T E.164, **`+` işareti OLMADAN** (ör. `905551234567`). Mobil ekip `+` gönderirse `normalizeAliasValue()` ile temizle.
- **EMAIL:** Standart format.
- **PAYNAME:** `+personalidentifier.domain` (Visa+ için, gelecekte).

## 1.6. Yaşam Döngüsü

```
PENDING ─── ALIAS_CREATED ─── PC_ADDED
   │
   └─── FAILED (retry ile tekrar denenebilir)
```

| Visa Alias Status | Anlam |
|---|---|
| `ACTIVE` | Lifecycle + Resolution çalışır |
| `DISABLED` | Resolution çalışmaz; tekrar ACTIVE yapılabilir |
| `BLOCKED` | Fraud/şüpheli — otomatik |
| `EXPIRED` | Süre aşımı — tekrar ACTIVE yapılabilir |
| `DELETED` | **Geri alınamaz** |

**Cascade kuralları:**
- Alias `DISABLED` → tüm credential'lar `DISABLED`
- Alias silinirse → tüm credential'lar silinir
- Alias `ACTIVE` → credential'lar da `ACTIVE`

## 1.7. Ödeme Aracı Zorunlu Alanları

**CARD:**
- `accountNumber`, `cardType`, `expirationDate` (`YYYY-MM`), `issuerName`, `nameOnCard`
- `billingAddress.country` (**zorunlu**, ISO-3166 Alpha-3)

**BANK:**
- `accountNumber`, `accountName`, `accountNumberType` (`IBAN`/`DEFAULT`/`TOKEN`)
- `bankName`, `countryCode`, `currencyCode`, `address.country`

## 1.8. Hata Kodları

| Kod | Anlam | Retry? |
|---|---|---|
| **400** | Validasyon hatası | ❌ |
| **404** | Kaynak yok | ❌ |
| **422** | İş kuralı ihlali | ❌ |
| **429** | Rate limit | ⚠️ `Retry-After` |
| **500/503** | Sunucu hatası | ⚠️ Backoff + jitter |
| **VAD-1000** | Data mismatch | ❌ |
| **VAD-1040** | Original alias bulunamadı | ⚠️ Race recovery |
| **VAD-1120/1130** | Internal error | ⚠️ Retry |
| **VAD-1260** | Default hesap silindi | ❌ |
| **CVC-30010** | Kart zaten ekli | ❌ |

---

# 2. Endpoint Kataloğu

## 2.1. Kullanacağımız Endpoint'ler

| Operasyon | Metot | Path | Yanıt |
|---|---|---|---|
| **Create Alias** | POST | `/aliasdirectory/v1/aliases` | 200 + `id`, `paymentCredentials[]` |
| **Get Alias** | GET | `/aliasdirectory/v1/aliases/{aliasId}` | 200 |
| **Update Alias** | PUT | `/aliasdirectory/v1/aliases/{aliasId}` | **204** (tam replace) |
| **Delete Alias** | DELETE | `/aliasdirectory/v1/aliases/{aliasId}` | **204** (cascade) |
| **Update Alias Status** | PUT | `/aliasdirectory/v1/aliases/{aliasId}/status` | **204** |
| **Find Alias ID by Value** | POST | `/aliasdirectory/v1/aliases/aliasId` | 200 + `id` (404 = yok) |
| **Get by External ID** | POST | `/aliasdirectory/v1/aliases/getByExternalId` | 200 |
| **Get All Payment Credentials** | GET | `/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials` | 200 |
| **Create Payment Credential** | POST | `/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials` | 200 |
| **Get Payment Credential** | GET | `/aliasdirectory/v1/paymentCredentials/{id}` | 200 |
| **Update Payment Credential** | PUT | `/aliasdirectory/v1/paymentCredentials/{id}` | **204** |
| **Delete Payment Credential** | DELETE | `/aliasdirectory/v1/paymentCredentials/{id}` | **204** |
| **Alias Resolve** | POST | `/aliasdirectory/v1/aliases/resolve` | 200 |
| **Alias Inquiry** | POST | `/aliasdirectory/v1/aliases/inquiry` | 200 |

**KULLANMADIKLARIMIZ:** Alias Metadata, Batch, Reports, Webhooks.

## 2.2. Path Parametreleri

| Parametre | Nereden Alınır | Kullanıldığı Yerler |
|---|---|---|
| **`aliasId`** | Create Alias `id` / `findAliasIdByValue` | Get/Update/Delete Alias, Create/GetAll PC |
| **`paymentCredentialId`** | Create PC `id` / `getAllPaymentCredentials` | Get/Update/Delete PC |

> `aliasId` ile `paymentCredentialId` **karıştırılmamalı**.

## 2.3. Sık Kullanılan Header'lar

| Header | Nerede | Açıklama |
|---|---|---|
| `X-Correlation-Id` | Tüm isteklerde | İzleme UUID (Spring Boot üretir) |
| `Authorization: Basic ...` | Tüm isteklerde | Basic Auth (Spring Boot üretir, DataPower pass eder) |
| `keyId` | MLE zarfında (`iat` ile) | MLE anahtar ID'si |

---

# 3. İş Kuralları

## 3.1. Create Alias — Her Tipten En Fazla 1

Create Alias'ta her tipten **en fazla bir araç**:
- ✅ 1 kart + 1 banka
- ❌ 2 kart, ❌ 2 banka

İkinci+ için **Create Payment Credential**.

## 3.2. CEMEA/EU İsim Kuralı

`nameOnCard` ve `accountName` için **soyadının ilk harfi + nokta**:
- "John Smith" → `"John S."`
- "Ayşe Yılmaz" → `"Ayşe Y."`

İhlal → 422.

## 3.3. PUT = Tam Replace

`PUT /paymentCredentials/{id}` ve `PUT /aliases/{id}` **tam replace**. Gönderilmeyen alanlar silinir. **Önce GET, sonra PUT.**

## 3.4. Cascade Silme

- `DELETE /aliases/{aliasId}` → tüm credential'lar silinir
- `DELETE /paymentCredentials/{id}` → sadece o credential

## 3.5. Status Cascade

- Alias `DISABLED` → tüm credential'lar `DISABLED`
- Alias `ACTIVE` → credential'lar `ACTIVE`

## 3.6. Idempotency — `externalId`

- Mobil her istek için **UUID `requestId`** üretir
- DB'de `request_id` (unique), Visa'da `externalId` olarak gönderilir
- Aynı `requestId` tekrar → `ALREADY_PROCESSED`

## 3.7. Race Condition Recovery

```
1. Lookup (findAliasIdByValue)
   ├── 200 → aliasId al → Create Payment Credential
   └── 404 → Create Alias'ı dene
              ├── 200 → başarılı
              └── VAD-1040/VAD-1000 → Lookup tekrar → Create Payment Credential
```

## 3.8. `userDetails.userName` PII İçermez

Anonim ID kullan (`usr_12345`). KVKK/GDPR uyumu için.

---

# 4. Mimari ve Orkestrasyon

## 4.1. DataPower ile Mimari

```
[Mobil İstek]
     │
     ▼
[Spring Boot Controller]
     │
     ▼
[MobileAliasOrchestrationService]
     │
     ├─ 1. DB'de requestId var mı? → EVET → idempotent
     ├─ 2. PENDING kaydet
     ├─ 3. DB'de visa_alias_id var mı?
     │       ├─ VAR → Create PC
     │       └─ YOK → Create Alias
     │
     ▼
[VisaAliasClient]
  ├─ mTLS (Visa client cert → DataPower'a sunulur)
  ├─ Basic Auth header (Visa username/password)
  ├─ MLE şifreleme (encData)
  └─ HTTP POST → DataPower URL
     │
     ▼
[DataPower API Gateway]
  ├─ /visa-alias prefix strip
  ├─ Backend mTLS (Visa'ya)
  ├─ Basic Auth PASS (değiştirmez)
  ├─ encData PASS (deşifre etmez)
  └─ Forward → Visa
     │
     ▼
[Visa Alias Directory]
  ├─ Basic Auth doğrular
  ├─ encData çözer (MLE decrypt)
  ├─ İşlemi yapar
  ├─ Yanıtı şifreler (MLE encrypt → encData)
  └─ Döndürür
     │
     ▼
[DataPower] → encData PASS → Spring Boot
     │
     ▼
[MLE decrypt] → iş mantığı → yanıt → Mobil
```

## 4.2. Katmanlar

```
controller/  → HTTP, doğrulama, status eşleme
service/     → İş mantığı, transaction, orkestrasyon
client/      → Visa HTTP çağrıları (mTLS + Basic + MLE)
repository/  → DB
domain/      → Entity'ler
dto/         → Visa API modelleri
web/dto/     → Mobil DTO'lar
```

**Kurallar:**
- Controller **asla** `VisaAliasClient`'ı doğrudan çağırmaz
- Client **asla** DB'ye yazmaz
- Entity **asla** Controller'a dönmez
- Controller'da `@Transactional` **olmaz**
- Visa API modeli ile domain model **ayrı**

## 4.3. Model Ayrımı

```java
// 1) Controller katmanı (mobil)
record MobileAliasRequest(...) {}
record MobileAliasResponse(...) {}

// 2) Visa API katmanı
class CreateAliasRequest { ... }
class CreateAliasResponse { ... }

// 3) Entity katmanı (DB)
@Entity class VisaAliasRecord { ... }
```

---

# BÖLÜM B — SPRING BOOT İMPLEMENTASYONU

# 5. Adım 0: Ön Gereksinimler

## 5.1. Visa Developer Erişimi
- [ ] [Visa Developer](https://developer.visa.com) hesabı
- [ ] **Alias Directory Service** ürünü projede
- [ ] Sandbox kimlik bilgileri

## 5.2. İndirilmesi Gerekenler

| Dosya | Nereden | Nerede Kullanılacak |
|---|---|---|
| `visa_keystore.jks` (veya `.p12`) | Two-Way SSL | mTLS (Spring Boot'ta) |
| `VisaRootCA.pem`, `VisaIntermediateCA.pem` | Two-Way SSL | Trust material |
| `visa_server_cert.pem` | MLE | Server public cert (MLE encrypt) |
| MLE private key `.pem` | MLE | Client private key (MLE decrypt) |
| `keyId` | MLE | MLE header |
| `username` / `password` | Two-Way SSL | Basic Auth |

> ⚠️ **Bu dosyalar Spring Boot'ta KALIR.** DataPower'a taşınmıyor (MLE Spring Boot'ta).

## 5.3. Yazılım Ortamı
- Java 17+
- Spring Boot 3.x
- Maven/Gradle
- PostgreSQL/Oracle/MySQL
- Postman/Insomnia
- IntelliJ IDEA / Cursor

## 5.4. DataPower Ekibiyle Netleştirilecekler

- [ ] **Front-side base path:** `/visa-alias` (strip edilecek)
- [ ] **Backend URL:** `https://api.visa.com` (veya sandbox/cert)
- [ ] **Path mapping:** `/visa-alias/aliasdirectory/v1/*` → `/aliasdirectory/v1/*`
- [ ] **Front-side TLS:** Spring Boot'un Visa client cert'ini kabul et
- [ ] **Back-side TLS:** Visa'ya mTLS
- [ ] **Basic Auth:** Pass-through (değiştirme)
- [ ] **encData:** Pass-through (deşifre etme)
- [ ] **Hata yanıtları:** Pass-through (formatı bozma)

---

# 6. Adım 1: Proje İskeleti

## 6.1. Paket Yapısı

```
com.banka.visa.alias/
├── config/
│   ├── VisaProperties.java
│   └── RestTemplateConfig.java
├── client/
│   ├── VisaApiClient.java      # Base (Basic Auth header)
│   ├── MleApiClient.java       # MLE'li
│   └── VisaAliasClient.java    # Domain
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
│   └── GlobalExceptionHandler.java
└── util/
    ├── SensitiveDataHasher.java
    └── AliasNormalizer.java
```

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
    <!-- MLE için -->
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

## 6.3. `.gitignore` (Zorunlu)

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
- [ ] `mvn clean compile` hatasız
- [ ] `.env`, `*.jks`, `*.pem` `.gitignore`'da

---

# 7. Adım 2: Konfigürasyon

## 7.1. `application.yml` — **DataPower Base URI**

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
  # ⭐ DEĞİŞEN TEK ALAN — DataPower'a işaret ediyor
  base-uri: ${DATAPOWER_BASE_URI:https://datapower.banka.local/visa-alias}

  # AŞAĞIDAKİLER AYNI KALIYOR
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

> ⚠️ **`base-uri` sonunda `/` OLMASIN.** `/visa-alias/` → `/visa-alias//aliasdirectory/...` (çift slash) oluşur, bazı sunucular 404 döner.

## 7.2. `.env.example`

```bash
DATAPOWER_BASE_URI=https://datapower.banka.local/visa-alias
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

## 7.3. `VisaProperties`

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

Ana sınıfta `@ConfigurationPropertiesScan` veya `@EnableConfigurationProperties(VisaProperties.class)`.

## 7.4. Sır Yönetimi

**Yasak:**
- Şifre, keystore şifresi, MLE key koda gömme
- `application.yml`'de gerçek sır

**Zorunlu:**
- Tüm sırlar **ortam değişkeni** veya **Vault/K8s Secrets**
- `.env.example` ile placeholder

## ✅ Doğrulama
- [ ] `VisaProperties` bind ediliyor
- [ ] `visa.base-uri` DataPower'ı gösteriyor
- [ ] `git grep "VISA_PASSWORD="` → sadece `.env.example`

---

# 8. Adım 3: Sertifika ve MLE Anahtarları

## 8.1. Sertifika Yerleşimi (Spring Boot'ta)

```
/etc/visa/
├── keystore.jks              (600, app user)  ← mTLS
├── server-cert.pem           (644)            ← MLE encrypt (Visa'nın public key'i)
├── client-key.pem            (600, app user)  ← MLE decrypt (bizim private key)
└── key-id.txt                (644)            ← MLE header
```

> **DataPower'a taşınmıyor.** MLE ve Basic Auth Spring Boot'ta.

## 8.2. Keystore İçeriği

`visa_keystore.jks` içermeli:
- **Client private key + cert** (Visa'dan indirilen)
- **Visa CA'ları** (trust material, DataPower'ı doğrulamak için — front-side TLS)

**Seçenek A (önerilen):** DataPower, Spring Boot'un sunduğu Visa client cert'ini kabul eder. Keystore'da değişiklik yok.

**Seçenek B:** Spring Boot, banka iç CA'sından cert kullanır. Keystore değişir; DataPower banka CA'sını kabul eder, Visa'ya kendi cert'i ile gider.

> **Tercih: Seçenek A** — DataPower ekibi truststore'a Visa CA'sını ekler.

## 8.3. MLE Anahtarları

- **Server public cert** (`server-cert.pem`): Visa'nın public key'i — MLE **encrypt** için
- **Client private key** (`client-key.pem`): Bizim private key'imiz — MLE **decrypt** için

**Kritik:** Bu iki dosya **Spring Boot'ta kalır**. DataPower MLE'ye dokunmaz.

## 8.4. Keystore Hazırlığı

```bash
# .p12 → JKS (gerekirse)
keytool -importkeystore \
  -srckeystore visa.p12 -srcstoretype PKCS12 \
  -destkeystore visa_keystore.jks -deststoretype JKS \
  -srcstorepass <P12_PASS> -deststorepass <JKS_PASS>

# Visa CA'larını trust olarak ekle
keytool -import -alias visa-root -keystore visa_keystore.jks \
  -file VisaRootCA.pem -storepass <PASS>
keytool -import -alias visa-intermediate -keystore visa_keystore.jks \
  -file VisaIntermediateCA.pem -storepass <PASS>
```

## ✅ Doğrulama
- [ ] `keytool -list -keystore visa_keystore.jks` → 1 private key + CA'lar
- [ ] `openssl x509 -in server-cert.pem -text -noout` → geçerli
- [ ] `openssl rsa -in client-key.pem -check` → geçerli

---

# 9. Adım 4: SSL/RestTemplate (mTLS)

**Değişmiyor** — mTLS Spring Boot'ta, hedef DataPower.

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
                .loadTrustMaterial(keyStore, null)   // ← Visa CA'ları (DataPower'ı doğrular)
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

        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
    }
}
```

> **Not:** `loadTrustMaterial`'da Visa CA'ları var. DataPower'ın sunucusu Visa'nın CA'sıyla mı imzalı? **Değilse**, banka iç CA'sını da trust material'a ekleyin (Seçenek B).

## 9.2. `VisaApiClient` (Basic Auth header)

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

        // Basic Auth header (Visa username/password)
        String auth = props.getUsername() + ":" + props.getPassword();
        headers.add(HttpHeaders.AUTHORIZATION, "Basic " +
                Base64.getEncoder().encodeToString(auth.getBytes(StandardCharsets.UTF_8)));
        return headers;
    }
}
```

## ✅ Doğrulama
- [ ] DataPower'a mTLS ile bağlanabiliyor (`curl --cert ... https://datapower.../visa-alias/...`)
- [ ] Basic Auth header'ı gönderiliyor (DataPower log'unda görünüyor)
- [ ] Timeout'lar makul (10s/30s)

---

# 10. Adım 5: MLE Katmanı

**Değişmiyor** — MLE Spring Boot'ta.

## 10.1. `MleEncryptionUtil`

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
        return new ObjectMapper().readValue(jwe.getPayload().toString(), type);
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

## 10.2. `MleApiClient`

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
            // 1. MLE şifrele (Spring Boot)
            String encryptedBody = body == null ? null : mle.encrypt(body);

            // 2. DataPower URL'ine POST
            String url = props.getBaseUri() + path;
            HttpEntity<Object> entity = new HttpEntity<>(encryptedBody, buildHeaders());

            ResponseEntity<String> resp = restTemplate.exchange(
                    url, method, entity, String.class);

            if (resp.getBody() == null || resp.getBody().isBlank()) {
                return null;  // 204 No Content
            }

            // 3. MLE çöz (Spring Boot)
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
- [ ] DataPower üzerinden `POST /aliases` çağrısı 200 dönüyor
- [ ] `encData` yanıtı decrypt ediliyor
- [ ] Private key loglarda görünmüyor

---

# 11. Adım 6: Visa API Modelleri

**Değişmiyor.** Önceki rehberdeki DTO'lar aynen.

## 11.1. İstek Modelleri

```java
// CreateAliasRequest
@Getter @Setter
public class CreateAliasRequest {
    @NotNull private String aliasType;
    @NotNull private String aliasValue;
    private String externalId;
    @Valid @NotNull private Consent consent;
    @Valid private Profile profile;
    @Valid @NotNull private List<PaymentCredential> paymentCredentials;
}

// Consent
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

// Profile
@Getter @Setter
public class Profile {
    @NotNull private String firstName;
    @NotNull private String lastName;
    private String middleName;
    private String dateOfBirth;
    private List<ContactInfo> contactInfo;
}

// PaymentCredential (polymorphic)
@Getter @Setter
@JsonTypeInfo(use = NAME, include = EXISTING_PROPERTY, property = "type", visible = true)
@JsonSubTypes({
    @JsonSubTypes.Type(value = CardCredential.class, name = "CARD"),
    @JsonSubTypes.Type(value = BankCredential.class, name = "BANK")
})
public abstract class PaymentCredential {
    @NotNull private String type;
}

// CardCredential
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

// BillingAddress
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

---

# 12. Adım 7: Visa Client

## 12.1. `VisaAliasClient` — `BASE` **Değişmiyor**

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class VisaAliasClient {

    // ⭐ DEĞİŞMİYOR — Visa path'i aynı. DataPower /visa-alias'ı strip edecek.
    private static final String BASE = "/aliasdirectory/v1";

    private final MleApiClient client;

    public CreateAliasResponse createAlias(CreateAliasRequest request) {
        log.info("createAlias başladı externalId={}", request.getExternalId());
        return client.executeMle(HttpMethod.POST, BASE + "/aliases",
                request, CreateAliasResponse.class);
    }

    public PaymentCredentialSummary createPaymentCredential(
            String aliasId, PaymentCredential request) {
        log.info("createPaymentCredential başladı aliasId={}", aliasId);
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

## 12.2. Oluşan Birleşik URL'ler

| Çağrı | Spring Boot'un çağırdığı URL |
|---|---|
| `createAlias` | `https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases` |
| `createPaymentCredential` | `https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials` |
| `findAliasIdByValue` | `https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases/aliasId` |
| `deleteAlias` | `https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases/{aliasId}` |

DataPower bunları alıp `/visa-alias` prefix'ini strip ederek Visa'ya iletir:
```
https://api.visa.com/aliasdirectory/v1/aliases
https://api.visa.com/aliasdirectory/v1/aliases/{aliasId}/paymentCredentials
...
```

## ✅ Doğrulama
- [ ] DataPower üzerinden her metot manuel test edildi
- [ ] `X-Correlation-Id` header'da
- [ ] `Authorization: Basic ...` header'da
- [ ] `findAliasIdByValue` 404 → `null`

---

# 13. Adım 8: Veritabanı

**Değişmiyor.**

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

## 13.4. `SensitiveDataHasher`

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

---

# 14. Adım 9: Orkestrasyon

**Değişmiyor.**

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
- [ ] Yeni alias → `ALIAS_CREATED`
- [ ] Mevcut alias'a kart → `PC_ADDED`
- [ ] Aynı requestId → `ALREADY_PROCESSED`
- [ ] `VAD-1040` → recovery
- [ ] PAN loglarda yok

---

# 15. Adım 10: Controller ve DTO'lar

**Değişmiyor.**

## 15.1. `MobileAliasRequest`

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

## 15.2. `MobileAliasResponse`

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

---

# 16. Adım 11: Hata Yönetimi

**Değişmiyor.**

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

## 16.2. `GlobalExceptionHandler`

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

---

# 17. Adım 12: Test

## 17.1. Unit Test

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
        // given: db empty, visa returns success
        // when: service.handle(req)
        // then: status = ALIAS_CREATED, verify visaClient.createAlias(any()) called once
    }

    @Test
    void givenSameRequestId_whenHandle_thenAlreadyProcessed() {
        // given: db returns existing record with ALIAS_CREATED
        // when: service.handle(req)
        // then: status = ALREADY_PROCESSED, verifyNoInteractions(visaClient)
    }

    @Test
    void givenAliasAlreadyExists_whenCreateAlias_thenRecovers() {
        // given: visa.createAlias throws VAD-1040, findAliasIdByValue returns "alias-1",
        //        createPaymentCredential returns "pc-2"
        // when: service.handle(req)
        // then: status = PAYMENT_CREDENTIAL_ADDED
    }

    @Test
    void givenVisaServerError_whenHandle_thenFailed() {
        // given: visa.createAlias throws 500
        // when: service.handle(req)
        // then: status = FAILED
    }
}
```

## 17.2. Integration Test (WireMock — DataPower simülasyonu)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@AutoConfigureMockMvc
class MobileAliasControllerIT {

    @Autowired MockMvc mockMvc;
    @RegisterExtension
    static WireMockExtension dataPower = WireMockExtension.newInstance().build();

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        // WireMock'u DataPower gibi davranmaya ayarla
        r.add("visa.base-uri", () -> dataPower.baseUrl() + "/visa-alias");
    }

    @Test
    void whenValidRequest_thenCreated() throws Exception {
        // DataPower, gelen /visa-alias/... isteğini /aliasdirectory/v1/... gibi yanıtlar
        dataPower.stubFor(post("/visa-alias/aliasdirectory/v1/aliases")
                .willReturn(okJson("{\"encData\":\"...\"}")));  // MLE'li yanıt simülasyonu

        mockMvc.perform(post("/api/mobile/aliases")
                .contentType(APPLICATION_JSON)
                .header("X-Api-Key", "test")
                .content(/* JSON */))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.status").value("ALIAS_CREATED"));
    }
}
```

> **Not:** Gerçek MLE test için encrypt/decrypt'i de simüle etmeniz gerekebilir. Alternatif: `MleEncryptionUtil`'ı test için mock'layın.

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

1. ✅ Yeni alias → `ALIAS_CREATED`
2. ✅ Mevcut alias'a kart → `PC_ADDED`
3. ✅ Aynı `requestId` iki kez → `ALREADY_PROCESSED`
4. ✅ `VAD-1040` race recovery
5. ✅ 5xx retry
6. ✅ 4xx `FAILED`
7. ✅ PAN loglanmıyor

---

# BÖLÜM C — DATAPOWER YAPILANDIRMASI

# 18. DataPower Path Mapping

## 18.1. Gelen / Giden Path

```
GELEN:  https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases
                                          └────┬────┘ └────────────┬─────────────┘
                                        STRIP                    VISA'YA GİDEN

GİDEN:  https://api.visa.com/aliasdirectory/v1/aliases
```

## 18.2. DataPower MPGW / API Gateway Kuralı

| Ayar | Değer |
|---|---|
| **Front-side base path** | `/visa-alias` (strip edilecek) |
| **Backend URL** | `https://api.visa.com` (veya sandbox/cert) |
| **Backend path** | Gelen path'ten `/visa-alias` çıkarılmış hali (`/aliasdirectory/v1/...`) |
| **Front-side TLS** | Spring Boot'un Visa client cert'ini kabul et (truststore'a Visa CA ekle) |
| **Back-side TLS** | Visa'ya mTLS (DataPower'da Visa client cert) |
| **Basic Auth** | **PASS-THROUGH** (değiştirme, silme) |
| **encData (body)** | **PASS-THROUGH** (deşifre etme, yeniden şifreleme) |
| **X-Correlation-Id** | PASS-THROUGH |
| **Response** | Visa'dan gelen `{"encData":"..."}` **olduğu gibi** Spring Boot'a |
| **Hata** | `{ "errorCode": "...", "errorMessages": [...] }` formatını **bozma**, status kodunu aynen dön |

## 18.3. Kritik Pass-Through Listesi

DataPower'ın **dokunmaması gerekenler:**

- ✅ `Authorization: Basic ...` header'ı → Visa'ya aynen
- ✅ `{"encData":"..."}` request body → Visa'ya aynen
- ✅ `{"encData":"..."}` response body → Spring Boot'a aynen
- ✅ `X-Correlation-Id` header → aynen
- ✅ Visa hata formatı → aynen
- ✅ HTTP status kodları (200/204/400/404/422/500) → aynen

> ⚠️ **DataPower'da JWE Encrypt/Decrypt action KULLANMAYIN.** MLE tamamen Spring Boot'ta. DataPower'da MLE varsa çift şifreleme olur, Visa çözemez.

---

# 19. DataPower'ın Yapması Gerekenler

## 19.1. Front-Side TLS Profile (mTLS)

```
DataPower → TLS Profiles → TLS Client Profile
├── Name: visa-front-mtls
├── Protocol: TLS 1.2+
├── Trust Store: bank-truststore (Visa CA'ları ekli)
└── Client Authentication: Required
```

Spring Boot, DataPower'a Visa client cert'ini sunar; DataPower bu cert'i Visa CA'sı ile doğrular.

## 19.2. Back-Side TLS Profile (mTLS to Visa)

```
DataPower → TLS Profiles → TLS Client Profile
├── Name: visa-back-mtls
├── Key Store: datapower-visa-keystore (Visa client cert)
├── Trust Store: datapower-visa-truststore (Visa CA'ları)
└── SNI: sandbox.api.visa.com (veya api.visa.com)
```

## 19.3. AAA Policy

Spring Boot'tan gelen Basic Auth header'ını **pass-through** yap. İki seçenek:

**Seçenek A (basit):** AAA policy yok; header aynen Visa'ya gider.
**Seçenek B (kontrol):** AAA policy ile Basic Auth'u doğrula, sonra Visa'ya ilet.

**Öneri:** Seçenek A — pass-through. Visa zaten doğrulamayı yapar.

## 19.4. MLE Action

**YOK.** DataPower'da JWE/JOSE action olmayacak. MLE tamamen Spring Boot'ta.

## 19.5. DataPower Ekibine İletilecek Özet

```
Visa Alias Directory — DataPower Yapılandırma Talebi

Mimari:
  Spring Boot → DataPower → Visa
  Spring Boot: mTLS (Visa client cert) + Basic Auth + MLE (JWE)
  DataPower:   yönlendirici (MLE'ye dokunmaz)

Front-side:
  Base path: /visa-alias
  mTLS: Spring Boot'un sunduğu Visa client cert'ini kabul et
  (truststore'a Visa CA ekle)

Back-side:
  Target: https://api.visa.com (veya sandbox/cert)
  Path: /visa-alias prefix strip → /aliasdirectory/v1/...
  mTLS: DataPower'da Visa client cert

Pass-Through (DEĞİŞTİRME):
  - Authorization: Basic ... header
  - {"encData":"..."} request body
  - {"encData":"..."} response body
  - X-Correlation-Id header
  - Visa hata formatı: {"errorCode":"...","errorMessages":[...]}
  - HTTP status kodları

KRİTİK: MLE için DataPower'da JWE Encrypt/Decrypt action KULLANMAYIN.
MLE tamamen Spring Boot tarafında yapılıyor.
```

---

# 20. Test ve Doğrulama

## 20.1. Path Mapping Testi

```bash
# Spring Boot'un DataPower'a atacağı URL'i manuel test et
curl -X POST "https://datapower.banka.local/visa-alias/aliasdirectory/v1/aliases" \
  -H "Content-Type: application/json" \
  -H "Authorization: Basic $(echo -n 'USER:PASS' | base64)" \
  -H "X-Correlation-Id: test-123" \
  --cert visa-client-cert.pem --key visa-client-key.pem \
  --cacert datapower-ca.pem \
  -d '{"encData":"<JWE>"}'
```

**Beklenen:** Visa'nın yanıtı DataPower'dan **olduğu gibi** döner.

## 20.2. Kontrol Listesi

- [ ] `curl` → 200/204/4xx dönüyor (404 değil — path mapping yanlış)
- [ ] `Authorization` header Visa'ya ulaşıyor (DataPower log'unda görün)
- [ ] `encData` değişmeden Visa'ya gidiyor
- [ ] Visa `encData` yanıtı değişmeden Spring Boot'a dönüyor
- [ ] Visa hata formatı bozulmuyor
- [ ] DataPower'da MLE action yok

## 20.3. E2E Test

1. Spring Boot'u çalıştır (Sandbox DataPower URL'i ile)
2. Postman ile `POST /api/mobile/aliases` çağır
3. Akış: Spring Boot → mTLS → DataPower → Basic Auth + encData pass → Visa → MLE decrypt → yanıt
4. Yanıt `ALIAS_CREATED` olmalı

---

# BÖLÜM D — STANDARTLAR VE OPERASYON

# 21. Kodlama Standartları

## 21.1. Teknoloji

- **Java 17+** (record, sealed class, pattern matching)
- **Spring Boot 3.x**
- **Jakarta EE** namespace
- **Lombok**: `@Getter`, `@Builder`, `@RequiredArgsConstructor`; `@Data` **kullanma**

## 21.2. Naming

| Tür | Kural | Örnek |
|---|---|---|
| Controller | `*Controller` | `MobileAliasController` |
| Service | `*Service` | `MobileAliasOrchestrationService` |
| Visa Client | `Visa*Client` | `VisaAliasClient` |
| Request DTO | `*Request` | `CreateAliasRequest` |
| Response DTO | `*Response` | `MobileAliasResponse` |
| Entity | Tekil | `VisaAliasRecord` |
| Repository | `*Repository` | `VisaAliasRecordRepository` |
| Exception | `*Exception` | `VisaClientException` |

## 21.3. Validation

- Jakarta Bean Validation (`@NotNull`, `@NotBlank`, `@Pattern`)
- Controller'da `@Valid`
- İş kuralı validasyonları Service'te

## 21.4. Logging

**Yasak:** PAN, CVV, şifre, token, private key, Visa kimlik bilgileri.
**Zorunlu:** `requestId`, `X-Correlation-Id`, operation adı.

```java
log.info("Visa alias create başladı requestId={} aliasType={}",
         requestId, aliasType);
log.debug("Kart son 4 hane: {}",
    accountNumber.substring(accountNumber.length() - 4));
```

---

# 22. Güvenlik Kuralları

## 22.1. MLE — ZORUNLU

- Hassas veri içeren **tüm** Visa çağrılarında MLE
- `withMLERequest(true)` **her zaman** true
- Algoritma: **RSA-OAEP-256 + A128GCM**
- İki çift anahtar: Visa server public cert + bizim client private key

## 22.2. mTLS

- `RestTemplate` bir keystore ile yapılandırılır
- Keystore yolu ve şifresi **ortam değişkeni**
- **Asla** git'e commit edilmez

## 22.3. Sır Yönetimi

**Yasak:**
- API key, şifre, keystore şifresi koda gömme
- `application.yml`'de gerçek sır
- Loglarda sır gösterme

**Zorunlu:**
- Tüm sırlar **Vault/K8s Secrets**
- `.env.example` ile placeholder

## 22.4. PCI-DSS / KVKK / GDPR

- PAN **asla** DB'ye yazılmaz → hash + son 4
- `userDetails.userName` PII içermez
- Silme talebi → `deleteAlias` + DB hard delete

## 22.5. Güvenlik Kontrol Listesi

**Sır/Sertifika:**
- [ ] Şifreler Vault/K8s Secrets
- [ ] `application.yml`'de placeholder
- [ ] `.env`, `*.jks`, `*.pem`, `*.key` `.gitignore`'da
- [ ] `git grep -iE "(password|secret|BEGIN.*PRIVATE)"` temiz

**Veri:**
- [ ] PAN DB'de sadece hash + son 4
- [ ] PAN loglarda yok
- [ ] Exception'larda PAN yok

**API:**
- [ ] Mobil endpoint'te `X-Api-Key` zorunlu
- [ ] Visa çağrıları HTTPS + mTLS
- [ ] MLE aktif
- [ ] `X-Correlation-Id` üretiliyor

**Uyumluluk:**
- [ ] KVKK/GDPR silme → `deleteAlias` + DB hard delete
- [ ] Saklama süreleri tanımlı

---

# 23. Yasaklar ve Kontrol Listesi

## 23.1. Yasaklar

1. ❌ PAN'ı loglama
2. ❌ Sırları koda gömme
3. ❌ `withMLERequest(false)`
4. ❌ Controller'da `@Transactional`
5. ❌ Entity'yi mobilize dönme
6. ❌ `e.printStackTrace()`
7. ❌ Boş `catch (Exception e) {}`
8. ❌ Hard-coded URL/keyId
9. ❌ `@Data` entity'lerde
10. ❌ Retry'siz 5xx
11. ❌ Idempotency'siz POST
12. ❌ `Thread.sleep()` retry
13. ❌ Test edilmemiş hata yolu
14. ❌ `PUT /paymentCredentials/{id}` öncesi GET yapmama
15. ❌ Alias `DISABLED` iken `resolve`
16. ❌ `+` ile telefon numarası
17. ❌ **DataPower'da MLE action açmak** (çift şifreleme)
18. ❌ **DataPower'da Basic Auth header'ını silmek/değiştirmek**
19. ❌ **`base-uri` sonuna `/` eklemek** (çift slash → 404)

## 23.2. Her Kod Önerisinde Kontrol

**Fonksiyonel:**
- [ ] Alias var/yok DB'den kontrol
- [ ] `requestId` idempotency
- [ ] `VAD-1040` recovery
- [ ] State machine doğru
- [ ] `MobileAliasResponse` dönüyor

**Güvenlik:**
- [ ] `withMLERequest(true)`
- [ ] PAN loglanmıyor
- [ ] Sırlar env'den
- [ ] `userDetails.userName` PII değil

**Hata Yönetimi:**
- [ ] 4xx → retry yok, `FAILED`
- [ ] 5xx/timeout → retry + backoff
- [ ] Merkezi `@ControllerAdvice`
- [ ] Exception'da sır/PAN yok

**Kod Kalitesi:**
- [ ] Katmanlar doğru
- [ ] DTO/Entity/API model ayrımı
- [ ] `@Valid`
- [ ] Test yazıldı
- [ ] Naming convention

**DataPower:**
- [ ] `base-uri` sonunda `/` yok
- [ ] `VisaAliasClient.BASE = "/aliasdirectory/v1"`
- [ ] DataPower path mapping doğru
- [ ] DataPower'da MLE action yok
- [ ] Basic Auth pass-through

## 23.3. SSS

**S: Mobil `+905551234567` gönderirse?**
C: `normalizeAliasValue()` ile `+` temizle.

**S: Alias `DISABLED` ise?**
C: Kart eklenebilir ama `resolve` çalışmaz. İş kararı; logla, devam.

**S: Aynı kart 2 kez?**
C: Visa `CVC-30010` → `FAILED`, mobilize "kart zaten ekli".

**S: Visa 500 döndü?**
C: Exponential backoff + jitter (max 3), aynı `requestId`.

**S: `externalId` zorunlu mu?**
C: Teknik olarak hayır, ama idempotency için zorunlu kabul et.

**S: `PUT /paymentCredentials/{id}` tehlikeli mi?**
C: Tam replace. Önce GET, sonra PUT.

**S: DataPower'da MLE yapmalı mıyım?**
C: **Hayır.** MLE tamamen Spring Boot'ta. DataPower sadece pass-through.

**S: DataPower Basic Auth header'ını değiştirmeli mi?**
C: **Hayır.** Pass-through. Visa kendi doğrular.

**S: 404 alıyorum (DataPower'dan)?**
C: Path mapping yanlış. DataPower `/visa-alias` prefix'ini strip etmiyor olabilir.

**S: `base-uri` sonuna `/` koymalı mıyım?**
C: **Hayır.** Çift slash → 404.

---

# 24. Canlıya Geçiş

## 24.1. Sandbox → Certification

- [ ] Test case'leri geçti
- [ ] Certification başvurusu
- [ ] Cert kimlik bilgileri ve sertifikalar
- [ ] DataPower cert backend URL güncellendi
- [ ] `DATAPOWER_BASE_URI` cert ortamına

## 24.2. Certification → Production

- [ ] Sertifikasyon test case'leri tamam
- [ ] Sözleşme imzalandı
- [ ] Production kimlik bilgileri
- [ ] DataPower production backend URL
- [ ] Pilot faz (sınırlı kullanıcı)
- [ ] Monitoring:
  - Visa hata oranı > %1 → alarm
  - p99 latency > 3sn → alarm
  - `FAILED` sayısı günlük rapor
- [ ] Runbook (Visa/DataPower down)
- [ ] Müşteri destek bilgilendirildi

## 24.3. Post-Launch

- [ ] İlk 24 saat: hata logları
- [ ] İlk 1 hafta: performans
- [ ] İlk 1 ay: iş metrikleri
- [ ] Sertifika rotasyon takvimi

## 24.4. Tahmini Takvim

| Aşama | Süre |
|---|---|
| Geliştirme | 2-3 hafta |
| DataPower yapılandırma | 1-2 hafta (paralel) |
| Sandbox E2E | 2-3 gün |
| Certification | 1 hafta |
| Production + pilot | 2-4 hafta |
| **Toplam** | **5-8 hafta** |

---

# 25. Sorun Giderme

| Belirti | Olası Neden | Çözüm |
|---|---|---|
| `SSLHandshakeException` (DataPower'a) | Keystore yolu/şifresi yanlış | `keytool -list` ile doğrula |
| `SSLHandshakeException` (DataPower'dan) | DataPower Spring Boot cert'ini kabul etmiyor | Truststore'a Visa CA ekle (front-side) |
| **404 Not Found** (DataPower'dan) | `/visa-alias` strip edilmiyor | DataPower'da **front-side base path strip** aktif |
| **404 Not Found** (Visa'dan) | Path yanlış map | `/visa-alias` sonrası olduğu gibi forward |
| **401 Unauthorized** | Basic Auth header düşüyor | DataPower pass-through kontrol |
| **400 Bad Request** (`invalid encData`) | DataPower MLE decrypt/re-encrypt | DataPower'da **JWE action kaldır** |
| **Çift slash** `/visa-alias//aliasdirectory/...` | `base-uri` sonunda `/` | Slash'ı kaldır |
| `VAD-1000` | Alias zaten var | `findAliasIdByValue` → `createPaymentCredential` |
| `VAD-1040` | Original alias yok | Race recovery |
| `CVC-30010` | Kart zaten ekli | İş kararı |
| Timeout | Ağ/proxy | Timeout artır (10s/30s) |

## 25.1. Debug

- **`X-Correlation-Id`'yi logla** — Visa destek ve DataPower log korelasyonu
- **DataPower log'unda header ve body'yi gör** — Basic Auth ve encData pass mı?
- **Postman'de doğrudan Visa'ya at** — sorun Spring Boot'ta mı, DataPower'da mı?
- **Postman'de doğrudan DataPower'a at** — path mapping doğru mu?
- **`org.apache.hc.client5: DEBUG`** — SSL handshake detayları

---

# 26. Şirket Standartları

> Doldurulacak.

## 26.1. Mimari
- **Katman isimlendirme:** `[controller → facade → service → repository]`
- **DTO dönüşümü:** `[MapStruct]`
- **Transaction:** `[Service katmanı]`
- **Event:** `[Kafka/RabbitMQ]`
- **Caching:** `[Redis, key formatı]`

## 26.2. Kodlama
- **Paket yapısı:** `com.[şirket].[proje].[katman]`
- **Lombok:** `[izin verilen anotasyonlar]`
- **Log formatı:** `[JSON/MDC]`
- **Checkstyle:** `[kurallar]`
- **Record vs POJO:** `[hangi durumda]`
- **Null handling:** `[Optional/@Nullable]`

## 26.3. Güvenlik
- **Sır yönetimi:** `[Vault/K8s/AWS]`
- **Sertifika rotasyonu:** `[sıklık/otomasyon]`
- **Log maskeleme:** `[pattern]`
- **Güvenlik code review:** `[zorunlu]`

## 26.4. Hata Yönetimi
- **Hata formatı:** `[RFC 7807]`
- **Retry kütüphanesi:** `[Resilience4j/Spring Retry]`
- **DLQ:** `[başarısız mesajlar]`
- **Alerting:** `[Slack/PagerDuty]`

## 26.5. Test
- **Coverage:** `[%80]`
- **Test framework:** `[JUnit 5]`
- **Mock:** `[Mockito]`
- **WireMock/Testcontainers:** `[kullanılıyor]`

## 26.6. DataPower
- **Ortam URL'leri:** `[sandbox/cert/prod DataPower]`
- **Front-side TLS:** `[Seçenek A/B]`
- **Pass-through onayı:** `[yazılı teyit]`
- **Path mapping:** `/visa-alias/aliasdirectory/v1/*` → `/aliasdirectory/v1/*`

---

## 📌 Son Not

Bu dosya **yaşayan bir dokümandır.** Visa API, banka standartları veya DataPower yapılandırması değiştiğinde güncellenmelidir.

**Kritik özet:**
- **Spring Boot'ta değişen tek şey:** `visa.base-uri` → DataPower URL'i
- **Spring Boot'ta değişmeyen:** mTLS, Basic Auth, MLE, tüm iş mantığı
- **DataPower'ın rolü:** `/visa-alias` prefix strip + Visa'ya forward; MLE'ye ve Basic Auth'a **dokunmaz**
- **Path:** `https://datapower.banka.local/visa-alias/aliasdirectory/v1/...`
- **`base-uri` sonunda `/` YOK**

**Visa kaynakları:**
- API Reference: `https://developer.visa.com/api-reference`
- Getting Started: `https://developer.visa.com/docs/alias-directory-service/docs-how-to`
- Encryption Guide: `https://developer.visa.com/pages/encryption_guide`
- Destek: `developer@visa.com`