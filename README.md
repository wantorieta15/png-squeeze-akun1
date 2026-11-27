# png-squeeze-01 — Akun 1 (@walta-exchange)

[![Make Fun](https://img.shields.io/badge/make.fun-walta--exchange-7c3aed)](https://make.fun/)
[![Node](https://img.shields.io/badge/node-22-green)](https://nodejs.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Program Node 22 (pure stdlib, tanpa install) yang menulis ulang 8 screenshot
1280x720 RGBA sebagai PNG lossless sekecil mungkin, dalam satu run.

## Cara kerja (`main.ts`, ditulis dari nol)

1. **Decode** PNG input ke RGBA (decoder sendiri, semua colour type 0/2/3/4/6).
2. **Analisa sekali jalan**: opaque? grey? warna distinct (berhenti di >256).
3. **Reduksi colour type** per gambar:
   - ≤256 warna → palet type 3 (urut by count, dan by luminance; entri
     transparan dulu supaya tRNS pendek);
   - grey + opaque → type 0; grey + alpha → type 4;
   - satu nilai transparan tunggal → type 0/2 + tRNS key;
   - opaque → type 2; alpha bervariasi → type 6.
4. **Filter per baris** dari kandidat {None, minsum, pairs, brute} — brute
   (zlib level 4 + dictionary 32 KB) hanya untuk bpp ≤ 2, karena di
   rich/RGBA gain-nya ~0 tapi biayanya besar.
5. **Sweep zlib final** 6 kombinasi {L9,L6} × {memLevel 8,9} × {strategi
   default, filtered}, ambil stream terkecil (strategi 2/3 tidak pernah
   dicoba — hasilnya katastrofis).
6. **Anggaran kerja deterministik** (hitung byte/langkah, bukan jam):
   degradasi berjenjang bila kerja melewati ambang. Dua run byte-identik
   (terverifikasi lokal). Satu chunk IDAT; hanya chunk IHDR/PLTE/tRNS/IDAT/IEND.
7. **P1 optimal-parse (AKTIF, hasil revisi 27 Nov)**: DEFLATE
   optimal-parse sendiri (hash-chain LZ77 + DP + dynamic Huffman, satu
   blok per gambar) — chain 64 dengan uji cepat "di luar terbaik"
   (kandidat gugur dalam 1 banding), tanpa berhenti di match 32, dan
   iterasi model ala zopfli (2 pass perbaikan, chain penuh) hanya bila
   sepadan: data besar (≥2 MB) dan 1-pass sudah menang >2.000 byte atas
   zlib. Palet dilewati (P1 tidak pernah menang di sana). Dua counter
   kerja terpisah (P0 vs P1) supaya kerja P1 tidak mematikan brute P0.
   P1 menang di i002/i003/i004/i006; i007/i008 tetap memakai zlib.

## Skor practice (scorer resmi kit)

| instance | tema | byte | poin |
|---|---|---|---|
| i001 | flat | 12.496 | 718,370 |
| i002 | rich | 51.601 | 708,578 |
| i003 | alpha | 104.537 | 687,397 |
| i004 | grey | 101.656 | 720,138 |
| i005 | flat | 16.552 | 742,251 |
| i006 | rich | 171.415 | 716,604 |
| i007 | alpha | 60.890 | 701,871 |
| i008 | grey | 24.897 | 714,158 |
| **total** | | **544.044** | **skor 713.671, passed 8/8** |

Waktu run: ~30 detik CPU di 1 core VM ini (batas 60 detik; P0 saja
~9,7 detik). Baseline rumah fixed-paeth: 420.399. Referensi
zopfli-like: ~760–790/instance (target 750.000 TIDAK tercapai —
P1 JS hanya sanggup ~+1,5–4% per gambar yang menang; lihat catatan
revisi di bawah).

## Catatan revisi P1 (27 Nov, Akun 1)

Yang dicoba reviser, terukur di practice set (1 core, `taskset -c 0`):

- **BUG KRITIS DIPERBAIKI — header Huffman dinamis korup:** di
  `dynamicHeaderBits`, cabang panjang non-nol menulis `i += run`
  SESUDAH `run` habis dipakai loop (bernilai 0), jadi indeks hanya maju
  1 dan ulangan panjang kode yang sama ditulis BERKALI-KALI ke header.
  Akibatnya SEMUA stream dynamic P1 builder TIDAK VALID (checker:
  "invalid literal/lengths set", "invalid bit length repeat") dan
  bernilai 0 poin — gain P1 yang dilaporkan builder (+0–3%) adalah
  angka stream korup yang tidak pernah lolos skor. Setelah diperbaiki
  (`i += 1 + run` sebelum loop), ke-8 output terverifikasi decode
  identik pikselnya dan skor P1 menjadi nyata: 713.671.
- P1 builder (chain 16, blok 128 KB, 1-pass DP): hanya i004 membaik
  (−984 byte), +13,7 dtk — model biaya dari greedy terbukti bias.
- Chain 128: +14 dtk tanpa gain tambahan — chain BUKAN pembatas.
- 2-pass DP (chain 64): i003 −2.705, i004 −4.868, i006 −5.810 byte,
  tapi total 52,9 dtk — DITOLAK (di atas batas aman 35 dtk).
- 4-pass DP: i006 −7.347, i003 −4.378, tapi i002 tetap 0 gain dan
  total ~70 dtk — DITOLAK.
- Blok 256 KB / 1 blok per gambar: 1 blok terbaik (i002 berbalik
  menang, i007 membaik 4.258 byte vs 128 KB).
- Perbaikan bug model Huffman (panjangkan kode jarang dulu) DICOBA dan
  DIBALIKKAN: tidak membantu i002/i007, malah merugikan i006 (−1.425).
- Pass perbaikan chain 16 DICOBA dan DIBUANG: 0 perbaikan (chain kecil
  membatasi match set; harus chain penuh 64).
- Urutan palet leksikografis/kemunculan pertama DICOBA dan DIBUANG:
  0 gain di i001/i005.
- Counter kerja P1/P0 DIPISAH (bug: kerja P1 sempat mematikan brute
  P0 untuk gambar berikutnya; i004 jatuh ke 109.665).

Kesimpulan: 750.000 butuh ~10% byte lebih kecil dari P0 di SEMUA
gambar — setara deflate referensi (zopfli ~15 iterasi + block split
berbasis biaya). Implementasi JS ini, dengan anggaran 35 dtk 1 core,
maksimal ~713.000–716.000 (konfigurasi penuh 2 pass tambahan di
i003+i006: ~716.000 tapi 35,4 dtk CPU — ditolak demi margin aman).

## Variasi vs Akun 2

Akun 1: filter kanal {0, minsum, pairs, brute(bpp≤2)}, palet luminance
{0, minsum, pairs}, sweep 6 kombinasi, brute dibatasi bpp≤2, P1 aktif
(chain 64, 1 blok/gambar, 2 pass perbaikan adaptif bila gain 1-pass
>2.000 byte dan data ≥2 MB; palet dilewati).

## Lisensi

MIT — lihat `LICENSE`.
