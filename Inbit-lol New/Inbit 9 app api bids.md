import { NextResponse } from 'next/server';
import { supabase } from '@/lib/supabase';

export const dynamic = 'force-dynamic';

export async function GET() {
  const { data, error } = await supabase
    .from('bids')
    .select('id, name, tagline, url, category, feedback_ask, perk, amount, clicks, feedback_count')
    .eq('status', 'paid')
    .order('amount', { ascending: false })
    .order('paid_at', { ascending: true })
    .limit(50);

  if (error) {
    console.error('bids error:', error);
    return NextResponse.json({ error: 'Could not load bids.' }, { status: 500 });
  }

  const bids = data.map((b, i) => ({
    id: b.id,
    rank: i + 1,
    name: b.name,
    tagline: b.tagline,
    url: b.url,
    category: b.category,
    feedbackAsk: b.feedback_ask,
    perk: b.perk,
    amount: b.amount,
    clicks: b.clicks,
    feedbackCount: b.feedback_count,
  }));

  return NextResponse.json(bids);
}