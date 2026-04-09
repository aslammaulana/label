# Label Maker — Implementasi Lengkap

## Gambaran Umum

Sistem ini adalah aplikasi **Next.js** yang mengotomatiskan pembuatan label harga dalam jumlah banyak. Pengguna cukup menyiapkan:

1. **Template PNG** — label kosong dengan area yang sudah ditentukan
2. **File JSON** — berisi daftar produk (`nama`, `origin_price`, `latest_price`)
3. **Folder `image-products/`** — foto produk dengan nama file yang sesuai

Lalu dengan satu klik, sistem akan men-*generate* semua label sekaligus dan menghasilkan file ZIP yang bisa langsung diunduh.

---

## Struktur Proyek

```
test-label/
├── images/
│   ├── label-template.png          ← template label kosong
│   └── MODIS SOFA RC ....png       ← contoh output
├── image-products/                 ← foto produk (nama file = nama produk)
│   └── MODIS SOFA RC CALIFORNIA PAGANI CHAMPIGNON.png
├── content_export.json             ← data produk (sumber utama)
└── label-maker/                    ← aplikasi Next.js
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx                ← halaman utama (UI)
    │   └── api/
    │       └── generate/
    │           └── route.ts        ← API endpoint generate label
    ├── public/
    │   └── template/
    │       └── label-template.png  ← template yang dipakai server
    ├── fonts/                      ← font custom untuk rendering teks
    ├── package.json
    └── next.config.ts
```

---

## Struktur Data JSON

File `content_export.json` adalah sumber data utama. Format:

```json
[
  {
    "No": 1,
    "nama": "MODIS SOFA RC CALIFORNIA PAGANI CHAMPIGNON",
    "origin_price": 10878000,
    "latest_price": 7832000
  },
  {
    "No": 2,
    "nama": "MODIS SOFA DAWSON L",
    "origin_price": 17760000,
    "latest_price": 12787000
  }
]
```

> **Konvensi nama file gambar produk:**
> Nama file di `image-products/` harus **sama persis** dengan field `nama` di JSON, termasuk huruf kapital (case-sensitive).
>
> | Nama di JSON | Nama File Gambar |
> |---|---|
> | `MODIS SOFA RC CALIFORNIA PAGANI CHAMPIGNON` | `MODIS SOFA RC CALIFORNIA PAGANI CHAMPIGNON.png` |
> | `MODIS SOFA DAWSON L` | `MODIS SOFA DAWSON L.jpg` |
>
> Ekstensi yang didukung secara otomatis: `.png`, `.jpg`, `.jpeg`, `.webp`

---

## Teknologi yang Digunakan

| Kebutuhan | Package | Keterangan |
|---|---|---|
| Generate & edit gambar | `sharp` | Komposisi gambar resolusi tinggi, tanpa lossy |
| Render teks di atas gambar | `@napi-rs/canvas` | Canvas API di Node.js, support font custom |
| Zip hasil output | `jszip` | Bundle semua label jadi satu file ZIP |
| Format angka ke Rupiah | Native `Intl.NumberFormat` | Tidak perlu package tambahan |

### Instalasi Dependency

```bash
cd label-maker
npm install sharp @napi-rs/canvas jszip
npm install --save-dev @types/jszip
```

> **Catatan:** Gunakan `@napi-rs/canvas` bukan `node-canvas` karena lebih mudah diinstall di Windows (tidak perlu build tools C++).

---

## Konfigurasi Koordinat Template

Ini adalah bagian paling kritis. Setiap area teks dan gambar di template harus didefinisikan koordinatnya dalam pixel. Koordinat diukur dari pojok kiri atas template.

Buat file `label-maker/lib/template-config.ts`:

```typescript
export const TEMPLATE_CONFIG = {
  // Ukuran template (sesuaikan dengan label-template.png)
  width: 1800,
  height: 1200,

  // Area foto produk
  productImage: {
    x: 60,          // jarak dari kiri
    y: 100,         // jarak dari atas
    width: 700,     // lebar kotak foto
    height: 700,    // tinggi kotak foto
    fit: "contain" as const, // 'contain' | 'cover' | 'fill'
  },

  // Nama produk
  productName: {
    x: 800,
    y: 150,
    maxWidth: 900,
    fontSize: 52,
    fontFamily: "Geist",
    color: "#1a1a1a",
    lineHeight: 1.3,
  },

  // Harga coret (origin_price)
  originPrice: {
    x: 800,
    y: 650,
    fontSize: 44,
    fontFamily: "Geist",
    color: "#888888",
    strikethrough: true,
  },

  // Harga terbaru (latest_price)
  latestPrice: {
    x: 800,
    y: 750,
    fontSize: 72,
    fontFamily: "Geist Bold",
    color: "#e63946",
  },
};
```

> **Cara kalibrasi koordinat:** Buka `label-template.png` di editor gambar (misal Paint, Photoshop, atau bahkan browser DevTools), hover ke area target, dan catat koordinat pixel-nya.

---

## Implementasi API Route

