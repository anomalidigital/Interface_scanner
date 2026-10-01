# Interface_scanner

Halaman scan untuk platform LOL Photobooth: tamu mengarahkan kamera ke **strip
foto** atau **QR** mereka. Kamera mendeteksi kartunya secara langsung dan
memberi animasi "sedang di-scan" (motion **Proyeksi**, warna `#ED4835`) yang
menempel di permukaan kartu.

> **Tahap sekarang: tampilan.** Ada tutorial di awal, layar kamera layar
> penuh dengan animasi scan, dan kalimat ajakan. Belum ada tombol jepret
> (`CAPTURE_ENABLED = false`) dan QR belum dibaca.

![Tutorial: tangan memegang HP mendekati kartu foto, kartu menyala merah lalu dicentang](docs/demo-tutorial.webp)

Video: [tutorial](docs/demo-tutorial.mp4) · [perbandingan motion yang sempat dicoba](docs/demo-motions.mp4).

## Isi repo

| Berkas | Untuk apa |
| --- | --- |
| `SCAN.html` | **Halaman yang ditempel ke CMS.** Satu blok gaya, satu `<main>` (sintaks Angular), satu blok skrip. Styling memakai Bootstrap 5.3 yang sudah dimuat platform. Aset tutorial sudah disematkan (WebP base64), jadi tetap satu berkas. |
| `preview.html` | Harness pengembangan (Angular → Vue), sama seperti di template Gemini. **Jangan diunggah ke platform.** |
| `index.html` | Pengalih ke `preview.html` (untuk GitHub Pages). |
| `assets/` | Aset tutorial dalam WebP (tangan + HP, kartu foto, bayangan) — salinan yang disematkan di `SCAN.html`. |
| `test/` | Foto contoh untuk kamera tiruan. |
| `docs/` | Rekaman tutorial dan motion. |

## Menjalankan

```bash
python -m http.server 5173
```

Lalu buka http://localhost:5173/preview.html — langsung memakai webcam.
Di HP: buka lewat GitHub Pages repo ini (kamera HP hanya bisa lewat **https**).

| Query | Gunanya |
| --- | --- |
| `?fake=test/kartu-meja-kayu.jpg` | Kamera tiruan dari sebuah foto (laptop tanpa kamera, pengujian). Foto lain: `kartu-alas-gelap.jpg`, `kartu-meja-putih.jpg`, `kartu-tegak.jpg` (tegak, cocok untuk ukuran HP). |
| `&shake=2` | Besar goyangan tangan tiruan (px). |
| `&walk` | Kartu keluar-masuk frame tiap 10 detik. |
| `?notutorial` | Tutorial tidak muncul saat halaman dibuka (untuk pengujian). |
| `?capture` | Menyalakan alur jepret → hasil untuk dicoba lewat `__state.capture()` di console. |
| `?IsUpload` | Tombol "Unggah foto" di layar kamera (sama dengan halaman Gemini). |
| `?noworker` | Menguji jalur cadangan: deteksi di thread utama. |

Di console: `__state` (state halaman), `__SCAN` (mesin), `__SCAN_BUILD` (versi).

## Tutorial

Muncul begitu halaman dibuka (`TUTORIAL_ON_START`), di atas kamera yang
diredupkan dan diburamkan supaya fokus. Animasi garis **tanpa teks**, memakai
aset dari tim desain (`Tutorial/Asset motion tutorial-02.png` dan `-03.png`,
dikonversi ke WebP), berulang terus (~6,2 detik per putaran):

1. kartu foto tergeletak, sedikit "bernapas";
2. tangan memegang HP datang dari kanan bawah lewat lintasan melengkung, makin
   turun mendekati kartu (skala dan bayangannya mengecil), lalu mendarat
   dengan pegas halus;
3. layar HP memperlihatkan kartu di bawahnya; titik cahaya "proyeksi" merah
   menyebar dari tengah kartu lalu menyatu jadi isian;
4. lencana centang muncul memantul;
5. HP ancang-ancang turun sedikit, lalu terangkat pergi ke kiri atas.

Kurva geraknya cubic-bezier + pegas (seperti graph editor After Effects),
digerakkan `requestAnimationFrame` — bukan CSS `@keyframes`, yang diam di
platform. Supaya kartu di belakang tertutup rapi, siluet tangan + HP diisi
gelap dan layarnya dibuat tembus pandang (diolah dari PNG aslinya).

**Lewati** ada di kanan, tepat di bawah animasinya, dan langsung menutup
tutorial. Tombol **?** di pojok kanan bawah layar kamera membukanya lagi.

