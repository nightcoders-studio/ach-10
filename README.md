# HanaPrice MVP — Panduan Lengkap

> Platform crowdsourced perbandingan harga pasar Banda Aceh + Chatbot AI

---

## Stack
- **Frontend**: Next.js 14 (App Router)
- **Database**: Supabase (PostgreSQL)
- **LLM**: Qwen3 1.7B via Ollama (lokal)
- **Deploy**: Vercel (frontend) + laptop lokal (LLM)

---

## Struktur Folder

```
hargaaceh/
├── app/
│   ├── page.tsx              # Dashboard utama
│   ├── submit/page.tsx       # Form input harga
│   ├── chat/page.tsx         # Halaman chatbot
│   ├── api/
│   │   ├── prices/route.ts   # GET semua harga, POST tambah harga
│   │   └── chat/route.ts     # POST chatbot endpoint
│   └── layout.tsx
├── components/
│   ├── PriceTable.tsx        # Tabel perbandingan harga
│   ├── PriceForm.tsx         # Form submit harga
│   ├── ChatWidget.tsx        # Komponen chatbot
│   └── Navbar.tsx
├── lib/
│   ├── supabase.ts           # Supabase client
│   └── ollama.ts             # Ollama/LLM helper
└── .env.local
```

---

## 1. Setup Supabase

### Jalankan SQL ini di Supabase SQL Editor:

```sql
-- Tabel lokasi toko/pasar
create table locations (
  id uuid default gen_random_uuid() primary key,
  name text not null,                    -- "Pasar Peunayong"
  area text not null,                    -- "Banda Aceh"
  gmaps_link text,                       -- link Google Maps
  created_at timestamptz default now()
);

-- Tabel produk
create table products (
  id uuid default gen_random_uuid() primary key,
  name text not null,                    -- "Cabai Merah"
  category text not null,               -- "Sayuran", "Daging", "Bumbu"
  unit text not null default 'kg',      -- "kg", "ikat", "butir"
  created_at timestamptz default now()
);

-- Tabel harga (inti aplikasi)
create table prices (
  id uuid default gen_random_uuid() primary key,
  product_id uuid references products(id) on delete cascade,
  location_id uuid references locations(id) on delete cascade,
  price integer not null,               -- harga dalam rupiah
  reported_by text default 'Anonim',   -- nama pelapor (opsional)
  note text,                            -- catatan tambahan
  reported_at timestamptz default now()
);

-- View untuk mempermudah query dashboard
create view price_summary as
select
  pr.name as product_name,
  pr.category,
  pr.unit,
  lo.name as location_name,
  lo.area,
  lo.gmaps_link,
  p.price,
  p.reported_by,
  p.note,
  p.reported_at
from prices p
join products pr on p.product_id = pr.id
join locations lo on p.location_id = lo.id
order by pr.name, p.price asc;

-- Seed data awal (pasar Banda Aceh)
insert into locations (name, area, gmaps_link) values
  ('Pasar Peunayong', 'Banda Aceh', 'https://maps.app.goo.gl/peunayong'),
  ('Pasar Seutui', 'Banda Aceh', 'https://maps.app.goo.gl/seutui'),
  ('Pasar Lambaro', 'Aceh Besar', 'https://maps.app.goo.gl/lambaro'),
  ('Pasar Ulee Kareng', 'Banda Aceh', 'https://maps.app.goo.gl/uleekareng'),
  ('Pasar Batoh', 'Banda Aceh', 'https://maps.app.goo.gl/batoh');

insert into products (name, category, unit) values
  ('Cabai Merah', 'Sayuran', 'kg'),
  ('Bawang Merah', 'Bumbu', 'kg'),
  ('Bawang Putih', 'Bumbu', 'kg'),
  ('Tomat', 'Sayuran', 'kg'),
  ('Beras Ramos', 'Pokok', 'kg'),
  ('Minyak Goreng', 'Pokok', 'liter'),
  ('Telur Ayam', 'Protein', 'butir'),
  ('Ayam Potong', 'Protein', 'kg'),
  ('Ikan Tongkol', 'Protein', 'kg'),
  ('Tempe', 'Protein', 'papan');

-- Enable Row Level Security (baca publik, tulis publik untuk MVP)
alter table prices enable row level security;
alter table products enable row level security;
alter table locations enable row level security;

create policy "Public read prices" on prices for select using (true);
create policy "Public insert prices" on prices for insert with check (true);
create policy "Public read products" on products for select using (true);
create policy "Public read locations" on locations for select using (true);
```

