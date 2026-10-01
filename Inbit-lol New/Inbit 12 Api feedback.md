import { NextResponse } from 'next/server';
import { supabase } from '@/lib/supabase';
import { allow } from '@/lib/ratelimit';

export async function POST(req) {
  if (!(await allow('feedback', req))) {
    return NextResponse.json({ error: 'You are sending feedback too fast. Wait a minute and try again.' }, { status: 429 });
  }

  let body;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'Invalid request.' }, { status: 400 });
  }

  const bidId = typeof body?.bidId === 'string' ? body.bidId : '';
  const message = typeof body?.message === 'string' ? body.message.trim().slice(0, 1000) : '';
  if (!bidId || !message) {
    return NextResponse.json({ error: 'Feedback message is required.' }, { status: 400 });
  }

  // Only accept feedback for live (paid) listings.
  const { data: bid } = await supabase
    .from('bids').select('id').eq('id', bidId).eq('status', 'paid').maybeSingle();
  if (!bid) return NextResponse.json({ error: 'Listing not found.' }, { status: 404 });

  const { error } = await supabase.from('feedback').insert({ bid_id: bidId, message });
  if (error) {
    console.error('feedback error:', error);
    return NextResponse.json({ error: 'Could not save feedback.' }, { status: 500 });
  }

  const { count } = await supabase
    .from('feedback').select('id', { count: 'exact', head: true }).eq('bid_id', bidId);
  await supabase.from('bids').update({ feedback_count: count ?? 0 }).eq('id', bidId);

  return NextResponse.json({ ok: true });
}