## Layar kamera

- Video **layar penuh** (tanpa bingkai hitam) di HP tegak/miring, tablet, dan
  desktop. Tidak ada judul dan tidak ada tombol jepret.
- Kartu yang terdeteksi diberi animasi **Proyeksi**: matriks titik halftone
  yang ukurannya mengikuti interferensi dua gelombang — seperti cahaya
  terstruktur pada mesin scan 3D. Hanya isian, tanpa garis tepi, warna
  `#ED4835`; sekelilingnya sedikit diredupkan. Animasinya digambar shader WebGL
  yang memetakan tiap piksel layar balik ke permukaan kartu, jadi ikut miring
  bersama kartunya.
- Di bawah ada ajakan **"Scan strip foto atau QR kamu"** (`copy.prompt`), dan
  tombol **?** di kanan bawah.

Kalau perangkat meminta animasi dikurangi (`prefers-reduced-motion`), animasi
berhenti di satu bingkai. Kalau WebGL tidak tersedia, kartu cukup diberi isian
warna polos.

## Alur lengkap (saat jepret dinyalakan lagi)

```
kamera ──jepret──> hasil ──"Atur sudut"──> atur sudut ──> hasil
```

- **Hasil** — kertas yang sudah lurus. Tampilan: *Asli*, *Bersih* (bawaan:
  bayangan dan warna lampu dihilangkan, kertas jadi putih), *Hitam putih*.
  Ada *Putar*, *Atur sudut*, *Scan lagi*, *Simpan* (di HP membuka lembar
  bagikan, jadi bisa "Simpan gambar" ke galeri).
- **Atur sudut** — geser empat titik sudut (ada kaca pembesar). Muncul sendiri
  kalau kertas tidak ketemu sama sekali.
- Saat jepret: kilau di kertas, lalu kertas yang sudah lurus "terangkat" dari
  foto dan mendarat di layar hasil.

## Cara kerja (hasil riset)