---

## 2. Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx...
OLLAMA_BASE_URL=http://localhost:11434
OLLAMA_MODEL=qwen3:1.7b
```

---

## 3. Supabase Client

```typescript
// lib/supabase.ts
import { createClient } from '@supabase/supabase-js'

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

---

## 4. API Routes

### GET/POST Harga

```typescript
// app/api/prices/route.ts
import { supabase } from '@/lib/supabase'
import { NextResponse } from 'next/server'

export async function GET(req: Request) {
  const { searchParams } = new URL(req.url)
  const product = searchParams.get('product')

  let query = supabase
    .from('price_summary')
    .select('*')
    .order('price', { ascending: true })

  if (product) {
    query = query.ilike('product_name', `%${product}%`)
  }

  const { data, error } = await query
  if (error) return NextResponse.json({ error }, { status: 500 })
  return NextResponse.json(data)
}

export async function POST(req: Request) {
  const body = await req.json()
  const { product_id, location_id, price, reported_by, note } = body

  if (!product_id || !location_id || !price) {
    return NextResponse.json({ error: 'Field wajib tidak lengkap' }, { status: 400 })
  }

  const { data, error } = await supabase
    .from('prices')
    .insert({ product_id, location_id, price: parseInt(price), reported_by, note })
    .select()
    .single()

  if (error) return NextResponse.json({ error }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}
```

### Chatbot API (streaming)

```typescript
// app/api/chat/route.ts
import { supabase } from '@/lib/supabase'

export async function POST(req: Request) {
  const { message } = await req.json()

  // Ambil data harga terbaru dari DB (50 data terakhir)
  const { data: prices } = await supabase
    .from('price_summary')
    .select('product_name, location_name, area, price, unit, gmaps_link, reported_at')
    .order('reported_at', { ascending: false })
    .limit(50)

  // Format data jadi teks ringkas untuk context LLM
  const priceContext = prices?.map(p =>
    `${p.product_name} (${p.unit}) di ${p.location_name}, ${p.area}: Rp${p.price.toLocaleString('id')} — ${p.gmaps_link ?? 'tidak ada link maps'}`
  ).join('\n') ?? 'Belum ada data harga.'

  const systemPrompt = `Kamu adalah asisten AI HargaAceh, membantu warga Banda Aceh mencari harga bahan pokok termurah.

DATA HARGA TERKINI:
${priceContext}

ATURAN:
- Jawab dalam bahasa Indonesia yang santai dan ramah
- Jika ditanya harga, cari dari data di atas dan sebutkan lokasi termurahnya
- Jika ada link Google Maps, sertakan di jawaban
- Jika data tidak ada, bilang jujur bahwa data belum tersedia dan minta warga untuk berkontribusi upload harga
- Jawaban singkat dan to the point, maksimal 3-4 kalimat
- Jangan mengarang harga yang tidak ada di data`

  // Stream dari Ollama
  const ollamaRes = await fetch(`${process.env.OLLAMA_BASE_URL}/api/chat`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: process.env.OLLAMA_MODEL,
      messages: [
        { role: 'system', content: systemPrompt },
        { role: 'user', content: message }
      ],
      stream: true
    })
  })

  // Forward stream ke client
  return new Response(ollamaRes.body, {
    headers: { 'Content-Type': 'text/plain; charset=utf-8' }
  })
}
```

---

## 5. Komponen Utama

### Form Submit Harga

