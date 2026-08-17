# Rencana Lanjutan Development — SIPRO (lanjut dari repo djdjskjs/sipro)

Problem statement (verbatim):
> "saya ingin anda lanjutkan development dari repo ini https://github.com/djdjskjs/sipro — sebelumnya development terhenti di sini saya ingin anda lanjutkan"

Konteks status saat ini:
- Fase Marketing/Ads (acuan `docs/v2/30_MARKETING_INTEGRATION_SPEC.md`) **sudah terimplementasi** (backend + frontend) dan terbukti merender state **jujur** (“data belum lengkap”).
- Yang tersisa untuk menutup fase tersebut: **gate baru + uji-mutasi + E2E multi-peran**.
- Saat restore, baseline gates: **24 PASS, 2 FAIL** (harus dipulihkan dulu).

---

> **STATUS PELAKSANAAN (diperbarui akhir sesi ini):**
> - **FASE A — SELESAI**: baseline hijau dipulihkan (models.py dipecah ke `models_procurement.py`;
>   `verify_ia_v2` memakai **ledger pintu resmi** `docs/v2/40 §7`; temuan form E2 ditutup).
> - **FASE B — SELESAI**: Fase 43 ditutup. Gate ke-27 `verify_ads.py` + `mutasi_43.py`
>   (19 mutasi, 38/38 PASS) + E2E multi-peran + 3 cacat UI & 1 cacat data CAPI diperbaiki.
> - **FASE C — SELESAI**: Analitik & BI dibuka (`/bi`). POC `poc/poc_44.py` hijau; lapisan
>   metrik `backend/metrics/*` (47 metrik, 5 persona); `routers/analytics_router.py`;
>   snapshot harian yang bisa dihitung ulang; 6 tab UI (5 dashboard + Kamus Metrik);
>   gate ke-28 `verify_analytics.py` + `mutasi_44.py` (15 mutasi, 30/30 PASS).
>   `bash scripts/run_all_gates.sh` → **OVERALL PASS (28 gates)**.
> - **FASE D (Target & Budget/RAB) & FASE E (Konsolidasi Konstruksi) — BELUM DIKERJAKAN.**

## 1) Objectives
1. **Pulihkan baseline kualitas** sampai `bash scripts/run_all_gates.sh` kembali **OVERALL PASS (26 gates)**.
2. **Tutup Fase 43 (Kampanye & Biaya Iklan + Atribusi/CAPI)** dengan:
   - gate baru **`verify_ads.py`** (gate ke-27),
   - uji-mutasi **`mutasi_43.py`**,
   - 1 putaran **E2E multi-peran**.
3. Setelah fase 43 tertutup: lanjut fitur besar berikutnya berurutan:
   - **Analitik & BI** (docs/v2/31), lalu
   - **Target Proyek & Budget/RAB** (docs/v2/32), lalu
   - **Konsolidasi Proyek & Konstruksi** (docs/v2/29).

---

## 2) Implementation Steps

### FASE A — Pulihkan baseline hijau (wajib sebelum gate baru)
**Tujuan:** 26 gate existing kembali PASS tanpa menurunkan kualitas.

User stories (operasional kualitas):
1. Sebagai maintainer, saya bisa menjalankan `run_all_gates.sh` dan mendapat **OVERALL PASS** tanpa pengecualian.
2. Sebagai maintainer, saya dijaga dari **file bengkak**: model Pydantic tidak melewati batas NFR.
3. Sebagai maintainer, gate IA V2 tetap mencegah **menu sprawl**, tetapi **tidak kadaluwarsa** saat fitur resmi dibuka.
4. Sebagai pengguna DM, input biaya iklan harian menerima **angka** (bukan teks) sehingga validasi lebih jelas.
5. Sebagai maintainer, perubahan ini tidak memecah import lama (kompatibel ke router-router existing).

Langkah:
- A1. **Compliance fix:** pecah `backend/models.py` menjadi per-domain (mis. `models_core.py`, `models_procurement.py`, `models_tax.py`, dll.) dan **re-export** dari `models.py` agar import lama tetap aman. Target: setiap file <800 baris.
- A2. **IA gate fix yang benar:** ubah `scripts/verify_ia_v2.py` CHECK 3 dari “angka hardcoded 26” menjadi:
  - membandingkan jumlah + himpunan “pintu sidebar non-admin” dengan **ledger pintu resmi** yang terdokumentasi,
  - tetap ada **cap anti-sprawl** (mis. ledger berisi daftar pintu resmi; menambah pintu harus update ledger+docs).
  - tambah tabel **machine-readable** (mis. blok JSON) ke `docs/v2/40_PETA_NAV_V2.md` sebagai SSOT ledger.