1. Frame kamera dikecilkan ke 384 px lalu dicari 4 sudut kertasnya dengan
   [scanic](https://github.com/marquaye/scanic) (MIT) memakai detektor ML
   DocCornerNet (~2 MB, diunduh sekali). Deteksinya berjalan di **Web Worker**,
   jadi animasi tetap mulus.
2. Saat jepret, deteksi diulang di foto resolusi penuh, lalu **sudutnya
   ditajamkan**: di sekitar tiap sisi dicari tepi kertas yang sebenarnya,
   ditarik garis lurus, dan garis-garisnya dipotongkan.
3. **Rasio asli kertas** ditebak dari perspektif foto (Zhang & He, 2007), jadi
   hasilnya tidak gepeng walau difoto miring. Bisa juga dikunci lewat
   `DOC_RATIO`.
4. Foto diluruskan (homografi + interpolasi bilinear), tepinya dipangkas 3 px
   supaya tidak ada garis sisa latar.
5. Filter *Bersih*: warna kertas diperkirakan per petak, lalu tiap piksel dibagi
   warna kertas di bawahnya.

### Perbandingan library

Diuji pada 24 foto: 15 foto asli dari repo scanic + 9 foto tiruan kondisi booth
(kartu di meja putih, meja kayu, meja berantakan, dipegang tangan, sudut
ekstrem, kartu kuning silau, kartu berbingkai cetak). Error = jarak sudut
terjauh dari posisi sebenarnya, dalam % diagonal foto.

| Metode | Pas (< 1%) | Meleset (1–2,5%) | Gagal | Waktu / frame | Unduhan |
| --- | --- | --- | --- | --- | --- |
| **scanic ML** (dipakai) | 19 | 2 | 3 | ~30 ms | ~2 MB model + 100 KB |
| jscanify (OpenCV.js) | 16 | 0 | 8 | ~70 ms | ~9 MB |
| scanic klasik (Canny) | 8 | 4 | 12 | ~110 ms | 100 KB |

- Penajaman sudut di atas scanic ML: median error 0,49% → 0,29%; pada kartu
  rata di meja 0,4–2,7% → sekitar 0,05%.
- Tebakan rasio kertas: median meleset 0,27% (2,8% kalau hanya memakai
  panjang sisi).
- Web Worker vs thread utama (desktop, 300 frame): frame tersendat > 50 ms
  turun dari 17 jadi 3.

Alternatif berbayar yang dicek: Dynamsoft (Mobile Web Capture / Document
Normalizer) dan Scanbot Web SDK — lebih lengkap dan ada dukungan, tapi
berlisensi. Google ML Kit Document Scanner (Android) dan VisionKit (iOS) hanya
untuk aplikasi native, bukan web.

### Supaya deteksi selalu kena saat acara

- Pakai **alas gelap atau kontras** di bawah kertas (matras hitam). Kertas
  putih di meja putih paling sulit.
- **Cahaya rata** — hindari lampu yang memantul langsung di kertas dan bayangan
  HP.
- Kertas **rata**. Kertas terlipat atau melengkung bisa menyisakan sedikit latar
  (bisa dibetulkan di *Atur sudut*).
- Jangan pegang kertas di sudutnya — jari menutupi sudut.
- Kalau kartunya dicetak sendiri, isi `DOC_RATIO` (mis. `4 / 6` untuk 4R) supaya
  rasio hasilnya persis.

## Konfigurasi

Semua ada di region **KONFIGURASI** paling atas di skrip `SCAN.html`.

| Nama | Isi |
| --- | --- |
| `BUILD` | Naikkan setiap kali berkas diubah, lalu cek `__SCAN_BUILD` di halaman live. |
| `ACCENT` | Warna cahaya scan dan aksen (`#ED4835`). |
| `CAPTURE_ENABLED` | `false` = belum ada jepret di tahap ini. |
| `TUTORIAL_ON_START` | Tutorial muncul saat halaman dibuka. |
| `SCANIC_URL`, `ML_OPTIONS` | Library deteksi. Untuk host sendiri: salin `dist/` scanic dan paket `scanic-ml` ke S3 (CORS `*`), lalu arahkan ke sana (`ML_OPTIONS.assetBaseUrl`). |
| `DOC_RATIO` | `"auto"` atau angka lebar / tinggi (`210 / 297` A4, `148 / 210` A5, `4 / 6` 4R). |
| `CAMERA_IDEAL` | Resolusi kamera yang diminta (bawaan 4K; browser memilih yang terdekat). |
| `OUTPUT_MAX`, `JPEG_QUALITY`, `DEFAULT_FILTER` | Ukuran, kualitas, dan tampilan awal hasil. |
| `ON_SCAN_READY` | Kait yang dipanggil setiap hasil siap — tempat mengirim hasil ke server, Gemini, atau cetak. |
| `this.copy`, `this.labels` | Semua tulisan di layar, termasuk ajakan `copy.prompt`. |

Di region **Penyetelan**: `OUTSIDE_DIM` (gelapnya area di luar kertas) dan
`LIGHT_SCALE_MAX` (resolusi kanvas cahaya).

## Aturan platform yang diikuti

Sama dengan template Gemini:

- Persis satu blok gaya, satu `<main>`, satu blok skrip; nama kedua tag itu
  tidak pernah ditulis utuh di komentar.
- Template Angular: `*ngIf`, `*ngFor`, `[ngClass]`, `[attr.*]`, `(click)`,
  `{{ }}`. Tanpa `ngModel`, `[class.x]`, atau `@if`.
- Semua `this.*` dipasang sinkron sebelum `await` pertama; state yang diubah dari
  callback async dibungkus `this.zone.run`.
- Semua class diawali `scn-`; Bootstrap dipakai lewat class dan variabel
  `--bs-btn-*`.
- Animasi tidak memakai CSS `@keyframes` / `[ngStyle]` (terbukti diam di
  platform): cahaya scan digambar WebGL lewat `requestAnimationFrame`.

## Belum ada / langkah berikutnya

- **Membaca QR** (ajakannya sudah menyebut QR): bisa memakai BarcodeDetector
  bawaan Chrome Android, dengan pustaka cadangan untuk iPhone.
- Rancang reaksi saat strip foto / QR berhasil terbaca, lalu tentukan alur
  sesudahnya (nyalakan lagi `CAPTURE_ENABLED` kalau perlu jepret).
- **Mode booth DSLR** (jembatan LOLBooth): foto Canon tinggal dilewatkan ke jalur
  yang sama dengan "Unggah foto" (deteksi → luruskan → hasil).
- **Unggah ke platform + QR**: sambungkan `ON_SCAN_READY` ke alur unggah dari
  template Gemini.
- **Host sendiri** scanic + model di S3 supaya tidak bergantung pada jsDelivr
  saat acara.

## Lisensi pihak ketiga

[scanic](https://github.com/marquaye/scanic) (MIT) dimuat dari jsDelivr saat
halaman dibuka; model DocCornerNet-nya juga MIT.