```typescript
// components/PriceForm.tsx
'use client'
import { useEffect, useState } from 'react'
import { supabase } from '@/lib/supabase'

export default function PriceForm() {
  const [products, setProducts] = useState<any[]>([])
  const [locations, setLocations] = useState<any[]>([])
  const [form, setForm] = useState({
    product_id: '', location_id: '', price: '', reported_by: '', note: ''
  })
  const [status, setStatus] = useState<'idle'|'loading'|'success'|'error'>('idle')

  useEffect(() => {
    supabase.from('products').select('*').order('name').then(({ data }) => setProducts(data ?? []))
    supabase.from('locations').select('*').order('name').then(({ data }) => setLocations(data ?? []))
  }, [])

  const handleSubmit = async () => {
    setStatus('loading')
    const res = await fetch('/api/prices', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(form)
    })
    setStatus(res.ok ? 'success' : 'error')
    if (res.ok) setForm({ product_id: '', location_id: '', price: '', reported_by: '', note: '' })
  }

  return (
    <div className="max-w-lg mx-auto p-6 bg-white rounded-2xl border border-gray-100 shadow-sm">
      <h2 className="text-xl font-semibold mb-6">Laporkan Harga</h2>

      <div className="space-y-4">
        <div>
          <label className="block text-sm text-gray-500 mb-1">Produk *</label>
          <select
            value={form.product_id}
            onChange={e => setForm({...form, product_id: e.target.value})}
            className="w-full border border-gray-200 rounded-xl px-3 py-2.5 text-sm"
          >
            <option value="">Pilih produk...</option>
            {products.map(p => (
              <option key={p.id} value={p.id}>{p.name} (per {p.unit})</option>
            ))}
          </select>
        </div>

        <div>
          <label className="block text-sm text-gray-500 mb-1">Lokasi / Pasar *</label>
          <select
            value={form.location_id}
            onChange={e => setForm({...form, location_id: e.target.value})}
            className="w-full border border-gray-200 rounded-xl px-3 py-2.5 text-sm"
          >
            <option value="">Pilih lokasi...</option>
            {locations.map(l => (
              <option key={l.id} value={l.id}>{l.name} — {l.area}</option>
            ))}
          </select>
        </div>

        <div>
          <label className="block text-sm text-gray-500 mb-1">Harga (Rp) *</label>
          <input
            type="number"
            placeholder="contoh: 45000"
            value={form.price}
            onChange={e => setForm({...form, price: e.target.value})}
            className="w-full border border-gray-200 rounded-xl px-3 py-2.5 text-sm"
          />
        </div>

        <div>
          <label className="block text-sm text-gray-500 mb-1">Nama kamu (opsional)</label>
          <input
            type="text"
            placeholder="Anonim"
            value={form.reported_by}
            onChange={e => setForm({...form, reported_by: e.target.value})}
            className="w-full border border-gray-200 rounded-xl px-3 py-2.5 text-sm"
          />
        </div>

        <div>
          <label className="block text-sm text-gray-500 mb-1">Catatan (opsional)</label>
          <input
            type="text"
            placeholder="contoh: harga hari Senin pagi"
            value={form.note}
            onChange={e => setForm({...form, note: e.target.value})}
            className="w-full border border-gray-200 rounded-xl px-3 py-2.5 text-sm"
          />
        </div>

        <button
          onClick={handleSubmit}
          disabled={status === 'loading' || !form.product_id || !form.location_id || !form.price}
          className="w-full bg-emerald-600 text-white rounded-xl py-2.5 text-sm font-medium disabled:opacity-50"
        >
          {status === 'loading' ? 'Menyimpan...' : 'Kirim Laporan Harga'}
        </button>

        {status === 'success' && (
          <p className="text-emerald-600 text-sm text-center">Terima kasih! Harga berhasil dilaporkan.</p>
        )}
        {status === 'error' && (
          <p className="text-red-500 text-sm text-center">Gagal menyimpan. Coba lagi.</p>
        )}
      </div>
    </div>
  )
}
```

### ChatWidget

