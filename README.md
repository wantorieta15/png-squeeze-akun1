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
7. **P1 optimal-parse (kode ada di repo, saat ini nonaktif)**: DEFLATE
   optimal-parse sendiri (hash-chain LZ77 + DP + dynamic Huffman + block
   split 128 KB) diukur hanya +0–3% byte untuk +19–38 detik — di batas
   60 detik dengan CPU grader 1 core, risiko timeout (skor 0) tidak
   sebanding, jadi dimatikan lewat `P1_WORK_CAP = 0`.

## Skor practice (scorer resmi kit)

| instance | tema | byte | poin |
|---|---|---|---|
| i001 | flat | 12.496 | 718,370 |
| i002 | rich | 51.852 | 705,539 |
| i003 | alpha | 106.527 | 675,101 |
| i004 | grey | 105.605 | 696,318 |
| i005 | flat | 16.552 | 742,251 |
| i006 | rich | 178.606 | 690,732 |
| i007 | alpha | 60.890 | 701,871 |
| i008 | grey | 24.897 | 714,158 |
| **total** | | **557.425** | **skor 705.542, passed 8/8** |

Waktu run: ~10 detik di VM ini (batas 60 detik). Baseline rumah fixed-paeth:
420.399. Referensi zopfli-like: ~760–790/instance.

## Variasi vs Akun 2

Akun 1: filter kanal {0, minsum, pairs, brute(bpp≤2)}, palet luminance
{0, minsum, pairs}, sweep 6 kombinasi, brute dibatasi bpp≤2, P1 nonaktif.
Akun 2 memakai parameter filter/sweep/optimal-parse yang berbeda.

## Lisensi

MIT — lihat `LICENSE`.
