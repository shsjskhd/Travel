# Plan — CMS Continuation (RahazaTrans Travel ERP)

## 1) Objectives
- Selesaikan **SEMUA sisa** dari `CMS_REVIEW_AND_ENHANCEMENT_PLAN.md`: **CMS-05..CMS-09** + perbaiki **3 defect baru (A1–A3)**.
- Pastikan integrasi CMS benar-benar “menghasilkan bisnis”: konten bisa dipreview, dipublish terjadwal, multi-bahasa, promo bisa dikelola end-to-end, ulasan masuk jadi testimoni, dan ada analitik konten→lead→booking.
- Tetap **non-regression**: `bash scripts/gate.sh` wajib HIJAU; RBAC+Audit+whitelist tetap konsisten; UI mengikuti `design_guidelines.md`.

## 2) Implementation Steps

### Phase 1 — Core POC (isolation, wajib)
Fokus pada mekanik paling riskan: token preview, auto-publish scheduler, review-token funnel, AI translate helper, rich-text sanitization.

**POC-1: Scheduled Publish + Preview Token (CMS-05)**
- BE: tambahkan model status `draft|scheduled|published`, `publish_at`, compat dengan `published` lama.
- BE: endpoint **preview-token** (signed+expiring) untuk akses konten belum published.
- Scheduler: job publish yang mempromosikan scheduled→published saat `publish_at <= now`.
- Script POC: `scripts/test_cms_publish_workflow.py` (create draft → mint token → public fetch works w/ token → schedule publish → assert visible w/o token).

**POC-2: Review Funnel Token (CMS-07)**
- BE: token review per trip selesai; endpoint submit ulasan (rating+quote+consent), masuk `testimonials approved=false`.
- Mock WA: kirim “link ulasan” via inbox log.
- Script POC: `scripts/test_review_funnel.py` (create trip completed → generate token → submit → appears in moderation list).

**POC-3: AI Translation helper (CMS-06)**
- BE: endpoint translate (ID→EN) memakai Emergent LLM key; output hanya “suggestion”, tidak auto overwrite.
- Script POC: `scripts/test_translate_ai.py` (call once, validate JSON response shape + fallback when key missing).
- Websearch singkat: best practice prompt + safe output schema (JSON) + rate-limit handling.

**POC-4: Rich text round-trip (CMS-09)**
- FE: block editor minimal (heading/paragraph/list/quote/image/divider).
- BE: simpan sebagai HTML tersanitasi + simpan sumber blocks (opsional) untuk edit.
- Script POC: `scripts/test_richtext_sanitize.py` (inject XSS payload → ensure sanitized).

> Stop rule: tidak lanjut Phase 2 sebelum semua POC PASS.

### Phase 2 — V1 App Development (implement menyeluruh)

**A) Fix 3 Defects (A1–A3)**
1. **A1 Preview link + Package public pages**
   - Perbaiki link preview destinations → `/destinations/:slug`.
   - Tambah public **Packages**: `/packages` (list) + `/packages/:slug` (detail) + CTA ke booking/quotation.
   - Update nav map + sitemap include packages pages.
2. **A2 Promo rules UI lengkap**
   - Lengkapi form promo di CMS: `valid_from`, `min_days`, `min_amount`, `vehicle_types[]`, `services[]`, `weekend_only`, `max_uses`, tampilkan `used_count` read-only.
   - Tampilkan badge “Eligible / Tidak eligible” untuk konteks contoh (opsional: mini evaluator).
3. **A3 Public `/promo` page**
   - Halaman `/promo`: daftar promo aktif, render syarat dari data (bukan teks), tombol copy code, CTA “Pakai di Pesan Online” (carry code ke BookingWizard).

**B) CMS-05 Draft/Preview/Scheduled Publish**
- Data: tambah field status, publish_at; migrasi kompatibel.
- Public endpoints: hanya published (kecuali ada preview token valid).
- Admin UI: filter status, badge status, set publish schedule, preview token button.

**C) CMS-06 Multi-language (ID/EN)**
- Data: `translations.en` per resource (destinations/packages/articles/promos) + fallback.
- Public: `?lang=en` + language selector di PublicLayout; `hreflang` + `og:locale`.
- Sitemap: include alternates EN.
- Admin UI: tab bahasa (ID/EN) + tombol “Terjemahkan dengan AI” per-field (opsional).