Buat file `label-maker/app/api/generate/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";
import path from "path";
import fs from "fs/promises";
import sharp from "sharp";
import { createCanvas, registerFont, loadImage } from "@napi-rs/canvas";
import JSZip from "jszip";
import { TEMPLATE_CONFIG } from "@/lib/template-config";

// --- Register Font Geist ---
try {
  // Pastikan Anda sudah mengunduh file font TTF ke folder /fonts di dalam project
  registerFont(path.join(process.cwd(), "fonts", "Geist-Regular.ttf"), { family: "Geist" });
  registerFont(path.join(process.cwd(), "fonts", "Geist-Bold.ttf"), { family: "Geist Bold" });
} catch (e) {
  console.warn("⚠️ Font Geist tidak ditemukan! Silakan download dari Google Fonts dan masukkan ke folder fonts/.");
}

// Format angka ke Rupiah
function formatRupiah(amount: number): string {
  return new Intl.NumberFormat("id-ID", {
    style: "currency",
    currency: "IDR",
    minimumFractionDigits: 0,
  }).format(amount);
}

// Fungsi utama generate satu label
async function generateLabel(item: {
  nama: string;
  origin_price: number;
  latest_price: number;
}): Promise<Buffer> {
  const cfg = TEMPLATE_CONFIG;

  // 1. Load template
  const templatePath = path.join(process.cwd(), "public", "template", "label-template.png");
  const templateBuffer = await fs.readFile(templatePath);
  const templateMeta = await sharp(templateBuffer).metadata();

  // 2. Buat canvas untuk teks
  const canvas = createCanvas(templateMeta.width!, templateMeta.height!);
  const ctx = canvas.getContext("2d");

  // 3. Gambar nama produk (dengan word wrap)
  ctx.font = `${cfg.productName.fontSize}px ${cfg.productName.fontFamily}`;
  ctx.fillStyle = cfg.productName.color;
  const words = item.nama.split(" ");
  let line = "";
  let lineY = cfg.productName.y;
  for (const word of words) {
    const testLine = line + word + " ";
    const { width } = ctx.measureText(testLine);
    if (width > cfg.productName.maxWidth && line !== "") {
      ctx.fillText(line.trim(), cfg.productName.x, lineY);
      line = word + " ";
      lineY += cfg.productName.fontSize * cfg.productName.lineHeight;
    } else {
      line = testLine;
    }
  }
  ctx.fillText(line.trim(), cfg.productName.x, lineY);

  // 4. Gambar harga coret
  ctx.font = `${cfg.originPrice.fontSize}px ${cfg.originPrice.fontFamily}`;
  ctx.fillStyle = cfg.originPrice.color;
  const originText = formatRupiah(item.origin_price);
  ctx.fillText(originText, cfg.originPrice.x, cfg.originPrice.y);
  if (cfg.originPrice.strikethrough) {
    const textWidth = ctx.measureText(originText).width;
    ctx.strokeStyle = cfg.originPrice.color;
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(cfg.originPrice.x, cfg.originPrice.y - cfg.originPrice.fontSize / 3);
    ctx.lineTo(cfg.originPrice.x + textWidth, cfg.originPrice.y - cfg.originPrice.fontSize / 3);
    ctx.stroke();
  }

  // 5. Gambar harga terbaru
  ctx.font = `bold ${cfg.latestPrice.fontSize}px ${cfg.latestPrice.fontFamily}`;
  ctx.fillStyle = cfg.latestPrice.color;
  ctx.fillText(formatRupiah(item.latest_price), cfg.latestPrice.x, cfg.latestPrice.y);

  // 6. Export canvas sebagai PNG buffer
  const textLayer = canvas.toBuffer("image/png");

  // 7. Cari foto produk
  const productImageDir = path.join(process.cwd(), "..", "image-products");
  const extensions = [".png", ".jpg", ".jpeg", ".webp"];
  let productImageBuffer: Buffer | null = null;

  for (const ext of extensions) {
    const imgPath = path.join(productImageDir, `${item.nama}${ext}`);
    try {
      productImageBuffer = await fs.readFile(imgPath);
      break;
    } catch {
      // coba ekstensi berikutnya
    }
  }

  // 8. Composite: template + foto produk + layer teks
  const composites: sharp.OverlayOptions[] = [];

  if (productImageBuffer) {
    const resizedProduct = await sharp(productImageBuffer)
      .resize(cfg.productImage.width, cfg.productImage.height, {
        fit: cfg.productImage.fit,
        background: { r: 255, g: 255, b: 255, alpha: 1 },
      })
      .png()
      .toBuffer();

    composites.push({
      input: resizedProduct,
      left: cfg.productImage.x,
      top: cfg.productImage.y,
    });
  }

  composites.push({ input: textLayer, left: 0, top: 0 });

  const outputBuffer = await sharp(templateBuffer)
    .composite(composites)
    .png({ compressionLevel: 6 })
    .toBuffer();

  return outputBuffer;
}

// Handler POST: terima JSON, kembalikan ZIP
export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const items: { nama: string; origin_price: number; latest_price: number }[] = body.items;

    if (!items || items.length === 0) {
      return NextResponse.json({ error: "Data produk kosong" }, { status: 400 });
    }

    const zip = new JSZip();

    // Generate semua label secara paralel
    await Promise.all(
      items.map(async (item) => {
        const buffer = await generateLabel(item);
        const fileName = `${item.nama}.png`;
        zip.file(fileName, buffer);
      })
    );

    const zipBuffer = await zip.generateAsync({ type: "nodebuffer" });

    return new NextResponse(zipBuffer, {
      status: 200,
      headers: {
        "Content-Type": "application/zip",
        "Content-Disposition": `attachment; filename="labels-${Date.now()}.zip"`,
      },
    });
  } catch (err) {
    console.error(err);
    return NextResponse.json({ error: "Gagal generate label" }, { status: 500 });
  }
}
```

---

## Implementasi UI (Halaman Utama)

Edit `label-maker/app/page.tsx`:

```typescript
"use client";

import { useState } from "react";

interface Product {
  No: number;
  nama: string;
  origin_price: number;
  latest_price: number;
}

export default function Home() {
  const [jsonText, setJsonText] = useState("");
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");

  // Parse JSON dari textarea
  const handleParseJSON = () => {
    try {
      const parsed = JSON.parse(jsonText);
      if (!Array.isArray(parsed)) throw new Error("Format harus array []");
      setProducts(parsed);
      setError("");
    } catch (e: unknown) {
      setError("JSON tidak valid: " + (e instanceof Error ? e.message : String(e)));
    }
  };

  // Load dari file JSON
  const handleFileLoad = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (ev) => {
      setJsonText(ev.target?.result as string);
    };
    reader.readAsText(file);
  };

  // Generate dan download ZIP
  const handleGenerate = async () => {
    if (products.length === 0) return;
    setLoading(true);
    try {
      const res = await fetch("/api/generate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ items: products }),
      });

      if (!res.ok) throw new Error("Gagal generate");

      const blob = await res.blob();
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `labels-${Date.now()}.zip`;
      a.click();
      URL.revokeObjectURL(url);
    } catch (e: unknown) {
      setError("Error: " + (e instanceof Error ? e.message : String(e)));
    } finally {
      setLoading(false);
    }
  };

  return (
    <main className="min-h-screen bg-gray-50 p-8">
      <div className="max-w-3xl mx-auto">
        <h1 className="text-3xl font-bold text-gray-800 mb-2">Label Maker</h1>
        <p className="text-gray-500 mb-8">Generate label harga otomatis dari data JSON</p>

        {/* Input JSON */}
        <div className="bg-white rounded-xl shadow p-6 mb-6">
          <h2 className="font-semibold text-lg mb-3">1. Upload atau Paste Data JSON</h2>
          <input
            type="file"
            accept=".json"
            onChange={handleFileLoad}
            className="mb-3 block w-full text-sm"
          />
          <textarea
            rows={8}
            value={jsonText}
            onChange={(e) => setJsonText(e.target.value)}
            placeholder='[{"nama":"NAMA PRODUK","origin_price":10000000,"latest_price":8000000}]'
            className="w-full border rounded-lg p-3 font-mono text-sm resize-y"
          />
          <button
            onClick={handleParseJSON}
            className="mt-3 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700"
          >
            Parse JSON
          </button>
          {error && <p className="mt-2 text-red-500 text-sm">{error}</p>}
        </div>

        {/* Preview daftar produk */}
        {products.length > 0 && (
          <div className="bg-white rounded-xl shadow p-6 mb-6">
            <h2 className="font-semibold text-lg mb-3">
              2. Preview Data ({products.length} produk)
            </h2>
            <div className="overflow-auto max-h-64">
              <table className="w-full text-sm">
                <thead className="bg-gray-100">
                  <tr>
                    <th className="text-left p-2">Nama</th>
                    <th className="text-right p-2">Harga Asli</th>
                    <th className="text-right p-2">Harga Terbaru</th>
                  </tr>
                </thead>
                <tbody>
                  {products.map((p, i) => (
                    <tr key={i} className="border-t">
                      <td className="p-2">{p.nama}</td>
                      <td className="p-2 text-right text-gray-400 line-through">
                        {p.origin_price.toLocaleString("id-ID")}
                      </td>
                      <td className="p-2 text-right font-semibold text-red-500">
                        {p.latest_price.toLocaleString("id-ID")}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          </div>
        )}

        {/* Tombol Generate */}
        <button
          onClick={handleGenerate}
          disabled={products.length === 0 || loading}
          className="w-full py-4 bg-green-600 text-white text-lg font-semibold rounded-xl hover:bg-green-700 disabled:opacity-40 disabled:cursor-not-allowed transition"
        >
          {loading
            ? `Sedang generate ${products.length} label...`
            : `Generate ${products.length} Label → Download ZIP`}
        </button>
      </div>
    </main>
  );
}
```

---

## Konfigurasi `next.config.ts`

Karena API Route mengakses folder di luar `label-maker/` (yaitu `image-products/`), tambahkan konfigurasi:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // Izinkan sharp digunakan di server
  experimental: {
    serverComponentsExternalPackages: ["sharp", "@napi-rs/canvas"],
  },
};

export default nextConfig;
```

---

## Langkah Penggunaan

### Setup Awal (1x saja)

```bash
# 1. Masuk ke folder aplikasi
cd label-maker

# 2. Install dependency
npm install sharp @napi-rs/canvas jszip

# 3. Salin template label dan Font
# - Salin template Anda ke: label-maker/public/template/label-template.png
# - Buat folder label-maker/fonts/ lalu masukkan file font:
#   1. Geist-Regular.ttf
#   2. Geist-Bold.ttf
#   (Font ini harus ada secara fisik (.ttf) agar bisa dipakai canvas di server)

