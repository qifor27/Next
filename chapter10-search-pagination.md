# Chapter 10: Adding Search and Pagination

## 📚 Pengantar

Chapter ini membahas cara mengimplementasikan fitur **Search (Pencarian)** dan **Pagination (Paginasi)** pada aplikasi Next.js menggunakan **URL Search Params**. Pendekatan ini memiliki beberapa keuntungan:

### Mengapa Menggunakan URL Search Params?

| Keuntungan | Penjelasan |
|------------|------------|
| **Bookmarkable & Shareable** | URL bisa di-bookmark atau dibagikan, termasuk state pencarian dan halaman |
| **Server-Side Rendering** | URL params bisa langsung dibaca di server untuk render awal |
| **Analytics & Tracking** | Mudah melacak perilaku pengguna tanpa JavaScript tambahan |
| **No Client State** | Tidak perlu mengelola state di client |

---

## 🔍 Bagian 1: Search (Pencarian)

### 1.1 Hooks yang Digunakan

```typescript
import { useSearchParams, usePathname, useRouter } from 'next/navigation';
import { useDebouncedCallback } from 'use-debounce';
```

| Hook | Fungsi |
|------|--------|
| `useSearchParams` | Membaca parameter dari URL saat ini (contoh: `?query=michael`) |
| `usePathname` | Membaca path URL saat ini (contoh: `/dashboard/invoices`) |
| `useRouter` | Menyediakan metode navigasi seperti `replace()` untuk mengubah URL |
| `useDebouncedCallback` | Menunda eksekusi fungsi untuk mengurangi request berlebihan |

### 1.2 Komponen Search (`app/ui/search.tsx`)

```typescript
'use client';

import { MagnifyingGlassIcon } from '@heroicons/react/24/outline';
import { useSearchParams, usePathname, useRouter } from 'next/navigation';
import { useDebouncedCallback } from 'use-debounce';

export default function Search({ placeholder }: { placeholder: string }) {
  const searchParams = useSearchParams();
  const pathname = usePathname();
  const { replace } = useRouter();

  // Debounce: tunggu 300ms setelah user berhenti mengetik
  const handleSearch = useDebouncedCallback((term: string) => {
    console.log(`Searching... ${term}`);
    
    // Buat URLSearchParams baru dari params yang ada
    const params = new URLSearchParams(searchParams);
    
    // Reset ke halaman 1 saat search berubah
    params.set('page', '1');
    
    if (term) {
      params.set('query', term);  // Set query jika ada
    } else {
      params.delete('query');     // Hapus query jika kosong
    }
    
    // Update URL tanpa refresh halaman
    replace(`${pathname}?${params.toString()}`);
  }, 300);

  return (
    <div className="relative flex flex-1 flex-shrink-0">
      <label htmlFor="search" className="sr-only">
        Search
      </label>
      <input
        className="peer block w-full rounded-md border border-gray-200 py-[9px] pl-10 text-sm outline-2 placeholder:text-gray-500"
        placeholder={placeholder}
        onChange={(e) => {
          handleSearch(e.target.value);
        }}
        // Sinkronkan input dengan URL params
        defaultValue={searchParams.get('query')?.toString()}
      />
      <MagnifyingGlassIcon className="absolute left-3 top-1/2 h-[18px] w-[18px] -translate-y-1/2 text-gray-500 peer-focus:text-gray-900" />
    </div>
  );
}
```

### 1.3 Alur Kerja Search

