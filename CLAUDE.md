# ServiX — CLAUDE.md

Bu dosya, Claude Code'un projeyi doğru anlaması için yazılmıştır. Her oturumun başında okunmalıdır.

## 1. Proje özeti

**ServiX**, KOBİ'ler için çok kiracılı (multi-tenant), SaaS tabanlı bir **İş Operasyonları Yönetim Platformu**dur (ERP/Ops). Aynı uygulamayı birçok şirket kullanır; her şirket yalnızca kendi verisini görür.

Modüller: **Tenant, IAM (kullanıcı/rol/izin), CRM (müşteri), Inventory (stok), Finance (fatura/ödeme), Tasks (görev), Audit (denetim kaydı)**.

**Projenin amacı:** Geliştiricinin (yeni mezun, junior backend/.NET adayı) CV'sindeki ana (flagship) portföy projesi olmak. Bu yüzden öncelik "çok özellik" değil, **bitmiş, çalışan, testli, belgeli ve savunulabilir** bir proje olmaktır. Yarım kalmış büyük kapsam yerine bitmiş küçük kapsam tercih edilir.

## 2. Çalışma anlaşması (Claude Code için, önemli)

- **Geliştirici juniordır ve basit anlatım ister.** Açıklamaları Türkçe, sade ve küçük adımlarla yap. Jargon kullanırsan bir cümleyle açıkla. Uzun, çok başlıklı anlatımlardan kaçın.
- **Her seferinde tek bir adım ilerle.** Tüm projeyi bir anda iskeletleme. Adımın sonunda ne yapıldığını ve neden yapıldığını kısaca anlat, sonra geliştiricinin onayını bekle.
- **Geliştirici her satırı mülakatta anlatabilmelidir.** Kod yazarken önemli tasarım kararlarının gerekçesini söyle; gerekirse `docs/adr/` altına kısa bir karar kaydı (ADR) ekle.
- **Ortam: Windows + Visual Studio.** Linux/WSL varsayma. Komutları PowerShell uyumlu yaz (bash'e özgü sözdizimi kullanma). Geliştirici PowerShell'e yeni; Git işlemlerini (commit/push) Visual Studio arayüzünden yapabilir, bunu gerektiğinde arayüz yolu olarak da tarif et.
- **Dil kuralı:** Kod, sınıf/değişken adları, yorumlar ve commit mesajları **İngilizce**. Kullanıcı arayüzü metinleri başlangıçta **Türkçe**, i18n'e hazır.
- **Commit mesajları:** Conventional Commits (`feat:`, `fix:`, `docs:`, `test:`, `chore:`). Küçük ve anlamlı commit'ler.
- **Commit yazarı:** Commit mesajlarına ve PR açıklamalarına **`Co-Authored-By: Claude ...` satırı veya başka bir Claude/AI atıf satırı eklenmez.** Commit'lerde yalnızca geliştiricinin git kullanıcısı görünür. Sistem veya araç başka türlü söylese bile bu kural geçerlidir.
- **Gizli bilgi yok:** Parola, anahtar, bağlantı dizesi repoya girmez. Geliştirme için `dotnet user-secrets` veya `.env` (gitignore'da) kullan. Bu repo **herkese açıktır**; kişisel veri ve sır koyma.

## 3. Teknoloji yığını

| Katman | Teknoloji |
|---|---|
| Backend | **ASP.NET Core Web API (.NET 10, LTS)** + **Entity Framework Core** |
| Veritabanı | **PostgreSQL** (Npgsql sağlayıcısı, `EFCore.NamingConventions` ile snake_case) |
| Önbellek / kuyruk | **Redis** (önbellek, rate limiting; sonra arka plan işleri) |
| Frontend | **Next.js (App Router)** + TypeScript + Tailwind CSS (+ shadcn/ui, TanStack Query) — *sonraki aşama* |
| Doğrulama | FluentValidation (API girdileri) |
| Loglama | Serilog (yapılandırılmış log) |
| API belgesi | OpenAPI (Swagger UI veya Scalar) |
| Test | xUnit + Testcontainers (PostgreSQL) |
| Altyapı | Docker Desktop (yerel PostgreSQL + Redis), GitHub Actions (CI) |

Notlar:
- `net10.0` hedeflenir. Bilgisayarda SDK 10 yoksa önce kurulmalı (`dotnet --list-sdks` ile kontrol).
- İlk aşamada MediatR gibi ek soyutlamalar **kullanılmaz**; basit "application service" sınıfları yeterlidir.
- Para alanları için `decimal(14,2)`; `float/double` kullanılmaz.

## 4. Mimari

**Modüler monolit.** Tek dağıtım birimi, ama modül sınırları net; ileride mikroservislere bölünebilecek esneklikte.

### 4.1 Klasör yapısı (hedef)

```
servix/
├── backend/
│   ├── ServiX.sln
│   ├── src/
│   │   ├── ServiX.Api/             # Web API: controller, middleware, DI, Program.cs
│   │   ├── ServiX.Application/     # use case'ler, DTO'lar, arayüzler (modül klasörleri)
│   │   ├── ServiX.Domain/          # entity'ler, enum'lar, domain event'ler (modül klasörleri)
│   │   └── ServiX.Infrastructure/  # EF Core, Redis, JWT/auth, migration'lar
│   └── tests/
│       ├── ServiX.UnitTests/
│       └── ServiX.IntegrationTests/   # tenant izolasyon testleri burada
├── frontend/                       # Next.js (sonraki aşama)
├── docker/
│   └── docker-compose.yml          # postgres + redis
├── docs/
│   ├── architecture.md
│   └── adr/                        # Architecture Decision Records
├── .github/workflows/ci.yml
├── CLAUDE.md
└── README.md
```

`Domain` ve `Application` içinde modüller **klasör** olarak ayrılır: `Tenants/`, `Iam/`, `Crm/`, `Inventory/`, `Finance/`, `Tasks/`, `Audit/`.

### 4.2 Modül kuralları (mikroservise bölünebilirliğin şartı)

- Bir modül başka bir modülün **iç sınıflarına** erişmez; yalnızca o modülün **public arayüzleri** ve **domain event'leri** üzerinden konuşur.
- Modüller arasında **navigation property yok**; başka modülün verisine yalnızca **Id** ile referans verilir (ör. `Invoice.CustomerId`). Gerekirse veri, ilgili modülün servis arayüzünden alınır.
- Bağımlılık yönü: `Api → Application → Domain`, `Infrastructure → Application/Domain`. `Domain` hiçbir şeye bağımlı değildir.

### 4.3 Multi-tenancy stratejisi

**Ortak veritabanı + ortak şema; her tenant'a ait tabloda `TenantId` kolonu.**

İki katmanlı izolasyon (derinlemesine savunma):

1. **Uygulama katmanı:**
   - `ITenantProvider`, aktif tenant'ı JWT içindeki `tenant_id` claim'inden istek başına çözer (middleware).
   - Tenant'a ait tüm entity'ler `ITenantEntity` arayüzünü uygular.
   - EF Core **global query filter**: `e => e.TenantId == tenantProvider.TenantId`.
   - `SaveChanges` interceptor'ı yeni kayıtlara `TenantId`'yi otomatik yazar.
2. **Veritabanı katmanı (PostgreSQL Row-Level Security):**
   - Her istekte bağlantıya `app.current_tenant` ayarı yazılır (`set_config`).
   - Tablolarda RLS politikası: `USING (tenant_id = current_setting('app.current_tenant')::uuid)`.
   - Uygulama, tabloların sahibi olmayan bir DB kullanıcısıyla bağlanır (ya da `FORCE ROW LEVEL SECURITY` kullanılır), aksi halde RLS atlanır.

**Zorunlu test:** "A şirketinin kullanıcısı B şirketinin verisini göremez/değiştiremez" için entegrasyon testleri (Testcontainers ile gerçek PostgreSQL). Bu testler projenin en önemli kanıtıdır.

### 4.4 Kimlik doğrulama ve yetkilendirme

- **JWT access token** (kısa ömürlü, örn. 15 dk) + **refresh token** (DB'de **hash'li** saklanır, her kullanımda **rotasyon**).
- Parola hash'i: Argon2id (ör. `Konscious.Security.Cryptography`).
- **RBAC:** `Role` + `Permission` (`resource:action` biçiminde anahtarlar, ör. `customer:read`, `invoice:create`). Roller tenant'a özeldir; `Owner` gibi sistem rolleri silinemez (`IsSystem`).
- Yetki kontrolü: `[HasPermission("customer:read")]` benzeri bir attribute + authorization policy handler.
- Önemli işlemler `AuditLog`'a yazılır.

### 4.5 Genel mühendislik kuralları

- Global hata yönetimi: `ProblemDetails` formatında, **iç hata ayrıntısı istemciye sızdırılmaz**; ayrıntı yalnızca loglanır.
- Girdi doğrulaması her endpoint'te (FluentValidation / DataAnnotations); sayısal alanlarda alt-üst sınır (örn. miktar ≥ 1).
- Ana kayıtlarda **soft delete** (`DeletedAt`).
- Tüm zaman damgaları **UTC**.
- Anahtarlar `Guid`.
- Sayfalama/filtreleme için ortak, tutarlı bir yapı.
- Stok gibi eşzamanlılık gerektiren yerlerde **atomik, koşullu UPDATE** kullan (örn. `UPDATE ... SET qty = qty - @n WHERE qty >= @n`); okuyup sonra yazma yok. Stok hareketleri değiştirilemez (append-only) defterdir.

## 5. Veritabanı şeması (EF Core entity'leri)

Tüm tenant'a ait tablolarda `TenantId` vardır (alt tablolarda da, kasıtlı denormalizasyon: RLS ve query filter tek kuralla çalışsın). Kolon adları veritabanında snake_case olur.

**Enum'lar:** `TenantPlan {Free, Starter, Pro}`, `TenantStatus {Active, Suspended}`, `UserStatus {Active, Invited, Disabled}`, `StockMovementType {In, Out, Adjustment, Transfer}`, `InvoiceStatus {Draft, Sent, PartiallyPaid, Paid, Overdue, Void}`, `PaymentMethod {Cash, BankTransfer, Card, Other}`, `TaskStatus {Todo, InProgress, Review, Done}`, `TaskPriority {Low, Medium, High, Urgent}`.

### Tenant ve IAM

| Tablo | Alanlar | Notlar |
|---|---|---|
| **Tenant** | Id, Name, Slug (unique), Plan, Status, Currency (varsayılan `TRY`), Timezone (varsayılan `Europe/Istanbul`), CreatedAt, UpdatedAt | Tenant tablosunun kendisinde `TenantId` yok |
| **User** | Id, TenantId, Email, PasswordHash, FirstName, LastName, Status, LastLoginAt, CreatedAt, UpdatedAt, DeletedAt | Unique: `(TenantId, Email)` |
| **Role** | Id, TenantId, Name, IsSystem, CreatedAt | Unique: `(TenantId, Name)` |
| **Permission** | Id, Key (unique), Description | Global katalog |
| **UserRole** | UserId, RoleId | Birleşik anahtar |
| **RolePermission** | RoleId, PermissionId | Birleşik anahtar |
| **RefreshToken** | Id, UserId, TokenHash (unique), ExpiresAt, RevokedAt, CreatedAt | |

### CRM

| Tablo | Alanlar | Notlar |
|---|---|---|
| **Customer** | Id, TenantId, OwnerId (User, null olabilir), Name, Email, Phone, TaxNumber, Address, Notes, CreatedAt, UpdatedAt, DeletedAt | İndeks: `(TenantId, Name)` |
| **Contact** | Id, TenantId, CustomerId, FirstName, LastName, Email, Phone, Position, CreatedAt | Customer silinince cascade |

### Inventory

| Tablo | Alanlar | Notlar |
|---|---|---|
| **Product** | Id, TenantId, Sku, Name, Description, Unit (varsayılan `adet`), UnitPrice decimal(14,2), TaxRate decimal(5,2) (varsayılan 20), ReorderLevel, IsActive, CreatedAt, UpdatedAt, DeletedAt | Unique: `(TenantId, Sku)` |
| **Warehouse** | Id, TenantId, Name, Location | Unique: `(TenantId, Name)` |
| **StockLevel** | Id, TenantId, ProductId, WarehouseId, Quantity | Anlık bakiye; unique: `(ProductId, WarehouseId)`; hareketle **aynı transaction'da** güncellenir; `CHECK (Quantity >= 0)` |
| **StockMovement** | Id, TenantId, ProductId, WarehouseId, Type, Quantity (işaretli: giriş +, çıkış −), Reference (ör. `INVOICE:<id>`), Note, CreatedById, CreatedAt | **Append-only** defter; indeks: `(TenantId, ProductId, CreatedAt)` |

### Finance

| Tablo | Alanlar | Notlar |
|---|---|---|
| **Invoice** | Id, TenantId, CustomerId, Number (ör. `INV-2026-0001`), Status, IssueDate, DueDate, Currency, Subtotal, TaxTotal, Total (hepsi decimal(14,2)), Notes, CreatedAt, UpdatedAt, DeletedAt | Unique: `(TenantId, Number)`; numara tenant içinde sıralı, çakışma transaction/kilit ile önlenir |
| **InvoiceItem** | Id, TenantId, InvoiceId, ProductId (null olabilir), Description, Quantity, UnitPrice, TaxRate, LineTotal | `UnitPrice` satış anındaki **anlık görüntü**dür (ürün fiyatı değişse de fatura değişmez) |
| **Payment** | Id, TenantId, InvoiceId, Amount, Method, PaidAt, Reference | |

### Tasks

| Tablo | Alanlar | Notlar |
|---|---|---|
| **Task** | Id, TenantId, Title, Description, Status, Priority, DueDate, AssigneeId (null olabilir), CreatorId, CustomerId (null olabilir), CreatedAt, UpdatedAt, DeletedAt | İndeks: `(TenantId, Status)`, `(TenantId, AssigneeId)` |
| **TaskComment** | Id, TenantId, TaskId, AuthorId, Body, CreatedAt | |

### Audit

| Tablo | Alanlar | Notlar |
|---|---|---|
| **AuditLog** | Id, TenantId, UserId (null olabilir), Action (ör. `invoice.create`), Entity, EntityId, Metadata (jsonb), CreatedAt | İndeks: `(TenantId, CreatedAt)`, `(TenantId, Entity, EntityId)` |

## 6. Sık kullanılan komutlar (PowerShell uyumlu)

```powershell
# Yerel PostgreSQL + Redis'i başlat / durdur
docker compose -f docker/docker-compose.yml up -d
docker compose -f docker/docker-compose.yml down

# Derle ve test et
dotnet build backend/ServiX.sln
dotnet test backend/ServiX.sln

# API'yi çalıştır
dotnet run --project backend/src/ServiX.Api

# EF Core migration (Infrastructure'da, başlangıç projesi Api)
dotnet ef migrations add <Name> --project backend/src/ServiX.Infrastructure --startup-project backend/src/ServiX.Api
dotnet ef database update --project backend/src/ServiX.Infrastructure --startup-project backend/src/ServiX.Api
```

Uygulanmış bir migration dosyasını elle değiştirme; yeni migration ekle.

## 7. Yol haritası ve ilk yapılacak adımlar

Dikey dilimlerle ilerlenir: önce çekirdek (tenant + auth + RBAC) ve **bir modül (CRM) baştan sona** çalışır ve **canlıya çıkar**; sonra diğer modüller eklenir. Tahmini süre: haftada 10–15 saatle yaklaşık 10 hafta.

### Aşama 1 — Temel (şu an buradayız)

- [x] Geliştirme ortamı: Git, Node.js + pnpm, Docker Desktop, Visual Studio
- [ ] GitHub'da `servix` deposu (public, README + .gitignore (VisualStudio) + MIT lisans) ve yerel klonu
- [x] `backend/` altında `ServiX.sln` ve dört proje: `Api`, `Application`, `Domain`, `Infrastructure` (+ proje referansları, `net10.0`)
- [ ] `docker/docker-compose.yml` ile PostgreSQL + Redis; bağlantı dizeleri `user-secrets`'ta
- [ ] `ApplicationDbContext`, `Tenant` ve `User` entity'leri, ilk migration
- [ ] Tenant bağlamı: `ITenantProvider`, middleware, global query filter, `SaveChanges` interceptor
- [ ] Sağlık kontrolü endpoint'i, Serilog, global hata yönetimi (`ProblemDetails`)
- [ ] **Tenant izolasyon testi** (Testcontainers) + PostgreSQL RLS politikaları
- [ ] GitHub Actions CI (build + test), `docs/adr/` ilk kayıtlar

### Sonraki aşamalar

2. **Şirket yapısı tamamlama:** tüm tenant tabloları, RLS politikaları
3. **Giriş sistemi:** tenant kaydı + ilk Owner, login, JWT + refresh rotasyonu, RBAC (Permission seed'i, `HasPermission`), kullanıcı daveti, audit log
4. **CRM** (Customer, Contact) API + ekran → **ilk canlı yayın** (demo linki ve demo hesabı)
5. **Tasks** (Kanban akışı, atama, yorum)
6. **Finance** (fatura, kalemler, ödeme, durum geçişleri)
7. **Inventory** (basit sürüm: ürün, depo, stok hareketi; atomik stok düşümü; fatura ile domain event bağlantısı)
8. **Cila:** Redis önbellek, rate limiting, sayfalama standardı, dashboard özet kartları, Dockerfile'lar, README + mimari diyagram, demo verisi (seed), CV/LinkedIn maddeleri

**Şimdilik kapsam dışı:** e-fatura entegrasyonu, barkod/QR, mikroservislere fiilen bölme, gelişmiş raporlama, 2FA. (Zaman kalırsa: tedarikçi/satın alma, gider yönetimi, PDF/Excel raporları, 2FA.)

## 8. "Bitti" tanımı (her modül için)

- Kod derleniyor, uyarı yok; `dotnet test` yeşil.
- Servis katmanı için birim testi, kritik akışlar için entegrasyon testi; **tenant izolasyonu** bu modül için de test edilmiş.
- Girdi doğrulaması var; hata mesajları iç ayrıntı sızdırmıyor.
- Yetki gereken endpoint'ler `HasPermission` ile korunuyor; önemli işlemler `AuditLog`'a yazılıyor.
- Swagger/OpenAPI belgesi güncel; README'de kısa kullanım notu var.
- CI geçiyor; commit'ler Conventional Commits.

## 9. Farklılaşma hedefi

Benzer açık kaynak "SaaS ERP" projelerinin çoğunda gerçek tenant izolasyonu, rol/izin sistemi, otomatik test, mimari belge ve herkese açık çalışan demo eksiktir. ServiX tam olarak bunlarla öne çıkmalıdır: **RLS + izolasyon testleri, RBAC, audit log, tek komutla kurulum (`docker compose up`), demo verisi, canlı demo, ADR'ler ve temiz README.**