# 4. Jalankan aplikasi
npm run dev
```

### Setiap Kali Generate Label

1. Pastikan foto produk sudah ada di `image-products/` dengan nama persis sama dengan field `nama` di JSON
2. Buka browser ke `http://localhost:3000`
3. Upload atau paste `content_export.json`
4. Klik **Parse JSON** → preview muncul
5. Klik **Generate → Download ZIP**
6. File ZIP berisi semua label PNG siap pakai

---

## Potensi Masalah & Solusinya

### 1. Koordinat Meleset
**Masalah:** Teks atau gambar tidak tepat di area yang diinginkan.  
**Solusi:** Buka `label-template.png` di editor gambar, catat koordinat pixel area target, update `TEMPLATE_CONFIG` di `lib/template-config.ts`.

### 2. Font Tidak Konsisten
**Masalah:** Font yang dirender berbeda dari template asli.  
**Solusi:** Copy file `.ttf` font yang digunakan ke folder `label-maker/fonts/`, lalu tambahkan di API route:
```typescript
registerFont(path.join(process.cwd(), "fonts", "font-name.ttf"), { family: "FontName" });
```

### 3. Foto Produk Tidak Ditemukan
**Masalah:** Nama file gambar tidak cocok dengan `nama` di JSON.  
**Solusi:** API akan tetap berjalan (label tetap dibuat tanpa foto). Pastikan naming convention konsisten. API sudah mencoba ekstensi `.png`, `.jpg`, `.jpeg`, `.webp` secara otomatis.

### 4. Performa Lambat untuk Batch Besar
**Masalah:** 100+ produk memakan waktu lama.  
**Solusi:** Sudah menggunakan `Promise.all` untuk paralelisme. Jika masih lambat, batasi concurrency menggunakan library `p-limit`.

### 5. Deploy ke Vercel Bermasalah
**Masalah:** `sharp` dan `@napi-rs/canvas` membutuhkan binary native yang tidak selalu tersedia di Vercel free tier.  
**Solusi alternatif:**
- Deploy ke **VPS** (DigitalOcean, Railway, dll.) → paling aman
- Gunakan **Docker** untuk memastikan environment konsisten
- Untuk skala kecil: jalankan locally saja sudah cukup

### 6. Resolusi Output
**Masalah:** Output PNG resolusinya turun.  
**Solusi:** `sharp` bekerja di level pixel, tidak ada downscale otomatis. Pastikan template yang digunakan sudah di resolusi final yang diinginkan (misal 300 DPI untuk cetak).

---

## Pengembangan Lanjutan (Opsional)

| Fitur | Deskripsi | Cara |
|---|---|---|
| **Visual Template Editor** | Geser elemen langsung di browser tanpa Photoshop | Lihat bagian di bawah |
| Preview di browser | Tampilkan preview 1 label sebelum download semua | Gunakan `<canvas>` atau `<img>` dengan blob URL dari `/api/preview` |
| Upload template via UI | Ganti template tanpa edit kode | API route `POST /api/template` + simpan ke `public/template/` |
| Progress bar | Tampilkan progress saat generate banyak produk | Server-Sent Events (SSE) dari API route |
| Format output PDF | Download sebagai PDF siap cetak | Tambahkan package `pdfkit` + embed semua PNG |
| Batch berdasarkan halaman | Tiap halaman A4 berisi N label | Komposisi multi-label per canvas besar |

---

## Visual Template Editor (Drag & Drop Positioning)

Fitur ini menghilangkan kebutuhan Photoshop untuk mengukur koordinat. User cukup geser elemen di browser, lalu koordinat otomatis tersimpan.

### Konsep Alur