```typescript
// components/ChatWidget.tsx
'use client'
import { useState, useRef, useEffect } from 'react'

type Message = { role: 'user' | 'assistant'; content: string }

export default function ChatWidget() {
  const [messages, setMessages] = useState<Message[]>([
    { role: 'assistant', content: 'Halo! Tanya aku harga bahan pokok di Banda Aceh. Contoh: "cabai berapa sekarang?" atau "beras termurah di mana?"' }
  ])
  const [input, setInput] = useState('')
  const [loading, setLoading] = useState(false)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  const send = async () => {
    if (!input.trim() || loading) return
    const userMsg = input.trim()
    setInput('')
    setMessages(prev => [...prev, { role: 'user', content: userMsg }])
    setLoading(true)

    // Tambah placeholder pesan AI
    setMessages(prev => [...prev, { role: 'assistant', content: '' }])

    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message: userMsg })
      })

      const reader = res.body!.getReader()
      const decoder = new TextDecoder()
      let full = ''

      while (true) {
        const { done, value } = await reader.read()
        if (done) break
        const chunk = decoder.decode(value)
        // Parse Ollama streaming JSON lines
        for (const line of chunk.split('\n').filter(Boolean)) {
          try {
            const json = JSON.parse(line)
            if (json.message?.content) {
              full += json.message.content
              setMessages(prev => {
                const updated = [...prev]
                updated[updated.length - 1] = { role: 'assistant', content: full }
                return updated
              })
            }
          } catch {}
        }
      }
    } catch {
      setMessages(prev => {
        const updated = [...prev]
        updated[updated.length - 1] = { role: 'assistant', content: 'Maaf, terjadi error. Coba lagi.' }
        return updated
      })
    }
    setLoading(false)
  }

  return (
    <div className="flex flex-col h-[500px] bg-white rounded-2xl border border-gray-100 shadow-sm overflow-hidden">
      <div className="px-4 py-3 border-b border-gray-100">
        <p className="font-semibold text-sm">Tanya HargaAceh AI</p>
        <p className="text-xs text-gray-400">Data dari komunitas · Powered by Qwen3</p>
      </div>

      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((m, i) => (
          <div key={i} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-[80%] px-3.5 py-2.5 rounded-2xl text-sm leading-relaxed whitespace-pre-wrap
              ${m.role === 'user'
                ? 'bg-emerald-600 text-white rounded-br-sm'
                : 'bg-gray-100 text-gray-800 rounded-bl-sm'
              }`}>
              {m.content || <span className="opacity-50">...</span>}
            </div>
          </div>
        ))}
        <div ref={bottomRef} />
      </div>

      <div className="p-3 border-t border-gray-100 flex gap-2">
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          onKeyDown={e => e.key === 'Enter' && send()}
          placeholder="Tanya harga..."
          className="flex-1 border border-gray-200 rounded-xl px-3 py-2 text-sm"
        />
        <button
          onClick={send}
          disabled={loading || !input.trim()}
          className="bg-emerald-600 text-white px-4 rounded-xl text-sm disabled:opacity-50"
        >
          Kirim
        </button>
      </div>
    </div>
  )
}
```

---

## 6. Ollama Setup (laptop kawanmu)

```bash
# Install Ollama
curl -fsSL https://ollama.ai/install.sh | sh

# Download & jalankan Qwen3
ollama pull qwen3:1.7b
ollama serve

# Test via curl
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3:1.7b",
  "messages": [{"role":"user","content":"halo"}],
  "stream": false
}'
```

---

## 7. Install & Jalankan Project

```bash
# Buat project Next.js
npx create-next-app@latest hargaaceh --typescript --tailwind --app
cd hargaaceh

# Install dependencies
npm install @supabase/supabase-js

# Copy .env.local dan isi dengan kredensial Supabase
# Jalankan
npm run dev
```

---

## 8. Skenario Demo (untuk presentasi juri)

1. Buka dashboard → tampilkan tabel harga cabai merah dari 3 pasar berbeda
2. Klik link Google Maps salah satu lokasi
3. Buka form → input harga baru secara live
4. Refresh dashboard → data langsung muncul (realtime)
5. Buka chatbot → tanya *"cabai paling murah di mana sekarang?"* → AI jawab dengan lokasi + link maps
6. Tanya *"beras ramos di Pasar Peunayong berapa?"* → AI jawab spesifik

---

## Checklist MVP 6 Jam

- [ ] Supabase project dibuat + SQL dijalankan
- [ ] `.env.local` diisi
- [ ] `npm run dev` jalan tanpa error
- [ ] Form submit harga berfungsi
- [ ] Dashboard tabel tampil data
- [ ] Chatbot bisa jawab pertanyaan harga
- [ ] Seed 20+ data harga nyata diinput
- [ ] Demo flow sudah dicoba 2x