- A3. **Form polish:** ubah `frontend/src/components/ads/SpendEntryDialog.js` field biaya menjadi `type="number"` (tetap format IDR di display).
- A4. Jalankan `bash scripts/run_all_gates.sh` sampai **OVERALL PASS (26)**.

Output:
- 26 gates PASS lagi (baseline).

---

### FASE B — Tutup Fase 43 (gates + mutasi + E2E)
**Tujuan:** mengunci Ads/Attribution/CAPI sebagai fitur “selesai” dengan guardrail bergigi.

#### Phase 1 — Core POC (isolasi, wajib)
**Core yang rawan gagal:** import spend **idempotent**, validasi dry-run, dan metrik **jujur**.

User stories (POC):
1. Sebagai DM, saya bisa melakukan **dry-run** impor CSV dan melihat baris valid vs ditolak beserta alasan.
2. Sebagai DM, mengimpor file yang sama 2× tidak membuat duplikasi (idempotensi natural key).
3. Sebagai analis, metrik tidak berbohong: periode tanpa biaya menampilkan status “biaya belum lengkap” (bukan 0).
4. Sebagai admin integrasi, saya bisa melihat mode platform (simulation/live) tanpa bocor nilai rahasia env.
5. Sebagai growth, event CAPI punya `event_id` dedup dan `user_data` ter-hash.

Langkah:
- B1. Jalankan/rapikan `poc/poc_43.py` (sudah ada) sebagai baseline dan pastikan stabil di DB seed.
- B2. Websearch singkat best-practice (CSV idempotent upsert + hashing PII SHA-256 untuk CAPI) bila perlu memperkuat validasi.

#### Phase 2 — V1 App Development (sudah ada, hanya hardening bila ditemukan gap)
User stories (E2E real):
1. Sebagai DM Supervisor, saya bisa membuat/ubah kampanye dan melihatnya tampil di tab Kampanye.
2. Sebagai staf DM, saya bisa input biaya harian manual dan melihatnya di tab Biaya.
3. Sebagai DM, saya bisa impor CSV via wizard: preview → commit → laporan.
4. Sebagai manajer, saya bisa melihat kinerja (Spend, Leads, CPL/CAC/ROAS) dengan warning “biaya belum lengkap” bila perlu.
5. Sebagai auditor, saya bisa melihat funnel atribusi + daftar event CAPI serta status transport.

#### Phase 3 — Gates + Uji-mutasi + E2E
- B3. Buat **`scripts/verify_ads.py`** dan registrasikan ke `scripts/run_all_gates.sh` sebagai gate ke-27.
  Gate memeriksa (pakai API nyata, bukan mock):
  - idempotensi import CSV (import 2× tidak menambah count),
  - dry-run menolak baris invalid dengan alasan,
  - honest metrics: tanpa biaya => flag/pesan “data biaya belum lengkap”, bukan 0,
  - RBAC endpoint ads: peran tanpa izin 403; peran berizin bukan 403,
  - nav: route live harus punya route; comingSoon tidak boleh punya `path`,
  - UI ads tidak hardcode enum/vocab (harus SSOT `reference_p43`),
  - `/api/ads/health` tidak membocorkan nilai env (hanya `filled/missing`).
- B4. Buat **`scripts/mutasi_43.py`** (8–10 mutasi) yang sengaja merusak:
  - upsert/index unik (idempotensi),
  - validasi dry-run,
  - honesty metrics (palsukan 0),
  - RBAC ads (longgarkan),
  - hardcode enum di UI,
  - health membocorkan env,
  lalu memastikan gate memerah dan baseline pulih hijau.
- B5. 1 putaran **E2E multi-peran** melalui `testing_agent_v3`:
  - dmlead (DM Supervisor), dm (DM Staff), marketing_admin, finance, owner.
  Fokus: create/edit campaign, manual spend, import wizard, performance warning honesty, attribution funnel, capi list/summary, RBAC 403.
- B6. Update `test_result.md` (set needs_retesting, catat hasil), update `plan.md` dan `docs/v2/30` bila ada kontrak yang berubah.

Output:
- `run_all_gates.sh` → **OVERALL PASS (27 gates)**.
- `python3 scripts/mutasi_43.py` → semua mutasi tertangkap, lalu pulih hijau.

---

### FASE C — Analitik & BI (fitur besar berikutnya)
**Tujuan:** membuka menu `/bi` menjadi dashboard nyata berprinsip **angka jujur + drill-down**.

