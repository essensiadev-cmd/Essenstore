# Essenstore

PWA manajemen bisnis retail **gudang → pelanggan** (tanpa toko fisik) yang menjual lewat marketplace (Shopee, Tokopedia, TikTok Shop, Lazada, Blibli) dan dikirim memakai ekspedisi marketplace.

**Fitur:** login Firebase (Email/Password) · peran admin/staff/pending · produk & stok gudang (foto di Firebase Storage) · pesanan marketplace dengan alur `baru → dikemas → dikirim → selesai` (+ retur/batal) · stok otomatis berkurang saat dikemas & kembali saat retur/batal · riwayat mutasi stok · dasbor omzet, laba bersih & stok menipis per marketplace · bisa di-install (PWA) & jalan offline untuk tampilan.

Struktur repo: `index.html` (seluruh aplikasi), `manifest.json`, `service-worker.js`, `icons/`, `.nojekyll`, `README.md`.

## 1. Setup Firebase (sekali saja)
1. **Authentication** → Sign-in method → Email/Password *(sudah aktif)*. Tab **Settings → Authorized domains** → tambahkan `USERNAME.github.io`.
2. **Firestore Database** → Rules → ganti rules default (`if false`) dengan:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{d}/documents {
    function signed() { return request.auth != null; }
    function mine() { return get(/databases/$(d)/documents/users/$(request.auth.uid)).data; }
    function ok() { return signed() && exists(/databases/$(d)/documents/users/$(request.auth.uid)) && mine().role in ['admin','staff']; }
    function adm() { return ok() && mine().role == 'admin'; }
    match /users/{u} {
      allow read: if signed();
      allow create: if request.auth.uid == u && (request.resource.data.role == 'pending' ||
        (request.resource.data.role == 'admin' && !exists(/databases/$(d)/documents/meta/bootstrap) && existsAfter(/databases/$(d)/documents/meta/bootstrap)));
      allow update, delete: if adm();
    }
    match /meta/{m} { allow read: if signed(); allow create: if signed() && !exists(/databases/$(d)/documents/meta/$(m)); }
    match /products/{p} { allow read, create, update: if ok(); allow delete: if adm(); }
    match /orders/{o} { allow read, create, update: if ok(); allow delete: if adm(); }
    match /moves/{m} { allow read, create: if ok(); }
  }
}
```
3. **Storage** → Get started (butuh paket Blaze; ada kuota gratis) → Rules:
```
rules_version = '2';
service firebase.storage {
  match /b/{b}/o {
    match /products/{f} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.resource.size < 2 * 1024 * 1024 && request.resource.contentType.matches('image/.*');
    }
  }
}
```
   Tanpa Storage, aplikasi tetap jalan; hanya foto produk yang tidak terunggah.
4. Pastikan `cfg` di bagian atas script `index.html` sama persis dengan **Project settings → Your apps**.

## 2. Build & test lokal
Tidak ada proses build. Jalankan `python3 -m http.server 8000` di folder ini, buka `http://localhost:8000` (tambahkan `localhost` di Authorized domains).

## 3. Release ke GitHub Pages
1. Upload semua file (termasuk `.nojekyll` dan folder `icons`) ke repo.
2. Repo → Settings → Pages → Deploy from a branch → `main` / root.
3. Buka `https://USERNAME.github.io/NAMA-REPO/`.
4. **Akun pertama yang mendaftar otomatis jadi admin.** Daftar sendiri dulu sebelum membagikan link. Akun berikutnya berstatus `pending` sampai admin mengubahnya jadi `staff` di tab **Tim**.

## 4. Alur operasional harian
1. Barang datang → Produk → **+Stok**.
2. Pesanan masuk di marketplace → **+ Pesanan marketplace** (isi no. pesanan, pembeli, ekspedisi; biaya admin % otomatis, bisa diubah).
3. **Kemas** (stok berkurang) → **Kirim + resi** → **Selesai** (atau **Retur** → stok kembali).
4. Pantau dasbor: omzet, laba (omzet − biaya admin − HPP), stok menipis.

## 5. Checklist rilis
- [ ] Rules Firestore & Storage sudah diganti · [ ] domain GitHub masuk Authorized domains
- [ ] Daftar akun admin · [ ] tes alur: produk → pesanan → kemas → kirim → selesai & retur
- [ ] Tes install PWA di HP · [ ] naikkan `CACHE` di `service-worker.js` (mis. `essenstore-v1.0.1`) setiap update
- [ ] Beri tag rilis di GitHub (`v1.0.0`)

## Catatan
Biaya admin tiap marketplace di aplikasi hanya nilai awal (Shopee 8%, Tokopedia/TikTok 6%, Lazada 7%, Blibli 5%); sesuaikan dengan kategori & program Anda. Ongkir diasumsikan dibayar pembeli/ditangani marketplace. Pesanan belum otomatis tersinkron dari marketplace (butuh API resmi/seller center masing-masing); input dilakukan manual atau bisa dikembangkan lanjut.
