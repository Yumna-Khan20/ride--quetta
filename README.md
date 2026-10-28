# RideQuetta &mdash; Production-Grade Rickshaw Booking Website

RideQuetta is a Quetta-tailored rickshaw booking platform built as a clean, high-craft, single-file static web application (`index.html`).

---

## 1. Quick Start (Instant Local Running)

- **Double-Click**: Open `index.html` in any web browser (Chrome, Safari, Edge, Firefox). No build step, node modules, or local server required.
- **Local Server (Optional)**:
  \`\`\`bash
  # Using Node (npx)
  npx serve .
  # Or using Python 3
  python -m http.server 8080
  \`\`\`

---

## 2. Deploying to Netlify & Vercel

### Netlify (Drag-and-Drop or Git)
1. Go to [app.netlify.com](https://app.netlify.com).
2. Go to **Sites** &rarr; Drag and drop the folder containing `index.html` directly into the upload area.
3. Your site will be instantly live with global HTTPS.
4. If deploying via GitHub:
   - **Build Command**: *Leave blank*
   - **Publish Directory**: `.` (or root)

### Vercel (Git Import)
1. Push `index.html` to a new GitHub repository.
2. Go to [vercel.com/new](https://vercel.com/new) and select your repository.
3. Under **Build & Development Settings**:
   - **Framework Preset**: `Other`
   - **Build Command**: *Disabled / Empty*
   - **Output Directory**: *Disabled / Empty (defaults to root)*
4. Click **Deploy**.

---

## 3. Configuration Reference (`CONFIG` in `index.html`)

At the very top of the `<script>` block in `index.html`, locate the `CONFIG` object:

\`\`\`javascript
const CONFIG = {
  pricing: {
    baseFare: 70,           // Base departure rate (PKR)
    perKm: 35,              // Rate per kilometer (PKR)
    minFare: 100,           // Minimum trip fare (PKR)
    rideTypeMultiplier: {
      standard: 1.0,        // Dedicated rickshaw
      shared: 0.75,         // Shared ride (route-matched)
      premium: 1.4          // Reserved for future premium cabin rickshaws
    },
    waitingPerMinute: 0     // Waiting charge if applicable
  },
  contact: {
    phone: '',              // e.g. "+92 81 2800000"
    whatsapp: '',           // e.g. "923001234567" (digits only for WhatsApp direct link)
    email: 'contact@ridequetta.local',
    address: 'Quetta, Balochistan, Pakistan',
    socials: { facebook: '', instagram: '', twitter: '' }
  },
  services: {
    premiumEnabled: false
  },
  backend: {
    provider: 'supabase',
    supabaseUrl: '',        // 'https://YOUR_PROJECT.supabase.co'
    supabaseAnonKey: ''     // 'eyJhbGciOi...' (public anon key only)
  }
};
\`\`\`

> [!NOTE]
> When `backend.supabaseUrl` and `supabaseAnonKey` are empty, RideQuetta automatically operates in **Demo Mode**, safely storing requests in your browser's `localStorage` and displaying a dismissible notification banner.

---

## 4. Supabase Setup & Production SQL Schema

To transition from Demo Mode to live database storage:

1. Create a project at [supabase.com](https://supabase.com).
2. Go to **SQL Editor** and run the following schema:

\`\`\`sql
-- 1. BOOKINGS TABLE
create table if not exists public.bookings (
  id uuid primary key default gen_random_uuid(),
  reference text unique not null,
  pickup text not null,
  destination text not null,
  ride_type text not null check (ride_type in ('standard', 'shared', 'premium')),
  passenger_name text not null,
  passenger_phone text not null,
  trip_notes text,
  estimated_fare integer not null,
  distance_km numeric(5,2),
  is_route_distance boolean default false,
  status text not null default 'awaiting_confirmation' 
    check (status in ('submitted', 'awaiting_confirmation', 'confirmed', 'cancelled', 'completed')),
  created_at timestamptz default now()
);

-- 2. DRIVER REGISTRATIONS TABLE
create table if not exists public.driver_registrations (
  id uuid primary key default gen_random_uuid(),
  reference text unique not null,
  full_name text not null,
  mobile text not null,
  cnic_masked text,
  vehicle_reg text not null,
  vehicle_type text not null,
  service_areas text[] default '{}',
  licence_status text not null,
  message text,
  status text not null default 'pending_review'
    check (status in ('pending_review', 'contacted', 'approved', 'rejected')),
  created_at timestamptz default now()
);

-- 3. CONTACT MESSAGES TABLE
create table if not exists public.contact_messages (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  contact text not null,
  message text not null,
  created_at timestamptz default now()
);

-- 4. ROW LEVEL SECURITY (RLS) POLICIES
alter table public.bookings enable row level security;
alter table public.driver_registrations enable row level security;
alter table public.contact_messages enable row level security;

-- Anonymous users may INSERT booking requests
create policy "Allow anonymous booking submissions"
  on public.bookings for insert
  to anon
  with check (true);

-- Anonymous users may INSERT driver applications
create policy "Allow anonymous driver registrations"
  on public.driver_registrations for insert
  to anon
  with check (true);

-- Anonymous users may INSERT contact messages
create policy "Allow anonymous contact messages"
  on public.contact_messages for insert
  to anon
  with check (true);

-- 5. SECURE STATUS LOOKUP FUNCTION (NO SENSITIVE DATA EXPOSURE)
create or replace function public.get_booking_status(ref text)
returns json
language plpgsql
security definer
as $$
declare
  booking_record record;
begin
  select reference, pickup, destination, ride_type, status, created_at
  into booking_record
  from public.bookings
  where upper(reference) = upper(ref)
  limit 1;

  if not found then
    return null;
  end if;

  return row_to_json(booking_record);
end;
$$;
\`\`\`

3. In Supabase Dashboard &rarr; **Project Settings** &rarr; **API**:
   - Copy **Project URL** into `CONFIG.backend.supabaseUrl`.
   - Copy **anon / public** key into `CONFIG.backend.supabaseAnonKey`.
   *(Never expose or use your service_role secret key in client-side code).*

---

## 5. Startup Honesty & Production Supply Checklist

The codebase strictly adheres to startup honesty standards: no fabricated stats, no fake driver headcounts, and no false guarantees. Before real-world commercial launch, the business owner must provide:

- [ ] **Confirmed Rates**: Update `CONFIG.pricing` with verified base and per-km prices agreed upon by local rickshaw operators.
- [ ] **Contact Channels**: Insert verified phone, WhatsApp number, physical address, and social links.
- [ ] **Coordinates & Routing**: Verify approximate coordinates in `LOCATIONS` against specific local landmarks; migrate from the public demo OSRM instance to a paid or self-hosted routing endpoint for high volume.
- [ ] **Legal Documents**: Review and replace the draft texts in the Terms of Service and Privacy Policy modals with counsel-approved documentation.
- [ ] **Driver Onboarding Process**: Establish the standard operating procedure for driver vehicle inspection and CNIC verification before updating driver status in Supabase.
