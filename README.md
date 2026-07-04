# WEB LCC — Live Score Lomba Cerdas Cermat

developer: el, very

Web live scoring lomba cerdas cermat (dan lomba lain), dengan dua babak: **Quarterfinal** (live score gaya Excel, multi-device) dan **Final** (animasi balap lari + podium juara).

---

## Request definedd

Poin-poin permintaan client (sumber kebenaran fitur):

- [ ] Web **live score** untuk lomba cerdas cermat (dan beberapa lomba lain menyusul).
- [ ] Ada **2 babak**: **Quarterfinal** dan **Final**.
- [ ] **Babak Quarterfinal** dibuat sederhana seperti live score di Excel.
- [ ] Bisa diakses **banyak laptop sekaligus**, dan score **langsung ter-update realtime** di semua perangkat.
- [ ] Ada **20 tim** → butuh **20 bagan live score**.
- [ ] Di akhir Quarterfinal ada tampilan **urutan rank**.
- [ ] **Babak Final** dibuat dengan **animasi lomba lari**: kalau score ditambah, pelari **melangkah maju**; yang **paling depan = menang**.
- [ ] Final ada **5 tim** → hanya **5 pelari**.
- [ ] Ada tampilan **podium juara 1, 2, 3**.
- [ ] Pendekatan **best approach & simpel**.
- [ ] Animasi **bagus tapi sederhana** (ringan, tidak berat).

---

## Ringkasan Fitur

| Babak | Tampilan Utama | Kontrol Juri | Output Akhir |
|-------|----------------|--------------|--------------|
| Quarterfinal | 20 kartu live score (grid) | Tambah/kurang poin per tim | Halaman **Rank / Leaderboard** |
| Final | 5 pelari di lintasan (animasi lari) | Tambah/kurang poin per tim | Halaman **Podium** juara 1–3 |

Semua perubahan score **broadcast realtime** ke semua device via Supabase Realtime — cukup satu juri input, semua penonton & laptop lain langsung ikut ter-update tanpa refresh.

---

## Tech Stack

| Layer | Teknologi | Alasan |
|-------|-----------|--------|
| Frontend | **React + Vite** | Cepat, ringan, HMR instan |
| Styling | **TailwindCSS** | Cepat bikin UI rapi & responsif |
| Animasi | **Framer Motion** | Animasi langkah pelari, count-up, reorder rank, reveal podium — deklaratif & smooth |
| Routing | **React Router** | Multi halaman (viewer, referee, rank, podium) |
| Database + Realtime | **Supabase** (Postgres + Realtime) | Live sync antar-device tanpa server sendiri |
| Auth juri | **Supabase Auth** (atau passcode sederhana) | Batasi input score hanya untuk juri |
| Deploy | **Vercel / Netlify** | Static hosting + env vars mudah |

---

## Arsitektur Singkat

```
                       ┌─────────────────────────┐
                       │   Supabase (Postgres)   │
                       │  matches / teams /      │
                       │  match_scores           │
                       └───────────┬─────────────┘
                                   │ Realtime (postgres_changes)
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
      ┌──────────────┐    ┌──────────────┐     ┌──────────────┐
      │ Referee/Juri │    │ Viewer Score │     │  Big Screen  │
      │ (input poin) │    │ (laptop lain)│     │ (proyektor)  │
      └──────────────┘    └──────────────┘     └──────────────┘
         write score          read-only            read-only
```

- **Satu sumber data** (`match_scores`) → semua tampilan subscribe ke tabel yang sama.
- Juri menekan tombol +/- → update ke Supabase → Realtime push ke semua client → UI & animasi jalan otomatis.

---

## Struktur Database (Supabase)

### Tabel `matches`
Mewakili satu sesi/babak lomba.

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `id` | uuid (PK) | id match |
| `title` | text | mis. "Cerdas Cermat — Quarterfinal Grup A" |
| `round_type` | text | `quarterfinal` \| `final` |
| `status` | text | `pending` \| `live` \| `finished` |
| `created_at` | timestamptz | default now() |

### Tabel `teams`
Daftar tim peserta.

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `id` | uuid (PK) | id tim |
| `name` | text | nama tim |
| `short_name` | text | singkatan (untuk kartu kecil) |
| `color` | text | warna kaos/pelari (hex) |
| `created_at` | timestamptz | default now() |

### Tabel `match_scores`
Score satu tim pada satu match (baris inilah yang disubscribe realtime).

