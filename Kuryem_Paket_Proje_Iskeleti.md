# Kuryem_Paket — Proje iskeleti

Bu belge, [pilot mimarisini](Kuryem_Paket_Mimari.md) kaynak koduna uygulamak için depo düzenini tanımlar. Başlangıçta tek depoda üç uygulama bulunur. NestJS API iş kurallarını uygular, Next.js işletme ve operasyon panellerini sunar, Kotlin uygulaması kuryenin vardiya ve teslimat işlerini yönetir.

## Depo düzeni

```text
kuryem-paket/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── main.ts
│   │   │   ├── app.module.ts
│   │   │   ├── modules/
│   │   │   │   ├── access/
│   │   │   │   ├── orders/
│   │   │   │   ├── deliveries/
│   │   │   │   ├── couriers/
│   │   │   │   └── tracking/
│   │   │   └── platform/
│   │   │       ├── config/
│   │   │       ├── database/
│   │   │       └── telemetry/
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   ├── test/
│   │   │   └── e2e/
│   │   └── package.json
│   ├── web/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── operations/
│   │   │   │   └── merchant/
│   │   │   ├── features/
│   │   │   └── lib/
│   │   │       └── api/
│   │   └── package.json
│   └── courier-android/
│       ├── app/
│       │   └── src/
│       │       ├── main/
│       │       │   └── java/<paket-yolu>/
│       │       │       ├── app/
│       │       │       ├── feature/
│       │       │       │   ├── deliveries/
│       │       │       │   └── shift/
│       │       │       └── data/
│       │       │           ├── local/
│       │       │           ├── remote/
│       │       │           ├── sync/
│       │       │           └── location/
│       │       ├── test/
│       │       └── androidTest/
│       ├── gradlew
│       └── settings.gradle.kts
├── contracts/
│   └── openapi.json
├── infra/
│   └── local/
│       └── compose.yaml
├── .github/
│   └── workflows/
│       └── ci.yml
├── .env.example
├── package.json
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
├── README.md
└── Kuryem_Paket_Mimari.md
```

Node uygulamaları tek bir pnpm çalışma alanı kullanır. Android uygulaması kendi Gradle Wrapper’ıyla derlenir. Yeni klasörler ilgili özellik geliştirildiğinde açılır.

## Kodun yeri ve sahipliği

| İş | Konum | Kural |
|---|---|---|
| Kiracı, şube üyeliği ve rol kontrolü | `api/src/modules/access` | Yetkili kapsamı API belirler. Web ve Android bu kapsamı sunucudan alır. |
| Sipariş oluşturma ve dış sipariş referansı | `api/src/modules/orders` | Sipariş verisini bu modül yönetir. Teslimat açılışını ilgili modülün işlemi üzerinden başlatır. |
| Atama, yeniden atama, iptal ve teslimat | `api/src/modules/deliveries` | Durum geçişleri ve aktif atama kuralları burada tutulur. |
| Vardiya ve kurye kapasitesi | `api/src/modules/couriers` | Atama işlemi, kapasite kontrolünü bu modül üzerinden aynı veritabanı işleminde yapar. |
| Konum alımı, geçmiş ve son konum | `api/src/modules/tracking` | Konum verisi kendi uç noktalarından alınır. |
| Web sayfaları ve ekran bileşenleri | `web/src/app`, `web/src/features` | Rota ve yerleşim `app` altında, işe özgü bileşenler `features` altında durur. |
| Android çevrimdışı işlemleri ve konum | `courier-android/.../data` | Room kayıtları `local`, yeniden gönderim `sync`, vardiya konumu `location` altında tutulur. |
| API sözleşmesi ve istemciler | `contracts/openapi.json` | NestJS’ten üretilir. Web ve Android istemcileri aynı sözleşmeyi kullanır. CI sözleşmenin güncel olduğunu kontrol eder. |

## Bir özelliğin dosyalara dağılımı

Kurye atama akışı için başlangıçta aşağıdaki dosyalar yeterlidir:

```text
modules/deliveries/
├── deliveries.module.ts
├── deliveries.controller.ts
├── assign-courier.dto.ts
├── assign-courier.service.ts
├── delivery.repository.ts
└── assign-courier.service.spec.ts
```

Controller HTTP girdisini alır ve servise aktarır. Servis kiracı ve şube yetkisini, teslimat sürümünü, aktif atamayı ve kurye kapasitesini kontrol eder. Repository sorguları yürütür. Atama, kapasite ve durum geçmişi aynı PostgreSQL işlemi içinde güncellenir. Aktif atama benzersizlik kuralı Prisma migration içindeki PostgreSQL indeksiyle korunur. Modüller birbirinin repository’sine doğrudan erişmek yerine NestJS modülünün dışa açtığı servis üzerinden çalışır.

## İlk geliştirme sırası

1. **Yerel ortam.** PostgreSQL/PostGIS’i Compose ile başlat. API’ye migration ve sağlık uç noktası ekle. Web ve Android geliştirme ortamlarının aynı API’ye bağlanmasını sağla.
2. **İlk uçtan uca akış.** Kiracı ve şube yetkisiyle sipariş aç, operasyon ekranından kurye ata ve Android uygulamasından teslimatı tamamla.
3. **Kritik doğrulamalar.** Gerçek PostgreSQL ile iki eşzamanlı atamayı, eski atamaya gelen mobil komutu ve tekrar gönderilen teslimat komutunu test et. Android’de ağ kesintisi, ekran kilidi ve uygulamanın yeniden açılmasını dene.
4. **Pilot kapsamını genişlet.** Bildirim veya dış entegrasyon devreye girince API’ye ilgili modülü ve PostgreSQL outbox’ını ekle. İşleyiciyi aynı `apps/api` kod tabanında `worker.ts` giriş noktasıyla çalıştır. Nakit pilot kapsamına girerse `payments` modülü, iOS kesinleşirse `apps/courier-ios` ekle.

`README.md` kurulum ve çalıştırma komutlarının giriş noktasıdır. CI, değişen uygulamaların derleme ve kontrollerini çalıştırır. Veritabanı migration’ları API’de tek yerde tutulur. Yerel ayarlar için `.env.example` kullanılır ve gerçek erişim bilgileri ortam yapılandırmasında saklanır.