```
┌─────────────────────────────────────────────────────────────────┐
│                        ALUR SEARCH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. User mengetik "michael" di input                            │
│           │                                                     │
│           ▼                                                     │
│  2. Debounce menunggu 300ms (mencegah request berlebihan)       │
│           │                                                     │
│           ▼                                                     │
│  3. handleSearch() dipanggil                                    │
│           │                                                     │
│           ▼                                                     │
│  4. URL diupdate: /dashboard/invoices?query=michael&page=1      │
│           │                                                     │
│           ▼                                                     │
│  5. Server Component membaca searchParams                       │
│           │                                                     │
│           ▼                                                     │
│  6. Database query dengan filter "michael"                      │
│           │                                                     │
│           ▼                                                     │
│  7. Hasil ditampilkan di tabel                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 Debouncing

**Debouncing** adalah teknik untuk menunda eksekusi fungsi sampai user berhenti melakukan aksi selama waktu tertentu.

**Tanpa Debouncing:**
```
Ketik "m"     → Request ke database
Ketik "mi"    → Request ke database
Ketik "mic"   → Request ke database
Ketik "mich"  → Request ke database
... → 7 request total!
```

**Dengan Debouncing (300ms):**
```
Ketik "michael" → Tunggu 300ms → 1 Request ke database
```

---

## 📖 Bagian 2: Pagination (Paginasi)

### 2.1 Komponen Pagination (`app/ui/invoices/pagination.tsx`)

```typescript
'use client';

import { ArrowLeftIcon, ArrowRightIcon } from '@heroicons/react/24/outline';
import clsx from 'clsx';
import Link from 'next/link';
import { generatePagination } from '@/app/lib/utils';
import { usePathname, useSearchParams } from 'next/navigation';

export default function Pagination({ totalPages }: { totalPages: number }) {
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const currentPage = Number(searchParams.get('page')) || 1;

  // Fungsi untuk membuat URL dengan page number baru
  const createPageURL = (pageNumber: number | string) => {
    const params = new URLSearchParams(searchParams);
    params.set('page', pageNumber.toString());  // ⬅️ PENTING: Set page number
    return `${pathname}?${params.toString()}`;
  };

  // Generate array halaman untuk ditampilkan
  const allPages = generatePagination(currentPage, totalPages);

  return (
    <div className="inline-flex">
      {/* Tombol Previous */}
      <PaginationArrow
        direction="left"
        href={createPageURL(currentPage - 1)}
        isDisabled={currentPage <= 1}
      />

      {/* Nomor Halaman */}
      <div className="flex -space-x-px">
        {allPages.map((page, index) => {
          let position: 'first' | 'last' | 'single' | 'middle' | undefined;

          if (index === 0) position = 'first';
          if (index === allPages.length - 1) position = 'last';
          if (allPages.length === 1) position = 'single';
          if (page === '...') position = 'middle';

          return (
            <PaginationNumber
              key={`${page}-${index}`}
              href={createPageURL(page)}
              page={page}
              position={position}
              isActive={currentPage === page}
            />
          );
        })}
      </div>

      {/* Tombol Next */}
      <PaginationArrow
        direction="right"
        href={createPageURL(currentPage + 1)}
        isDisabled={currentPage >= totalPages}
      />
    </div>
  );
}
```

### 2.2 Fungsi generatePagination (`app/lib/utils.ts`)

Fungsi ini menghasilkan array nomor halaman yang akan ditampilkan:

```typescript
export const generatePagination = (currentPage: number, totalPages: number) => {
  // Jika total halaman <= 7, tampilkan semua
  if (totalPages <= 7) {
    return Array.from({ length: totalPages }, (_, i) => i + 1);
    // Contoh: [1, 2, 3, 4, 5, 6, 7]
  }

  // Jika di 3 halaman pertama
  if (currentPage <= 3) {
    return [1, 2, 3, '...', totalPages - 1, totalPages];
    // Contoh: [1, 2, 3, '...', 9, 10]
  }

  // Jika di 3 halaman terakhir
  if (currentPage >= totalPages - 2) {
    return [1, 2, '...', totalPages - 2, totalPages - 1, totalPages];
    // Contoh: [1, 2, '...', 8, 9, 10]
  }

  // Jika di tengah-tengah
  return [
    1,
    '...',
    currentPage - 1,
    currentPage,
    currentPage + 1,
    '...',
    totalPages,
  ];
  // Contoh (page 5): [1, '...', 4, 5, 6, '...', 10]
};
```

### 2.3 Visualisasi Pagination

```
Total 10 halaman, currentPage = 5:

┌───┐ ┌─────┐ ┌───┐ ┌───┐ ┌───┐ ┌─────┐ ┌────┐
│ ← │ │  1  │ │...│ │ 4 │ │ 5 │ │  6  │ │... │ │ 10 │ │ → │
└───┘ └─────┘ └───┘ └───┘ └───┘ └─────┘ └────┘
                          ▲
                          │
                    (Halaman aktif)
```

---

## 🔗 Bagian 3: Integrasi di Server Component

### 3.1 Halaman Invoices (`app/dashboard/invoices/page.tsx`)

```typescript
import Pagination from "@/app/ui/invoices/pagination";
import Search from "@/app/ui/search";
import Table from "@/app/ui/invoices/table";
import { CreateInvoice } from "@/app/ui/invoices/buttons";
import { lusitana } from "@/app/ui/fonts";
import { InvoicesTableSkeleton } from "@/app/ui/skeletons";
import { Suspense } from "react";
import { fetchInvoicesPages } from "@/app/lib/data";

export default async function Page(props: {
  searchParams?: Promise<{
    query?: string;
    page?: string;
  }>;
}) {
  // Membaca searchParams dari URL
  const searchParams = await props.searchParams;
  const query = searchParams?.query || '';
  const currentPage = Number(searchParams?.page) || 1;
  
  // Fetch total halaman berdasarkan query
  const totalPages = await fetchInvoicesPages(query);

  return (
    <div className="w-full">
      <div className="flex w-full items-center justify-between">
        <h1 className={`${lusitana.className} text-2xl`}>Invoices</h1>
      </div>
      
      <div className="mt-4 flex items-center justify-between gap-2 md:mt-8">
        <Search placeholder="Search invoices..." />
        <CreateInvoice />
      </div>
      
      {/* Suspense untuk loading state */}
      <Suspense key={query + currentPage} fallback={<InvoicesTableSkeleton />}>
        <Table query={query} currentPage={currentPage} />
      </Suspense>
      
      <div className="mt-5 flex w-full justify-center">
        <Pagination totalPages={totalPages} />
      </div>
    </div>
  );
}
```

### 3.2 Data Fetching Functions (`app/lib/data.ts`)

```typescript
const ITEMS_PER_PAGE = 6;

// Fetch invoices dengan filter dan pagination
export async function fetchFilteredInvoices(
  query: string,
  currentPage: number,
) {
  const offset = (currentPage - 1) * ITEMS_PER_PAGE;

  const invoices = await sql<InvoicesTable[]>`
    SELECT
      invoices.id,
      invoices.amount,
      invoices.date,
      invoices.status,
      customers.name,
      customers.email,
      customers.image_url
    FROM invoices
    JOIN customers ON invoices.customer_id = customers.id
    WHERE
      customers.name ILIKE ${`%${query}%`} OR
      customers.email ILIKE ${`%${query}%`} OR
      invoices.amount::text ILIKE ${`%${query}%`} OR
      invoices.date::text ILIKE ${`%${query}%`} OR
      invoices.status ILIKE ${`%${query}%`}
    ORDER BY invoices.date DESC
    LIMIT ${ITEMS_PER_PAGE} OFFSET ${offset}
  `;

  return invoices;
}