#### Phase 1 — Core POC (isolasi, wajib)
User stories (POC metrik):
1. Sebagai owner, setiap KPI menampilkan `complete/missing` dan tidak memalsukan 0.
2. Sebagai analis, metrik CPL/CAC/ROAS tie-out dengan `/api/ads/performance`.
3. Sebagai finance, kas/AR tie-out dengan sumber GL/AR yang sudah ada.
4. Sebagai sales manager, funnel/aging tie-out dengan laporan umur tahap.
5. Sebagai maintainer, metrik bisa dihitung ulang (snapshot bukan sumber kebenaran).

Langkah:
- C1. Buat `poc/poc_44.py` untuk menguji kontrak `metrics` fungsi murni: `{value, breakdown, inputs, complete, missing}`.
- C2. Verifikasi tie-out minimal (1–2 metrik per persona) sebelum UI dibangun.

#### Phase 2 — V1 App Development
- Backend:
  - `backend/metrics/*.py` (fungsi murni per metrik),
  - `routers/analytics_router.py` (`/api/analytics/*`),
  - `metric_snapshots` + job harian (rebuildable).
- Frontend:
  - aktifkan menu `/bi` (hapus comingSoon),
  - hub bertab 5 persona (recharts + DataTable),
  - semua KPI punya drill ke route daftar terfilter.

#### Phase 3 — Gates + Mutasi + E2E
- C3. `scripts/verify_analytics.py` + registrasi gate.
- C4. `scripts/mutasi_44.py` (mutasi kejujuran angka, drill mismatch, tie-out palsu).
- C5. E2E multi-peran (owner, manager sales, dmlead, pm, finance).

---

### FASE D — Target Proyek & Budget/RAB (docs/v2/32)
User stories (MVP):
1. Sebagai owner, saya bisa membuat target unit+pendapatan per proyek dengan metode `linear_remaining`.
2. Sebagai owner, saya bisa memilih metode lain (`s_curve`, `manual`, `velocity_forecast`, `revenue_first`) dan melihat period plan berubah.
3. Sebagai PM/Finance, saya bisa melihat realisasi RAB dari PO/AP/SPK/material/jurnal dan status overbudget.
4. Sebagai user, saya bisa menambah master item budget tanpa ubah kode.
5. Sebagai owner, saya bisa drill overbudget dari general → detail.

Langkah:
- D1. POC perhitungan target & re-baseline bulanan.
- D2. Implement `project_targets` + wiring realisasi.
- D3. Gate `verify_budget_target.py` + mutasi + E2E.

---

### FASE E — Konsolidasi Proyek & Konstruksi (docs/v2/29)
User stories (MVP):
1. Sebagai PM, saya punya satu hub pembangunan yang konsisten untuk semua proyek.
2. Sebagai owner, saya bisa melihat progres portofolio + risiko keterlambatan.
3. Sebagai procurement/finance, biaya konstruksi tersaji konsisten dengan sumbernya.
4. Sebagai site engineer, saya melihat aksi hanya yang saya berhak (RBAC).
5. Sebagai maintainer, invariant build tidak rusak oleh refactor.

Langkah:
- E1. Gate `verify_build_hub.py` + mutasi + E2E.

---

## 3) Next Actions
1. FASE A: pecah `backend/models.py` (re-export) + perbaiki `verify_ia_v2.py` dengan ledger pintu resmi + ubah SpendEntryDialog `type=number`.
2. Jalankan `bash scripts/run_all_gates.sh` hingga **26 gate PASS**.
3. FASE B: implement `scripts/verify_ads.py` + registrasi gate ke-27.
4. FASE B: implement `scripts/mutasi_43.py` (8–10 mutasi) dan pastikan semua tertangkap.
5. Update `test_result.md` lalu delegasikan E2E Fase 43 ke `testing_agent_v3`.

---

## 4) Success Criteria
- Baseline:
  - `bash scripts/run_all_gates.sh` → **OVERALL PASS (26)**.
- Fase 43 tertutup:
  - `verify_ads.py` lulus dan “bergigi” (gagal saat idempotensi/kejujuran/RBAC/nav/SSOT/env-leak dirusak).
  - `python3 scripts/mutasi_43.py` → semua mutasi tertangkap, baseline pulih hijau.
  - E2E multi-peran tidak menemukan tombol mati/403 tak terduga, dan UI menampilkan “data belum lengkap” bila input tidak lengkap.
- Governance:
  - `plan.md` + `test_result.md` + docs/v2 terkait diperbarui di akhir tiap fase (tanpa angka karangan; mode simulasi tetap).