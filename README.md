# Rails E-Commerce API

Ruby on Rails ile geliştirilmiş, token tabanlı kimlik doğrulamaya sahip bir **e-ticaret REST API**'si. Ürün ve kategori yönetimini (CRUD, görsel yükleme, isme göre arama) JSON uç noktaları üzerinden sunar; yetkilendirme Devise Token Auth + Pundit + özel rol modülleriyle çok katmanlı olarak kurgulanmıştır. Uygulamanın öne çıkan yanı, önbellekleme / loglama / hata yönetimi / güvenlik gibi **cross-cutting concern**'lerin `app/cross_cutting_concern` altında bağımsız modüllere ayrılıp `ApplicationController` üzerinden tüm controller'lara otomatik olarak kazandırılmasıdır. Redis tabanlı yanıt önbelleği ve Sidekiq ile arka plan iş kuyruğu da uygulamanın bir parçasıdır.

![Ruby](https://img.shields.io/badge/Ruby-3.0.0-CC342D?style=flat&logo=ruby&logoColor=white)
![Rails](https://img.shields.io/badge/Rails-6.1.4-CC0000?style=flat&logo=rubyonrails&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-pg%201.2-336791?style=flat&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-4.6-DC382D?style=flat&logo=redis&logoColor=white)
![Sidekiq](https://img.shields.io/badge/Sidekiq-6.4-B1003E?style=flat&logo=sidekiq&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

---

## Özellikler

### Kimlik Doğrulama ve Kullanıcı Yönetimi
- **Devise Token Auth** ile token tabanlı (stateless) kimlik doğrulama; tüm auth uç noktaları `/api/auth` altında sunulur.
- Kayıt olma, giriş/çıkış, token doğrulama ve şifre sıfırlama akışları.
- Kayıt sırasında `first_name`, `last_name`, `username` alanları `ApplicationController#configure_permitted_parameters` ile whitelist'e alınır ve modelde zorunlu tutulur.
- Kullanıcı modelinde `enum role: [:user, :admin, :superadmin]` ve ayrıca `users ↔ roles` arasında `user_roles` ara tablosuyla **çoklu rol** desteği.

### Ürün Yönetimi (`/api/products`)
- Tam CRUD: listeleme, detay, oluşturma, güncelleme, silme.
- **İsme göre arama**: `GET /api/products/get_by_name` (custom collection route), sonuçlar `created_at` azalan sıralı.
- **Active Storage** ile ürün görseli yükleme (`has_one_attached :product_image`); JSON yanıtlarında görsel `rails_blob_url` ile mutlak URL olarak döner.
- Jbuilder şablonlarıyla (`index/show/create/update/destroy/error.json.jbuilder`) aksiyona özel, tutarlı JSON çıktısı — her yanıt `message` ve `success` alanları taşır.
- Zengin model doğrulamaları: ad uzunluğu (2–15), açıklama uzunluğu (100–500), stok ve fiyat için sayısal kontroller ve özel bir doğrulama kuralı (`name_start_with_a`).

### Kategori Yönetimi (`/api/categories`)
- Tam CRUD; listeleme `created_at` azalan sıralı.
- **Rol bazlı erişim kontrolü**: `create`, `update`, `destroy` yalnızca `admin` ve `superadmin` rollerine; okuma uç noktaları tüm tanımlı rollere açıktır (`check_user_roles`).
- İstek/yanıt döngüsünün her adımı filtrelerle örülmüştür: `read_cache` → `set_category` → aksiyon → `write_cache` / `remove_cache` → `log_file` → `log_database`.

### Önbellekleme (Redis)
- `before_action :read_cache` ile istek daha controller'a girmeden önbellekten karşılanır; anahtar `"#{request.fullpath},#{action_name}"` formatındadır.
- Cache isabeti varsa ilgili Jbuilder şablonu doğrudan render edilir (`@is_cached` bayrağı ile), veritabanına hiç gidilmez.
- Cache miss durumunda `after_action :write_cache` sonucu **60 dakika** TTL ile yazar.
- Yazma işlemlerinden (`create/update/destroy`) sonra `remove_cache`, ilgili controller'a ait tüm anahtarları Redis'ten temizleyerek önbelleği geçersiz kılar.

### Loglama ve Hata Yönetimi
- **Dosya loglama** (`Log::FileLogger`): her istek için controller'a özel log dosyasına (`log/log_<controller>.log`) parametreler, oturumdaki kullanıcı, HTTP metodu, `remote_ip`/`ip`, tam URL ve zaman damgası JSON olarak yazılır.
- **Hata loglama**: yakalanan istisnalar mesaj ve backtrace ile birlikte `log/log_error_<controller>.log` dosyasına düşer.
- **Merkezi hata yakalama** (`CustomError::ErrorHandler`): `rescue_from` ile `Exception`, `ActiveRecord::RecordNotFound` ve `ActionController::RoutingError` tek noktadan ele alınıp standart JSON hata yanıtına dönüştürülür.
- Veritabanı tabanlı loglama için `db_logs` ve `db_log_errors` tabloları/modelleri hazırdır (dosya loglarıyla birebir aynı şema).

### Arka Plan İşleri (Sidekiq)
- Active Job adaptörü olarak **Sidekiq** yapılandırılmıştır (`config.active_job.queue_adapter = :sidekiq`), Redis üzerinde çalışır.
- `ProductJob`: ürün kaydedildikten sonra zamanlanmış olarak tetiklenir ve ürünü kuyruktan siler — gecikmeli/zamanlanmış iş kurgusunun örneğidir.
- `HelloJob`: kuyruk altyapısını doğrulamak için basit bir örnek iş.

---

## Teknolojiler

| Katman | Teknoloji | Versiyon |
|---|---|---|
| Dil | Ruby | 3.0.0 |
| Framework | Rails | 6.1.4.1 |
| Veritabanı | PostgreSQL (`pg`) | 1.2.3 |
| Uygulama sunucusu | Puma | 5.5.2 |
| Kimlik doğrulama | Devise | 4.8.1 |
| Token auth | devise_token_auth | 1.2.0 |
| Yetkilendirme | Pundit | 2.1.1 |
| Şifreleme | bcrypt | 3.1.16 |
| Önbellek / kuyruk | Redis + redis-rails | 4.6.0 / 5.0.2 |
| Arka plan işleri | Sidekiq | 6.4.1 |
| JSON şablonları | Jbuilder | 2.11.3 |
| Frontend derleme | Webpacker | 5.4.3 |
| Stil | sass-rails | 6.0.0 |
| Sayfa geçişleri | Turbolinks | 5.2.1 |
| Boot hızlandırma | Bootsnap | 1.9.3 |

**Geliştirme araçları:** `byebug`, `web-console`, `rack-mini-profiler` (SQL süresi ve flame graph), `listen`, `spring`, `annotate` (model/fixture/test dosyalarına şema bilgisini otomatik yazar).
**Test araçları:** Minitest (Rails yerleşik), `capybara`, `selenium-webdriver`, `webdrivers`.

---

## Mimari

### Cross-Cutting Concern Katmanı

Projenin ayırt edici tasarım kararı, uygulamanın **dikey kesen (cross-cutting) sorumluluklarının** iş mantığından tamamen ayrılmış olmasıdır. Rails'in Zeitwerk yükleyicisi `app/` altındaki her klasörü otomatik bir kök dizin olarak ele aldığı için, bu modüller namespace'leriyle birlikte (`Cache::RedisCache`, `Log::FileLogger`, …) hiçbir ek yapılandırma olmadan yüklenir.

```
app/cross_cutting_concern/
├── cache/
│   ├── redis_cache.rb        # Cache::RedisCache   → read_cache / write_cache / remove_cache
│   └── memory_cache.rb       # (yer tutucu)
├── custom_error/
│   └── error_handler.rb      # CustomError::ErrorHandler → merkezi rescue_from + hata logu
├── log/
│   └── file_logger.rb        # Log::FileLogger     → log_file / log_error_to_file
└── security/
    ├── role_module.rb        # Security::RoleModule      → rol sabitleri ve rol kümeleri
    └── security_operation.rb # Security::SecurityOperation → check_user_roles
```

Tüm bu modüller `ApplicationController` içinde tek noktadan `include` edilir; böylece her controller, tek satır bile yazmadan önbellekleme, loglama, hata yakalama ve rol denetimi yeteneklerini devralır:

```ruby
class ApplicationController < ActionController::Base
  include DeviseTokenAuth::Concerns::SetUserByToken
  include Pundit
  include Security::SecurityOperation
  include Security::RoleModule
  include Cache::RedisCache
  include Log::FileLogger
  include CustomError::ErrorHandler
end
```

Controller'lar bu yetenekleri deklaratif filtrelerle (`before_action` / `after_action`) devreye alır — aksiyon gövdeleri yalnızca iş mantığı içerir.

### Çok Katmanlı Yetkilendirme

Yetkilendirme iki bağımsız mekanizmayla kurgulanmıştır:

1. **Pundit politikaları** (`app/policies/`) — `ApplicationPolicy`, yapıcısına geçirilen `role_result` bayrağına göre tüm izin sorularını (`index?`, `show?`, `create?`, `update?`, `destroy?`) tek bir yerden yanıtlar. `AdminPolicy` bu bayrağı `user.admin? || user.superadmin?` olarak hesaplar; `ProductPolicy`, `CategoryPolicy` ve `IstatistikPolicy` ondan türeyerek kuralı devralır ve gerektiğinde ezer (örn. `ProductPolicy#index?` herkese açıktır). Yetkisiz erişimde `Pundit::NotAuthorizedError` yakalanıp `401` JSON yanıtına çevrilir.
2. **Rol modülü** (`Security::RoleModule` + `SecurityOperation#check_user_roles`) — `user_roles` ara tablosu üzerinden kullanıcının rollerini okur ve izin verilen rol kümesiyle (`only_admin_and_superadmin`, `super_admin`, `only_human_resources`, `all_roles`) karşılaştırır. Controller'larda lambda filtresi olarak kullanılır:
   ```ruby
   before_action -> { check_user_roles(Security::RoleModule.only_admin_and_superadmin) },
                 only: %i[update create destroy]
   ```

### Model İlişkileri

```
Category 1 ──── n Product ──── 1 ActiveStorage::Attachment (product_image)

User n ──── n Role
      └── UserRole (user_id, role_id)

DbLog / DbLogError   (bağımsız log tabloları)
```

- `Category has_many :products` / `Product belongs_to :category`
- `UserRole belongs_to :user` ve `belongs_to :role` — kullanıcı-rol çoka-çok ilişkisinin ara modeli
- `Product has_one_attached :product_image` (Active Storage)
- `User`, Devise modülleri (`database_authenticatable`, `registerable`, `recoverable`, `rememberable`, `validatable`) ve `DeviseTokenAuth::Concerns::User` ile genişletilmiştir.

### Model Yaşam Döngüsü Kancaları

- `Product after_create :send_notification` — yeni ürün bildirimi için kanca noktası.
- `Product after_save :delete_product_after_30days` — `ProductJob`'ı gecikmeli olarak kuyruğa alır (`set(wait: ...).perform_later`).
- `Category` üzerinde `before_save`, `after_save`, `after_update`, `before_destroy` kancalarının tamamı tanımlıdır.

### API Uç Noktaları

Tüm uç noktalar `api` namespace/scope'u altındadır.

| Metod | Endpoint | Açıklama |
|---|---|---|
| `POST` | `/api/auth` | Kullanıcı kaydı |
| `POST` | `/api/auth/sign_in` | Giriş (token üretir) |
| `DELETE` | `/api/auth/sign_out` | Çıkış |
| `GET` | `/api/auth/validate_token` | Token doğrulama |
| `POST` / `PUT` | `/api/auth/password` | Şifre sıfırlama / güncelleme |
| `GET` | `/api/products` | Ürünleri listele |
| `GET` | `/api/products/get_by_name` | Ürünü isme göre ara |
| `GET` | `/api/products/:id` | Ürün detayı |
| `POST` | `/api/products` | Ürün oluştur (görsel yüklenebilir) |
| `PATCH` / `PUT` | `/api/products/:id` | Ürün güncelle |
| `DELETE` | `/api/products/:id` | Ürün sil |
| `GET` | `/api/categories` | Kategorileri listele |
| `GET` | `/api/categories/:id` | Kategori detayı |
| `POST` | `/api/categories` | Kategori oluştur *(admin/superadmin)* |
| `PATCH` / `PUT` | `/api/categories/:id` | Kategori güncelle *(admin/superadmin)* |
| `DELETE` | `/api/categories/:id` | Kategori sil *(admin/superadmin)* |

---

## Veritabanı Şeması

PostgreSQL üzerinde, `plpgsql` eklentisi etkin. Şema sürümü: `2022_02_23_195821`.

### `products`
| Kolon | Tip | Not |
|---|---|---|
| `id` | bigint | PK |
| `name` | string | |
| `description` | string | |
| `quantity` | integer | |
| `price` | float | |
| `category_id` | bigint | FK → `categories.id`, `not null`, indeksli |
| `created_at` / `updated_at` | datetime | |

### `categories`
| Kolon | Tip |
|---|---|
| `id` | bigint (PK) |
| `name` | string |
| `created_at` / `updated_at` | datetime |

### `users`
| Kolon | Tip | Not |
|---|---|---|
| `id` | bigint | PK |
| `provider` / `uid` | string | varsayılan `email` / `""`, birlikte **unique** indeksli |
| `encrypted_password` | string | |
| `email` | string | **unique** indeksli |
| `username`, `first_name`, `last_name` | string | |
| `tokens` | json | devise_token_auth oturum token'ları |
| `role` | integer | enum, varsayılan `0` (`user`) |
| `reset_password_token` | string | **unique** indeksli |
| `reset_password_sent_at`, `allow_password_change`, `remember_created_at` | — | Devise recoverable/rememberable |
| `confirmation_token` (unique), `confirmed_at`, `confirmation_sent_at`, `unconfirmed_email` | — | Devise confirmable alanları |

### `roles` / `user_roles`
- `roles`: `id`, `name`, zaman damgaları.
- `user_roles`: `user_id` (FK → `users`) ve `role_id` (FK → `roles`), her ikisi de `not null` ve indeksli — kullanıcı ile rol arasındaki çoka-çok bağını kurar.

### `db_logs` / `db_log_errors`
İstek denetim kaydı için hazırlanan tablolar: `parameters` (json), `current_user` (json), `date_time`, `method`, `controller_name`, `remote_ip`, `ip`, `request`, `request_fullpath`. `db_log_errors` bunlara ek olarak `exception` (string) ve `exception_detail` (json) kolonlarını taşır.

### `active_storage_*`
Rails Active Storage standart tabloları: `active_storage_blobs`, `active_storage_attachments` (polimorfik `record_type`/`record_id` + unique indeks) ve `active_storage_variant_records`.

### Yabancı Anahtarlar
```
products.category_id                      → categories.id
user_roles.user_id                        → users.id
user_roles.role_id                        → roles.id
active_storage_attachments.blob_id        → active_storage_blobs.id
active_storage_variant_records.blob_id    → active_storage_blobs.id
```

---

## Kurulum

### Gereksinimler
- Ruby **3.0.0** (bkz. `.ruby-version`)
- PostgreSQL 9.3+
- Redis 6+ (önbellek ve Sidekiq için, `localhost:6379`)
- Node.js ve Yarn (Webpacker için)

### Adımlar

```bash
# 1) Depoyu klonlayın
git clone https://github.com/db-akyol/Rails-Ecommerce.git
cd Rails-Ecommerce

# 2) Ruby bağımlılıklarını kurun
bundle install

# 3) JavaScript bağımlılıklarını kurun
yarn install

# 4) Veritabanını oluşturun ve şemayı uygulayın
rails db:create
rails db:migrate

# 5) (Opsiyonel) Başlangıç verilerini yükleyin
rails db:seed
```

Varsayılan veritabanı adları `config/database.yml` içinde tanımlıdır: geliştirme için `ecommerce_development`, test için `ecommerce_test`. Üretimde veritabanı parolası `ECOMMERCE_DATABASE_PASSWORD` ortam değişkeninden okunur.

---

## Çalıştırma

Uygulamanın tam olarak çalışabilmesi için üç süreç gerekir:

```bash
# 1) Redis (önbellek + Sidekiq kuyruğu)
redis-server

# 2) Sidekiq işçisi (arka plan işleri)
bundle exec sidekiq

# 3) Rails sunucusu
rails server
```

Uygulama varsayılan olarak `http://localhost:3000` adresinde ayağa kalkar; API uç noktalarına `http://localhost:3000/api/...` üzerinden erişilir.

Geliştirme ortamında önbellek deposu Redis'tir (`redis://localhost:6379/0`); `rails dev:cache` komutuyla controller seviyesindeki Rails önbelleklemesi açılıp kapatılabilir.

### Örnek İstek

```bash
# Giriş yaparak token alın
curl -i -X POST http://localhost:3000/api/auth/sign_in \
  -d "email=user@example.com&password=secret"

# Dönen access-token / client / uid başlıklarıyla ürünleri listeleyin
curl http://localhost:3000/api/products \
  -H "access-token: <token>" -H "client: <client>" -H "uid: <uid>"
```

### Testler

```bash
rails test
```

Test iskeleti `test/` altında modeller, controller'lar, job'lar ve **Pundit politikaları** için hazırlanmıştır; fixture'lar `test/fixtures/` içinde tanımlıdır. Sistem testleri Capybara + Selenium ile çalıştırılabilir.

---

## Lisans

Bu proje **MIT Lisansı** ile lisanslanmıştır. Ayrıntılar için [LICENSE](LICENSE) dosyasına bakınız.