// Fetch total halaman
export async function fetchInvoicesPages(query: string) {
  const data = await sql`
    SELECT COUNT(*)
    FROM invoices
    JOIN customers ON invoices.customer_id = customers.id
    WHERE
      customers.name ILIKE ${`%${query}%`} OR
      customers.email ILIKE ${`%${query}%`} OR
      invoices.amount::text ILIKE ${`%${query}%`} OR
      invoices.date::text ILIKE ${`%${query}%`} OR
      invoices.status ILIKE ${`%${query}%`}
  `;

  const totalPages = Math.ceil(Number(data[0].count) / ITEMS_PER_PAGE);
  return totalPages;
}
```

---

## 🧩 Bagian 4: Konsep Penting

### 4.1 Client vs Server Components

| Aspek | Client Component | Server Component |
|-------|-----------------|------------------|
| **Direktif** | `'use client'` di baris pertama | Default (tanpa direktif) |
| **Hooks** | ✅ Bisa pakai `useState`, `useEffect`, dll | ❌ Tidak bisa |
| **URL Hooks** | ✅ `useSearchParams`, `usePathname`, `useRouter` | ❌ Gunakan `props.searchParams` |
| **Data Fetching** | `fetch` di `useEffect` | `await` langsung di komponen |
| **Contoh** | `Search`, `Pagination` | `Page`, `Table` |

### 4.2 Mengapa Search & Pagination adalah Client Component?

1. **Interaktivitas**: Perlu menangani event (`onChange`, `onClick`)
2. **URL Manipulation**: Perlu `useRouter` untuk mengubah URL
3. **Real-time State**: Perlu membaca `useSearchParams` yang bisa berubah

### 4.3 Pola URL Search Params

```
/dashboard/invoices                           → Default (page 1, no search)
/dashboard/invoices?query=michael             → Search "michael", page 1
/dashboard/invoices?page=2                    → Page 2, no search
/dashboard/invoices?query=michael&page=2      → Search "michael", page 2
```

---

## 📝 Bagian 5: Best Practices

### 5.1 Reset Page Saat Search Berubah

```typescript
const handleSearch = useDebouncedCallback((term: string) => {
  const params = new URLSearchParams(searchParams);
  params.set('page', '1');  // ⬅️ Selalu reset ke halaman 1
  // ... rest of code
}, 300);
```

**Mengapa?** Jika user di halaman 5 lalu search sesuatu, hasil mungkin hanya 1 halaman. Tanpa reset, user akan stuck di halaman kosong.

### 5.2 Gunakan defaultValue, Bukan value

```tsx
<input
  defaultValue={searchParams.get('query')?.toString()}  // ✅ Benar
  // value={...}  // ❌ Salah - akan mengontrol input sepenuhnya
/>
```

**Mengapa?** `defaultValue` hanya set nilai awal, user tetap bisa mengetik bebas. `value` membuat input terkontrol (controlled) dan perlu state management.

### 5.3 Key pada Suspense untuk Re-render

```tsx
<Suspense key={query + currentPage} fallback={<InvoicesTableSkeleton />}>
  <Table query={query} currentPage={currentPage} />
</Suspense>
```

**Mengapa?** `key` yang berubah memaksa React me-render ulang komponen, memicu loading state baru saat query/page berubah.

---

## 🎯 Ringkasan

1. **Search** menggunakan `useDebouncedCallback` untuk efisiensi dan `useRouter.replace()` untuk update URL
2. **Pagination** menggunakan `Link` dengan URL yang berisi parameter `page`
3. **Server Component** membaca `searchParams` untuk fetch data yang sesuai
4. **URL sebagai Single Source of Truth** - state pencarian dan halaman tersimpan di URL
5. **Debouncing** mencegah request berlebihan saat user mengetik

---

## 🔧 Troubleshooting Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| Pagination tidak berpindah halaman | `params.set('page', ...)` tidak dipanggil | Tambahkan `params.set('page', pageNumber.toString())` di `createPageURL` |
| Search tidak bekerja | `replace()` tidak dipanggil | Pastikan `replace(\`${pathname}?${params.toString()}\`)` ada |
| Input search kosong saat reload | `defaultValue` tidak di-set | Tambahkan `defaultValue={searchParams.get('query')?.toString()}` |
| Terlalu banyak request | Tidak ada debouncing | Gunakan `useDebouncedCallback` dari `use-debounce` |
