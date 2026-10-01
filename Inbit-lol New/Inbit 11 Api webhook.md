import crypto from 'crypto';
import { NextResponse } from 'next/server';
import { supabase } from '@/lib/supabase';

export async function POST(req) {
  const raw = await req.text(); // raw body is required for signature verification
  const signature = req.headers.get('x-razorpay-signature') || '';

  const expected = crypto
    .createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET || '')
    .update(raw)
    .digest('hex');

  const a = Buffer.from(signature);
  const b = Buffer.from(expected);
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
    return NextResponse.json({ error: 'Invalid signature.' }, { status: 400 });
  }

  const event = JSON.parse(raw);

  if (event.event === 'order.paid') {
    const order = event.payload?.order?.entity;
    const bidId = order?.notes?.bid_id;

    if (bidId) {
      const { data: bid } = await supabase
        .from('bids').select('amount, status').eq('id', bidId).maybeSingle();

      // Only go live if the full amount was actually paid.
      if (bid && bid.status === 'pending' && order.amount_paid >= bid.amount * 100) {
        const { error } = await supabase
          .from('bids')
          .update({ status: 'paid', paid_at: new Date().toISOString() })
          .eq('id', bidId)
          .eq('status', 'pending'); // idempotent

        if (error) {
          console.error('webhook update error:', error);
          return NextResponse.json({ error: 'Update failed.' }, { status: 500 }); // Razorpay retries
        }
      }
    }
  }

  return NextResponse.json({ received: true });
}