| Kolom | Tipe | Keterangan |
|-------|------|-----------|
| `id` | uuid (PK) | |
| `match_id` | uuid (FK → matches) | |
| `team_id` | uuid (FK → teams) | |
| `score` | int (default 0) | poin tim |
| `lane` | int | nomor lintasan (khusus final, 1–5) |
| `updated_at` | timestamptz | default now() |

> Rank/urutan **tidak disimpan** — dihitung on-the-fly dari `score` (ORDER BY score DESC). Posisi pelari di Final = fungsi dari `score`, jadi cukup simpan `score` saja.

### SQL awal (bootstrap)

```sql
create table matches (
  id uuid primary key default gen_random_uuid(),
  title text not null,
  round_type text not null check (round_type in ('quarterfinal','final')),
  status text not null default 'pending' check (status in ('pending','live','finished')),
  created_at timestamptz default now()
);

create table teams (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  short_name text,
  color text default '#3b82f6',
  created_at timestamptz default now()
);

create table match_scores (
  id uuid primary key default gen_random_uuid(),
  match_id uuid references matches(id) on delete cascade,
  team_id uuid references teams(id) on delete cascade,
  score int not null default 0,
  lane int,
  updated_at timestamptz default now(),
  unique (match_id, team_id)
);

-- aktifkan realtime
alter publication supabase_realtime add table match_scores;
alter publication supabase_realtime add table matches;
```

### Keamanan (RLS)
- **SELECT**: publik (semua boleh lihat live score).
- **UPDATE/INSERT** ke `match_scores` & `matches`: hanya user login (juri) via Supabase Auth.
- Alternatif simpel tanpa Auth: halaman referee dilindungi **passcode** + policy `service_role` dari sisi tepercaya. (Rekomendasi tetap pakai Supabase Auth agar aman.)

---

## Daftar Halaman (Routes)

| Route | Nama Halaman | Akses | Fungsi |
|-------|--------------|-------|--------|
| `/` | **Home / Landing** | Publik | Pilih lomba & babak; link ke viewer/referee |
| `/referee` | **Referee — Match Control** | Juri | Pilih match, set `match type` & `match status`, buka panel input |
| `/referee/quarterfinal/:matchId` | **Referee Quarterfinal** | Juri | Grid 20 tim, tombol +/− poin per tim |
| `/referee/final/:matchId` | **Referee Final** | Juri | 5 tim, tombol +/− poin, tombol "Selesai → Podium" |
| `/score/:matchId` | **Viewer — Live Score (Quarterfinal)** | Publik | 20 bagan live score realtime, read-only |
| `/rank/:matchId` | **Rank / Leaderboard** | Publik | Urutan tim dari poin tertinggi (animasi reorder) |
| `/race/:matchId` | **Viewer — Final Race** | Publik | Animasi 5 pelari di lintasan, realtime |
| `/podium/:matchId` | **Podium** | Publik | Reveal juara 1, 2, 3 |

### Detail tiap halaman

**1. Home / Landing (`/`)**
- Daftar match aktif + badge status (`pending`/`live`/`finished`).
- Tombol besar: "Lihat Live Score", "Buka Panel Juri".
- Simpel, sebagai pintu masuk.

**2. Referee — Match Control (`/referee`)**
- **Match Type**: pilih `quarterfinal` atau `final`.
- **Match Status**: ubah `pending → live → finished`.
- Buat match baru, assign tim ke match, atur lane (final).
- Dari sini masuk ke panel input sesuai tipe.

**3. Referee Quarterfinal (`/referee/quarterfinal/:matchId`)**
- Grid 20 kartu tim. Tiap kartu: nama tim, score besar, tombol **+1 / +5 / −1** (nilai bisa disesuaikan aturan lomba).
- Optimistic update + tulis ke Supabase.
- Tombol "Tampilkan Rank" → buka `/rank/:matchId`.

**4. Referee Final (`/referee/final/:matchId`)**
- 5 kartu tim (5 pelari). Tombol +/− poin.
- Setiap penambahan poin → pelari di `/race` melangkah maju.
- Tombol "Akhiri & Tampilkan Podium" → set status `finished` → `/podium/:matchId`.

**5. Viewer Live Score Quarterfinal (`/score/:matchId`)**
- **20 bagan live score** dalam grid responsif (mis. 4×5 atau 5×4).
- Tiap bagan: nama tim + angka score. Angka **count-up** saat berubah, kartu **highlight** sebentar.
- 100% read-only, aman dibuka di banyak laptop.

**6. Rank / Leaderboard (`/rank/:matchId`)**
- Daftar tim urut poin tertinggi → terendah.
- **Animasi reorder** (Framer Motion `layout`) saat peringkat berubah.
- Highlight top 3.

