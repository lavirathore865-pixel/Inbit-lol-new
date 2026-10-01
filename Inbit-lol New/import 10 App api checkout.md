import { NextResponse } from 'next/server';
import { supabase } from '@/lib/supabase';
import { validateBidInput } from '@/lib/validate';
import { allow } from '@/lib/ratelimit';

export async function POST(req) {
  if (!(await allow('checkout', req))) {
    return NextResponse.json({ error: 'Too many attempts. Please try again in a while.' }, { status: 429 });
  }

  let body;
  try {
    body = await req.json();
  } catch {
    return NextResponse.json({ error: 'Invalid request.' }, { status: 400 });
  }

  const { error: vError, value } = validateBidInput(body);
  if (vError) return NextResponse.json({ error: vError }, { status: 400 });

  // Server-side rule: the bid must beat the current #1.
  const { data: top, error: topError } = await supabase
    .from('bids')
    .select('amount')
    .eq('status', 'paid')
    .order('amount', { ascending: false })
    .limit(1)
    .maybeSingle();

  if (topError) {
    console.error('top bid error:', topError);
    return NextResponse.json({ error: 'Could not check the current #1.' }, { status: 500 });
  }
  const minBid = top ? top.amount + 1 : 1;
  if (value.amount < minBid) {
    return NextResponse.json({ error: `Your bid must be at least ₹${minBid}.` }, { status: 400 });
  }

  // Save as pending. It goes live only when the webhook confirms payment.
  const { data: bid, error: insertError } = await supabase
    .from('bids')
    .insert({
      name: value.name,
      tagline: value.tagline,
      url: value.url,
      category: value.category,
      feedback_ask: value.feedbackAsk,
      perk: value.perk || null,
      amount: value.amount,
      status: 'pending',
    })
    .select('id')
    .single();

  if (insertError) {
    console.error('insert error:', insertError);
    return NextResponse.json({ error: 'Could not save your bid.' }, { status: 500 });
  }

  try {
    const auth = Buffer.from(
      `${process.env.RAZORPAY_KEY_ID}:${process.env.RAZORPAY_KEY_SECRET}`
    ).toString('base64');

    const rzpRes = await fetch('https://api.razorpay.com/v1/orders', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Basic ${auth}` },
      body: JSON.stringify({
        amount: value.amount * 100, // paise
        currency: 'INR',
        receipt: bid.id,
        notes: { bid_id: bid.id },
      }),
    });
    const order = await rzpRes.json();
    if (!rzpRes.ok || !order.id) throw new Error(JSON.stringify(order));

    await supabase.from('bids').update({ razorpay_order_id: order.id }).eq('id', bid.id);

    return NextResponse.json({
      orderId: order.id,
      amount: order.amount,
      keyId: process.env.RAZORPAY_KEY_ID,
    });
  } catch (err) {
    console.error('razorpay error:', err);
    await supabase.from('bids').delete().eq('id', bid.id).eq('status', 'pending');
    return NextResponse.json({ error: 'Payment could not be started. Please try again.' }, { status: 500 });
  }
}