# DKRC × Duraking — Shopify Voucher Integration

> Spesifikasi setup Shopify untuk fitur voucher member DKRC di **duraking.co.id**

![Status](https://img.shields.io/badge/status-siap%20diimplementasi-blue)
![Shopify](https://img.shields.io/badge/shopify-Admin%20API%202025--01-95BF47?logo=shopify&logoColor=white)
![Waktu](https://img.shields.io/badge/estimasi%20setup-~1%20jam-lightgrey)
![Handoff](https://img.shields.io/badge/handoff-zero%20coding-brightgreen)

Repo ini berisi **dokumen spesifikasi** untuk tim marketplace/Shopify dev yang
akan setup integrasi voucher member DKRC di store duraking.co.id. Semua setup
di sisi app DKRC sudah live di production — tinggal setup di sisi Shopify.

---

## ⚡ Ringkasan Fitur

Member DKRC bisa generate voucher diskon 10% dari app DKRC. Detail:

- 🎟️ **Diskon:** 10% off
- ⏱️ **Berlaku:** 2 jam setelah generate
- 🔒 **Limit:** 1x pakai per kode, 1x per customer
- 💰 **Minimum belanja:** Rp 100.000
- 🏷️ **Format kode:** `MBR-XXXXXX` (contoh: `MBR-A7K3P9`)
- 🛒 **Auto-apply URL:** `https://duraking.co.id/discount/{code}`

---

## 📋 Daftar Isi

- [Overview Arsitektur](#-overview-arsitektur)
- [Tanggung Jawab Setup](#-tanggung-jawab-setup)
- [Langkah Setup Lengkap](#-langkah-setup-lengkap)
- [Format Kirim Balik Credentials](#-format-kirim-balik-credentials)
- [Testing End-to-End](#-testing-end-to-end)
- [FAQ](#-faq)
- [Kontak](#-kontak)

---

## 🏗️ Overview Arsitektur

```
┌─────────────────┐        ┌─────────────────┐        ┌──────────────────┐
│   App DKRC      │        │  Shopify Admin  │        │   duraking.co.id │
│  (Next.js)      │        │      API        │        │   (storefront)   │
├─────────────────┤        ├─────────────────┤        ├──────────────────┤
│ Member klik     │──POST─▶│ Buat Price Rule │        │ Customer apply   │
│ "Generate       │        │ + Discount Code │        │ kode MBR-XXXXXX  │
│ Voucher"        │        │                 │        │ di cart          │
│                 │◀──ID───│                 │        │                  │
│                 │        │                 │        │  10% off applied │
│                 │        │                 │        │  min. Rp 100k    │
│                 │        │                 │        │                  │
│                 │        │                 │──POST─▶│ Checkout         │
│                 │        │                 │webhook │                  │
│                 │◀───────│                 │        │                  │
│ Update used_at  │        │                 │        │                  │
│ di DB           │        │                 │        │                  │
└─────────────────┘        └─────────────────┘        └──────────────────┘
```

---

## 👥 Tanggung Jawab Setup

| Area | DKRC | Marketplace Team |
|---|:---:|:---:|
| Bikin Custom App di Shopify Admin | — | ✅ |
| Provide Admin API access token | — | ✅ |
| Setup webhook `orders/create` | — | ✅ |
| Share webhook signing secret | — | ✅ |
| Theme customization (banner + error copy) | — | ✅ |
| Bikin API route generate voucher | ✅ | — |
| Bikin webhook receiver endpoint | ✅ | — |
| Bikin UI tombol generate di app DKRC | ✅ | — |
| Verify webhook signature (HMAC) | ✅ | — |
| Input credentials ke Vercel env | ✅ | — |

**Marketplace team = klik-klik di UI Shopify Admin, zero coding.**
Estimasi total waktu setup: **~1 jam**.

---

## 🚀 Langkah Setup Lengkap

Baca dokumen lengkap dengan step-by-step (screenshot-friendly, siap eksekusi):

📄 **[SETUP.md](SETUP.md)** — versi markdown (bisa dibaca langsung di GitHub)

📎 **[SETUP.pdf](SETUP.pdf)** — versi PDF (untuk offline reading / print)

Isi dokumen setup:

1. **Custom App + Admin API Access Token** — bikin app di Shopify Admin, aktifkan 5 scope
2. **Webhook `orders/create`** — config webhook + copy signing secret
3. **Theme Customization** — banner "Voucher DKRC Aktif" + error copy override
4. **Format Kirim Balik Credentials** — template reply ke DKRC team
5. **Testing End-to-End** — checklist 9 skenario

---

## 📤 Format Kirim Balik Credentials

Setelah setup 1 + 2 selesai, reply dengan format berikut:

```env
SHOPIFY_STORE_DOMAIN=duraking.myshopify.com
SHOPIFY_ADMIN_TOKEN=shpat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SHOPIFY_API_VERSION=2025-01
SHOPIFY_WEBHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

DKRC team akan input ke environment variables Vercel production → feature
aktif dalam **~5-10 menit** setelah credentials diterima.

---

## ✅ Testing End-to-End

Setelah env aktif, test bareng di ruangan yang sama (~30 menit):

- [ ] Generate voucher dari app DKRC → dapat kode `MBR-XXXXXX`
- [ ] Kode di-apply di cart dengan subtotal **≥ Rp 100.000** → 10% off applied
- [ ] Kode di-apply dengan subtotal **< Rp 100.000** → gagal ("minimum Rp 100.000")
- [ ] Banner "Voucher DKRC Aktif" muncul di cart & checkout
- [ ] Complete checkout dengan test payment (Shopify Bogus Gateway)
- [ ] Cek webhook fire di Shopify Admin → status Success (200 OK)
- [ ] Kode yang sama dipakai lagi → gagal (`usage_limit=1`)
- [ ] Cek Shopify Admin → `Discounts` → kode `MBR-*` muncul dengan usage stats
- [ ] Cek database DKRC: kolom `used_at` ter-update untuk voucher yang dipakai

Detail lengkap ada di [SETUP.md](SETUP.md#5-testing-end-to-end).

---

## ❓ FAQ

<details>
<summary><strong>Apakah butuh coding di sisi Shopify?</strong></summary>

Untuk setup #1 (Custom App) dan #2 (Webhook): **tidak**, cukup klik-klik di UI
Shopify Admin. Untuk setup #3 (Theme customization): butuh sedikit Liquid +
JavaScript untuk banner. Sisanya sudah di-handle di sisi DKRC (Next.js).
</details>

<details>
<summary><strong>Kalau minimum order Rp 100k mau diganti nanti?</strong></summary>

Cukup ubah 1 konstanta di kode DKRC (`MIN_ORDER_IDR` di
`app/api/vouchers/generate/route.ts`) lalu redeploy. **Tidak perlu setup ulang
apapun di Shopify** — parameter dikirim per-request saat generate voucher.
</details>

<details>
<summary><strong>Kalau mau minimum berbeda per tier (Silver/Gold/Platinum)?</strong></summary>

Bisa. Cukup ubah kode DKRC (baca `profile.tier` di route generate, mapping ke
nilai minimum yang beda). Zero perubahan di sisi Shopify. Marketplace team
juga tidak perlu re-setup apapun.
</details>

<details>
<summary><strong>Kalau kode voucher bocor ke non-member?</strong></summary>

4 lapisan proteksi mencegah abuse:
1. **Rate limit per member** — 1 voucher aktif per 2 jam per member
2. **Shopify `usage_limit=1`** — kode mati setelah 1x pakai oleh siapapun
3. **Shopify `once_per_customer=true`** — 1 customer hanya bisa pakai 1x
4. **Auto expire 2 jam** — kode invalid otomatis setelah 2 jam

Walaupun kode di-share ke 100 orang, hanya 1 orang pertama yang berhasil pakai.
</details>

<details>
<summary><strong>Bagaimana kalau signing secret bocor?</strong></summary>

Signing secret hanya bisa dipakai untuk **impersonate Shopify** (kirim fake
webhook ke endpoint DKRC). Efek maksimum: attacker bisa mark voucher sebagai
used tanpa order real — merugikan diri sendiri, tidak menguntungkan.

Kalau tetap khawatir bocor: delete webhook di Shopify Admin lalu bikin ulang
(auto-generate secret baru), kirim update ke DKRC team.
</details>

<details>
<summary><strong>Rotate token best practice?</strong></summary>

Rotate `SHOPIFY_ADMIN_TOKEN` tiap 90 hari:
1. Bikin Custom App baru dengan scope yang sama
2. Kirim token baru ke DKRC team
3. Uninstall Custom App yang lama
</details>

---

## 📞 Kontak

- **PIC DKRC Engineering:** [nama] — WA/Slack `[handle]`
- **Repo app DKRC:** private (internal)
- **Preview URL fitur:** `https://app.dkrc.id/dashboard/member/voucher`
  _(hanya bisa diakses member yang login — untuk testing sebelum launch publik)_

Pertanyaan / issue apapun, chat langsung di grup ini atau buka
[Issue di repo ini](../../issues).

---

## 📄 Lisensi

Dokumentasi ini bersifat internal DKRC — free to reference untuk keperluan
integrasi resmi dengan Duraking / partner Shopify. Tidak untuk redistribusi
publik.
