# Setup Shopify — Fitur Voucher DKRC Member

Spesifikasi setup di sisi Shopify (duraking.co.id) untuk mendukung fitur voucher
member DKRC.

**Fitur:** Member DKRC generate voucher diskon 10% dari app DKRC. Voucher unik
per klik, berlaku 2 jam, single-use, **minimum belanja Rp 100.000**. Format
kode: `MBR-XXXXXX`.

**Estimasi total waktu setup:** ~1 jam (mostly klik-klik di Shopify Admin,
zero coding).

---

## 1. Custom App + Admin API Access Token

**Langkah di Shopify Admin:**

1. `Settings` → `Apps and sales channels` → `Develop apps`
2. `Create an app` → nama: **`DKRC Member Voucher`**
3. `Configure Admin API scopes` → aktifkan 5 scope:
   - `write_discounts`
   - `read_discounts`
   - `write_price_rules`
   - `read_price_rules`
   - `read_orders`
4. Save → `Install app`
5. Copy **Admin API access token** (format `shpat_xxxxxxxx…`)

**Output yang perlu dikirim balik:**

- `SHOPIFY_STORE_DOMAIN` — biasanya `duraking.myshopify.com` (bukan `duraking.co.id`)
- `SHOPIFY_ADMIN_TOKEN` — string `shpat_...` dari langkah 5

---

## 2. Webhook `orders/create`

Fungsi: setiap kali ada order di duraking.co.id, Shopify notify endpoint DKRC
untuk mark voucher sebagai used di database.

### Pembagian Kerja

| Bagian | Owner | Status |
|---|---|---|
| Endpoint receiver (URL terima request) | DKRC — sudah live di `https://app.dkrc.id/api/webhooks/shopify/order` | ✅ Selesai |
| Konfigurasi webhook di Shopify Admin | Marketplace team | ⏳ Butuh setup |
| Signing secret (untuk HMAC verify) | Auto-generate Shopify saat webhook dibuat | ⏳ Perlu dikirim |

Bagian marketplace team = klik-klik di UI Shopify Admin, zero coding, ~5 menit.

### Langkah di Shopify Admin

1. `Settings` → `Notifications` → scroll ke section `Webhooks`
2. `Create webhook`:
   - **Event:** `Order creation`
   - **Format:** `JSON`
   - **URL:** `https://app.dkrc.id/api/webhooks/shopify/order`
   - **Webhook API version:** `2025-01`
3. Save → klik webhook yang baru dibuat → reveal & copy **Signing secret**

**Output yang perlu dikirim balik:**

- `SHOPIFY_WEBHOOK_SECRET` — signing secret dari langkah 3

### Alur Kerja Setelah Setup

```
Customer checkout di duraking.co.id (pakai kode MBR-ABC123)
    ↓
Shopify buat order → cek daftar webhook → trigger orders/create
    ↓
POST https://app.dkrc.id/api/webhooks/shopify/order
    - Body: detail order + discount codes yang dipakai
    - Header X-Shopify-Hmac-Sha256: signature pakai SHOPIFY_WEBHOOK_SECRET
    ↓
Endpoint DKRC verify signature → update kolom used_at di DB
    → return 200 OK ke Shopify
    ↓
Selesai. Voucher tercatat used, tidak bisa dipakai lagi.
```

---

## 3. Theme Customization (recommended)

### 3a. Banner "Voucher DKRC Aktif" di Cart & Checkout

Trigger: discount code yang di-apply match `.startsWith("MBR-")`.

Tampilan banner:
```
✨ Voucher DKRC Member Aktif — Anda hemat 10%
```

- **Lokasi:** cart drawer + cart page + checkout summary
- **Style:** gradient warna DKRC (biru → gold), icon sparkles ✨
- **Teknik:** Liquid + JavaScript di `theme.liquid` / `cart-drawer.liquid`

### 3b. Copy Error Message

Override default error copy Shopify supaya user-friendly:

| Kondisi | Copy |
|---|---|
| Kode expired (>2 jam) | "Kode voucher sudah kadaluarsa. Generate kode baru di app DKRC." |
| Kode sudah dipakai | "Kode voucher ini sudah pernah dipakai." |
| Belanja di bawah minimum | "Voucher berlaku untuk belanja minimum Rp 100.000." |
| Kode salah format | "Kode tidak valid. Kode voucher DKRC berformat `MBR-XXXXXX`." |

### 3c. Landing Page Info Voucher (opsional)

Halaman `/pages/member-voucher` yang jelasin cara member dapat diskon 10%.

- **URL:** `duraking.co.id/pages/member-voucher`
- **CTA button:** `https://app.dkrc.id/dashboard/member/voucher` (target blank)
- **Copy CTA:** "Login ke app DKRC untuk generate kode voucher"

---

## 4. Format Kirim Balik Credentials

Setelah setup 1 + 2 selesai, reply dengan format:

```
SHOPIFY_STORE_DOMAIN=duraking.myshopify.com
SHOPIFY_ADMIN_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SHOPIFY_API_VERSION=2025-01
SHOPIFY_WEBHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Env di-input ke Vercel production → feature aktif dalam ~5-10 menit setelah
credentials diterima.

---

## 5. Testing End-to-End

Checklist yang perlu dites setelah env aktif:

- [ ] Generate voucher dari app DKRC → dapat kode `MBR-XXXXXX`
- [ ] Kode berhasil di-apply di cart duraking.co.id dengan subtotal ≥ Rp 100.000 → 10% off applied
- [ ] Kode di-apply dengan subtotal < Rp 100.000 → gagal dengan pesan "minimum Rp 100.000"
- [ ] Banner "Voucher DKRC Aktif" muncul di cart & checkout
- [ ] Complete checkout dengan test payment (Shopify Bogus Gateway)
- [ ] Cek webhook fire di Shopify Admin → status Success (200 OK)
- [ ] Kode yang sama coba dipakai lagi di cart lain → gagal (usage_limit 1)
- [ ] Cek Shopify Admin → `Discounts` → kode `MBR-*` muncul dengan usage stats
- [ ] Cek di database DKRC: kolom `used_at` ter-update untuk voucher yang dipakai

---

## Catatan Keamanan

- Token sensitif — jangan commit ke git. Rotate tiap 90 hari (best practice Shopify).
- Kalau bocor: revoke Custom App di Shopify Admin (`Apps` → klik app → `Uninstall`), lalu setup ulang.
- Webhook signature di-verify HMAC-SHA256 timing-safe — request tanpa signature valid di-reject 401.

---

## Referensi Teknis

- **API version:** `2025-01`
- **Endpoint receiver DKRC:** `https://app.dkrc.id/api/webhooks/shopify/order`
- **Preview URL app DKRC:** `https://app.dkrc.id/dashboard/member/voucher`
- **Format kode:** `MBR-XXXXXX` (6char base32, contoh `MBR-A7K3P9`)
- **Discount rule:** `-10%`, `usage_limit=1`, `once_per_customer=true`, `ends_at=+2h`, `min_subtotal=100000`