```
Halaman /configure
    ↓
Preview label-template.png sebagai background
    ↓
Elemen (nama, harga asli, harga terbaru, foto) bisa di-drag
    ↓
Koordinat (x, y, w, h) tampil real-time
    ↓
Klik "Simpan Konfigurasi"
    ↓
POST /api/save-config → tulis ulang template-config.ts
    ↓
Halaman utama generate pakai koordinat baru ✅
```

### Library yang Digunakan

```bash
npm install react-rnd
```

`react-rnd` adalah library ringan yang menyediakan komponen `<Rnd>` — setiap elemen bisa di-drag dan di-resize secara visual di atas gambar template.

### Struktur File Baru

```
label-maker/
└── app/
    └── configure/
        └── page.tsx          ← halaman visual editor
└── app/api/
    └── save-config/
        └── route.ts          ← simpan koordinat ke template-config.ts
```

### Implementasi `app/configure/page.tsx`

```typescript
"use client";

import { useState } from "react";
import { Rnd } from "react-rnd";

// Nilai awal dari template-config.ts (bisa di-fetch dari API juga)
const DEFAULT_ELEMENTS = {
  productImage: { x: 60, y: 100, width: 680, height: 680 },
  productName:  { x: 800, y: 160, width: 920, height: 80 },
  originPrice:  { x: 800, y: 640, width: 600, height: 60 },
  latestPrice:  { x: 800, y: 750, width: 700, height: 100 },
};

type ElementKey = keyof typeof DEFAULT_ELEMENTS;

export default function ConfigurePage() {
  const [elements, setElements] = useState(DEFAULT_ELEMENTS);
  const [saving, setSaving] = useState(false);
  const [saved, setSaved] = useState(false);

  const update = (key: ElementKey, data: Partial<typeof elements[ElementKey]>) => {
    setElements((prev) => ({ ...prev, [key]: { ...prev[key], ...data } }));
    setSaved(false);
  };

  const handleSave = async () => {
    setSaving(true);
    await fetch("/api/save-config", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(elements),
    });
    setSaving(false);
    setSaved(true);
  };

  const LABELS: Record<ElementKey, string> = {
    productImage: "📷 Foto Produk",
    productName: "🔤 Nama Produk",
    originPrice: "💸 Harga Asli",
    latestPrice: "🏷 Harga Terbaru",
  };

  const COLORS: Record<ElementKey, string> = {
    productImage: "rgba(59,130,246,0.3)",
    productName: "rgba(16,185,129,0.3)",
    originPrice: "rgba(156,163,175,0.3)",
    latestPrice: "rgba(239,68,68,0.3)",
  };

  return (
    <main className="min-h-screen bg-zinc-950 text-white p-8">
      <h1 className="text-2xl font-semibold mb-2">Template Editor</h1>
      <p className="text-zinc-400 text-sm mb-6">
        Geser dan resize setiap elemen di atas template. Klik Simpan jika sudah sesuai.
      </p>

      {/* Canvas Editor */}
      <div
        className="relative border border-zinc-700 overflow-hidden mx-auto"
        style={{ width: 900, height: 600 }}
      >
        {/* Background template (diskalakan agar muat di layar) */}
        {/* eslint-disable-next-line @next/next/no-img-element */}
        <img
          src="/template/label-template.png"
          alt="template"
          className="absolute inset-0 w-full h-full object-contain pointer-events-none"
        />

        {/* Elemen draggable */}
        {(Object.keys(elements) as ElementKey[]).map((key) => {
          const el = elements[key];
          // Faktor skala: template asli misal 1800x1200, preview 900x600 = skala 0.5
          const scale = 0.5;
          return (
            <Rnd
              key={key}
              position={{ x: el.x * scale, y: el.y * scale }}
              size={{ width: el.width * scale, height: el.height * scale }}
              onDragStop={(_, d) => update(key, { x: Math.round(d.x / scale), y: Math.round(d.y / scale) })}
              onResizeStop={(_, __, ref, ___, pos) =>
                update(key, {
                  x: Math.round(pos.x / scale),
                  y: Math.round(pos.y / scale),
                  width: Math.round(parseInt(ref.style.width) / scale),
                  height: Math.round(parseInt(ref.style.height) / scale),
                })
              }
              bounds="parent"
              className="border-2 border-dashed border-white/60 cursor-move"
              style={{ background: COLORS[key] }}
            >
              <span className="text-xs font-semibold text-white drop-shadow px-1">
                {LABELS[key]}
              </span>
            </Rnd>
          );
        })}
      </div>

      {/* Koordinat real-time */}
      <div className="mt-6 grid grid-cols-2 md:grid-cols-4 gap-3 text-xs font-mono">
        {(Object.keys(elements) as ElementKey[]).map((key) => {
          const el = elements[key];
          return (
            <div key={key} className="bg-zinc-900 rounded-lg p-3 border border-zinc-800">
              <div className="font-semibold text-white mb-1">{LABELS[key]}</div>
              <div className="text-zinc-400">x: {el.x} | y: {el.y}</div>
              <div className="text-zinc-400">w: {el.width} | h: {el.height}</div>
            </div>
          );
        })}
      </div>

      {/* Tombol Simpan */}
      <button
        onClick={handleSave}
        disabled={saving}
        className="mt-6 px-6 py-3 bg-white text-zinc-900 font-semibold rounded-xl hover:bg-zinc-200 disabled:opacity-50 transition-colors"
      >
        {saving ? "Menyimpan..." : saved ? "✅ Tersimpan!" : "Simpan Konfigurasi"}
      </button>
    </main>
  );
}
```

