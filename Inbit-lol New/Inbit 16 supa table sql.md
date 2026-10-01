create table if not exists bids (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  tagline text not null,
  url text not null,
  category text not null,
  feedback_ask text not null,
  perk text,
  amount integer not null check (amount > 0),
  status text not null default 'pending' check (status in ('pending','paid')),
  razorpay_order_id text unique,
  clicks integer not null default 0,
  feedback_count integer not null default 0,
  created_at timestamptz not null default now(),
  paid_at timestamptz
);

create table if not exists feedback (
  id uuid primary key default gen_random_uuid(),
  bid_id uuid not null references bids(id) on delete cascade,
  message text not null,
  created_at timestamptz not null default now()
);

create index if not exists bids_paid_rank_idx on bids (status, amount desc, paid_at asc);