**7. Viewer Final Race (`/race/:matchId`)**
- Lintasan lari dengan **5 pelari** (5 lane).
- Posisi horizontal pelari = fungsi dari `score` (mis. `left = score / maxScore * trackWidth`).
- Saat poin bertambah → pelari **melangkah maju** dengan transisi smooth.
- Garis finish + indikator siapa terdepan.

**8. Podium (`/podium/:matchId`)**
- Tiga tiang podium: **1**, **2**, **3**.
- **Reveal beranimasi** (naik dari bawah, juara 1 terakhir + efek konfeti sederhana).

---

## Animasi 

| Animasi | Di mana | Teknik |
|---------|---------|--------|
| **Count-up angka** | Live score & race | Framer Motion `animate` / interpolasi angka |
| **Highlight kartu** | Live score | Perubahan warna singkat saat `score` update |
| **Reorder rank** | Leaderboard | Framer Motion `layout` + `AnimatePresence` |
| **Langkah pelari** | Final Race | `transform: translateX()` + `transition` (CSS/Framer) |
| **Reveal podium** | Podium | `spring` masuk dari bawah, delay bertahap |

Prinsip: animasi digerakkan **data (score)**, bukan trigger manual → otomatis konsisten di semua device. Ringan, tanpa library berat, aman untuk proyektor.

---

## Struktur Folder planning by rev

```
WEB_LCC/
├── src/
│   ├── lib/
│   │   └── supabase.js          # inisialisasi client
│   ├── hooks/
│   │   ├── useMatchScores.js    # subscribe realtime match_scores
│   │   └── useMatch.js          # data & status match
│   ├── components/
│   │   ├── ScoreCard.jsx        # kartu live score (quarterfinal)
│   │   ├── ScoreControls.jsx    # tombol +/- juri
│   │   ├── Runner.jsx           # pelari (final)
│   │   ├── RaceTrack.jsx        # lintasan 5 lane
│   │   ├── RankRow.jsx          # baris leaderboard
│   │   └── Podium.jsx           # podium juara
│   ├── pages/
│   │   ├── Home.jsx
│   │   ├── RefereeControl.jsx
│   │   ├── RefereeQuarterfinal.jsx
│   │   ├── RefereeFinal.jsx
│   │   ├── ViewerScore.jsx
│   │   ├── Rank.jsx
│   │   ├── Race.jsx
│   │   └── PodiumPage.jsx
│   ├── App.jsx                  # routing
│   └── main.jsx
├── .env.local                   # VITE_SUPABASE_URL / VITE_SUPABASE_ANON_KEY
├── tailwind.config.js
├── vite.config.js
└── README.md
```

---

## Alur Realtime req realtime

1. Semua tampilan viewer subscribe: `supabase.channel('scores').on('postgres_changes', { table: 'match_scores', filter: 'match_id=eq.<id>' }, ...)`.
2. Juri klik **+poin** → `UPDATE match_scores SET score = score + n`.
3. Supabase broadcast perubahan → semua client menerima payload baru.
4. State React ter-update → komponen re-render → animasi jalan (count-up / langkah pelari / reorder).

> Karena semua device baca dari tabel yang sama, tidak ada "score beda-beda" antar laptop.

---

## Roadmap / Milestone

- [ ] **M1 — Setup**: Vite + React + Tailwind + Framer Motion + Supabase client, buat tabel & seed 20+5 tim.
- [ ] **M2 — Realtime core**: hook `useMatchScores`, uji sync 2 browser.
- [ ] **M3 — Quarterfinal**: viewer 20 bagan + referee panel + halaman Rank.
- [ ] **M4 — Final**: race track 5 pelari + referee panel + podium.
- [ ] **M5 — Auth & RLS**: login juri, kunci input score.
- [ ] **M6 — Polish**: animasi, responsif proyektor, mode fullscreen, deploy.

---

## Setup (abis scafhold)

```bash
npm install
cp .env.example .env.local   # isi VITE_SUPABASE_URL & VITE_SUPABASE_ANON_KEY
npm run dev
```

Env yang dibutuhkan:

```
VITE_SUPABASE_URL=...
VITE_SUPABASE_ANON_KEY=...
```

---

## Advance deleopment (opsional)

- Dukungan **multi-lomba** (bukan cuma cerdas cermat) lewat kolom `competition` di `matches`.
- **Sound effect** saat poin bertambah / juara diumumkan.
- **Buzzer** online untuk rebutan jawab.
- **Export hasil** ke PDF/Excel.
- Mode **fullscreen kiosk** untuk layar besar.