### Implementasi `app/api/save-config/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";
import path from "path";
import fs from "fs/promises";

export async function POST(req: NextRequest) {
  try {
    const body = await req.json();
    const { productImage, productName, originPrice, latestPrice } = body;

    const content = `// Auto-generated oleh Visual Template Editor
// Terakhir disimpan: ${new Date().toLocaleString("id-ID")}

export const TEMPLATE_CONFIG = {
  productImage: {
    x: ${productImage.x},
    y: ${productImage.y},
    width: ${productImage.width},
    height: ${productImage.height},
    fit: "contain" as const,
    background: { r: 255, g: 255, b: 255, alpha: 0 },
  },
  productName: {
    x: ${productName.x},
    y: ${productName.y},
    maxWidth: ${productName.width},
    fontSize: 50,
    fontFamily: "Geist Sans",
    fontWeight: "normal" as const,
    color: "#1a1a1a",
    lineHeight: 1.35,
  },
  originPrice: {
    x: ${originPrice.x},
    y: ${originPrice.y},
    fontSize: 42,
    fontFamily: "Geist Sans",
    color: "#999999",
    strikethrough: true,
    strikeColor: "#999999",
    strikeLineWidth: 2,
  },
  latestPrice: {
    x: ${latestPrice.x},
    y: ${latestPrice.y},
    fontSize: 72,
    fontFamily: "Geist Sans",
    fontWeight: "bold" as const,
    color: "#e63946",
  },
} as const;
`;

    const configPath = path.join(process.cwd(), "lib", "template-config.ts");
    await fs.writeFile(configPath, content, "utf-8");

    return NextResponse.json({ success: true });
  } catch (err) {
    return NextResponse.json({ error: String(err) }, { status: 500 });
  }
}
```

### Catatan Penting

> [!NOTE]
> **Faktor Skala:** Template asli mungkin berukuran besar (misal 1800×1200 px). Preview di browser dikecilkan dengan faktor skala (misal `0.5`). Semua koordinat yang disimpan sudah dikonversi kembali ke ukuran asli secara otomatis (`d.x / scale`).

> [!TIP]
> **Navigasi:** Tambahkan link di `app/page.tsx` ke halaman `/configure` agar user mudah berpindah antara editor dan generator.

> [!WARNING]
> **Hot-reload konflik:** Karena `save-config` menulis ulang `template-config.ts`, Next.js dev server akan otomatis me-restart modul (hot module replacement). Ini normal — generate berikutnya akan langsung pakai koordinat baru.