**D) CMS-07 Review → Testimonials**
- Trigger: saat trip completed → buat review request token + log WA mock ke Inbox.
- Public: halaman review `/review/:token`.
- CMS: moderasi testimoni (approve/reject), list pending.
- Public: tampilkan rating agregat + JSON-LD AggregateRating.

**E) CMS-08 Content analytics + attribution**
- Views: endpoint increment view untuk destination/article/package (bot-safe, rate-limit, no PII).
- Attribution: simpan last-content/utm ke lead/booking.
- CMS dashboard: “Top konten” (views, leads, bookings, revenue attributed).

**F) CMS-09 Rich text**
- FE editor di ContentFormDialog utk `articles.body` (block editor).
- Rendering aman di BlogDetail + fallback untuk body lama.

**V1 User Stories (min 5)**
1. Sebagai ops/owner, saya bisa preview konten **draft/scheduled** lewat link token tanpa mempublish ke publik.
2. Sebagai editor, saya bisa menjadwalkan artikel terbit otomatis pada waktu tertentu.
3. Sebagai owner, saya bisa membuat promo dengan syarat nyata (min hari, kuota, tipe armada) dan pelanggan dapat menggunakannya di booking.
4. Sebagai calon pelanggan, saya bisa membuka halaman `/promo`, copy kode promo, lalu masuk ke Pesan Online dengan kode terisi.
5. Sebagai calon turis asing, saya bisa membaca halaman destinasi/artikel dalam English dengan fallback ke Indonesia.
6. Sebagai pelanggan selesai trip, saya menerima (mock) link ulasan, mengisi rating+testimoni, dan itu masuk antrian moderasi.
7. Sebagai owner, saya melihat dashboard konten mana yang menghasilkan lead/booking terbanyak.

**Close Phase 2:** jalankan `gate.sh` + minta `testing_agent` uji E2E CMS+public.

### Phase 3 — Hardening + UX polish (production-friendly)
- Rate-limit untuk view counter + translate endpoint.
- Better empty/error states (public+CMS), tambah data-testid penting.
- Tambah export ringan untuk analytics (CSV) bila cepat.

**Phase 3 User Stories (min 5)**
1. Sebagai owner, saya bisa memfilter konten berdasarkan status/locale untuk kerja cepat.
2. Sebagai ops, saya bisa melihat quota promo terpakai vs tersisa.
3. Sebagai editor, saya bisa mengembalikan versi body lama ke format baru tanpa kehilangan konten.
4. Sebagai owner, saya bisa melihat sumber lead (UTM/halaman terakhir) dari konten.
5. Sebagai pengguna publik, halaman tetap cepat (no heavy JS) dan SEO tags konsisten.

**Close Phase 3:** `gate.sh` HIJAU + `testing_agent` round 2.

## 3) Next Actions (immediate)
1. Tambah **plan baru** ini ke `/app/plan.md` (archive plan booking lama ke `docs/PLAN_ARCHIVE_*`).
2. Implement **POC-1..4 scripts** + endpoint minimal yang dibutuhkan.
3. Setelah POC PASS: mulai Phase 2 dari A1 (routes packages + preview link) lalu A2/A3.
4. Delegasikan verifikasi ke **testing_agent** setiap selesai wave besar (POC done, Phase 2 done, Phase 3 done).

## 4) Success Criteria
- Fitur A1–A3 + CMS-05..CMS-09 **berfungsi end-to-end** pada UI publik dan admin.
- `bash scripts/gate.sh` **HIJAU (0 FAIL, 0 SKIP)**.
- Dokumen SSOT update: `docs/03_DATA_MODEL.md`, `docs/04_API_CONTRACT.md`, `docs/05_NAVIGATION_MAP.md`, `docs/10_CODEBASE_MAP.md`, `memory/*` relevan, `CMS_REVIEW_AND_ENHANCEMENT_PLAN.md` status.
- testing_agent memverifikasi user stories utama (POC, publish workflow, promo, i18n, review funnel, rich text, analytics) dan melaporkan hasilnya di `test_result.md`.
- WA/Ads/GA4 tetap **MOCKED** namun jelas dilabeli di UI/dokumen.