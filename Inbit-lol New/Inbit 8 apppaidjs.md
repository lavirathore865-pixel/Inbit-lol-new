'use client';
import { useState, useEffect, useMemo } from 'react';

const CATEGORIES = ['All', 'Indie Apps', 'SaaS Tools', 'Dev Tools', 'AI Tools', 'Digital Products'];
const FORM_CATEGORIES = CATEGORIES.filter((c) => c !== 'All');

const DEMO_BIDS = [
  {
    id: 1, rank: 1, name: 'DevFlow SaaS', tagline: 'Automated CI/CD pipeline',
    category: 'Dev Tools', amount: 150, url: '#',
    feedbackAsk: 'Is the onboarding clear in the first 2 minutes?',
    perk: '3 months free for Inbit users', clicks: 0, feedbackCount: 0,
  },
  {
    id: 2, rank: 2, name: 'PixelCraft AI', tagline: 'Generate UI assets',
    category: 'AI Tools', amount: 100, url: '#',
    feedbackAsk: 'Which export formats are you missing?',
    perk: '50 free credits', clicks: 0, feedbackCount: 0,
  },
];

const VALUE_POINTS = [
  {
    title: 'Targeted traffic',
    text: 'Visitors here are founders, makers and early adopters looking for new tools. No random clicks, no bots chasing a leaderboard.',
  },
  {
    title: 'Real first users',
    text: 'Attach a launch perk (free trial, discount, credits) so visitors have a reason to actually sign up, not just look.',
  },
  {
    title: 'Genuine feedback',
    text: 'Tell us the one question you want answered. It is shown to every visitor next to your product, so you get answers you can use.',
  },
];

const HOW_IT_WORKS = [
  'Submit your product, pick a niche, and say what feedback you need.',
  'Bid above the current #1 and pay through secure checkout.',
  'Your product takes the top spot as soon as payment is confirmed.',
  'Visitors try it, claim your perk and leave feedback on your listing.',
];

function isHttpUrl(value) {
  try {
    const u = new URL(value);
    return u.protocol === 'http:' || u.protocol === 'https:';
  } catch {
    return false;
  }
}

function safeHref(value) {
  return isHttpUrl(value) ? value : '#';
}

export default function Home() {
  const [bids, setBids] = useState([]);
  const [loading, setLoading] = useState(true);
  const [loadError, setLoadError] = useState('');
  const [isDemo, setIsDemo] = useState(false);
  const [filter, setFilter] = useState('All');

  const [name, setName] = useState('');
  const [tagline, setTagline] = useState('');
  const [url, setUrl] = useState('');
  const [category, setCategory] = useState(FORM_CATEGORIES[0]);
  const [feedbackAsk, setFeedbackAsk] = useState('');
  const [perk, setPerk] = useState('');
  const [bidAmount, setBidAmount] = useState('');
  const [formError, setFormError] = useState('');
  const [submitting, setSubmitting] = useState(false);

  const [feedbackText, setFeedbackText] = useState('');
  const [feedbackStatus, setFeedbackStatus] = useState('');
  const [banner, setBanner] = useState('');

  useEffect(() => {
    const q = new URLSearchParams(window.location.search);
    if (q.get('paid')) setBanner('Payment received. Your product goes live at the top as soon as it is confirmed.');
    else if (q.get('canceled')) setBanner('Checkout was canceled. Your bid was not charged.');
  }, []);

  useEffect(() => {
    fetchBids();
  }, []);

  const fetchBids = async () => {
    try {
      const res = await fetch('/api/bids');
      if (!res.ok) throw new Error('Bad response');
      const data = await res.json();
      if (Array.isArray(data) && data.length > 0) {
        setBids(data);
      } else {
        setBids(DEMO_BIDS);
        setIsDemo(true);
      }
    } catch (err) {
      console.error('Error fetching bids:', err);
      setLoadError('Could not load the live board. Showing sample listings.');
      setBids(DEMO_BIDS);
      setIsDemo(true);
    } finally {
      setLoading(false);
    }
  };

  const leader = bids[0];
  const minBid = leader ? Number(leader.amount) + 1 : 1;

  const queue = useMemo(() => {
    const rest = bids.slice(1);
    return filter === 'All' ? rest : rest.filter((b) => b.category === filter);
  }, [bids, filter]);

  const handleCheckout = async (e) => {
    e.preventDefault();
    setFormError('');

    if (!name.trim() || !tagline.trim() || !url.trim() || !bidAmount) {
      setFormError('Fill in the project name, tagline, URL and bid amount.');
      return;
    }
    if (!isHttpUrl(url)) {
      setFormError('Enter a full project URL starting with http:// or https://');
      return;
    }
    if (!feedbackAsk.trim()) {
      setFormError('Add the one question you want visitors to answer. This is what gets you useful feedback.');
      return;
    }
    if (Number(bidAmount) < minBid) {
      setFormError(`Your bid must be at least ₹${minBid} to beat the current #1.`);
      return;
    }

    setSubmitting(true);
    try {
      const res = await fetch('/api/checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name: name.trim(),
          tagline: tagline.trim(),
          url: url.trim(),
          category,
          feedbackAsk: feedbackAsk.trim(),
          perk: perk.trim(),
          amount: Number(bidAmount),
        }),
      });
      const data = await res.json();

      if (!res.ok || !data.orderId) {
        setFormError(data.error || 'Payment could not be started. Please try again.');
        return;
      }
      if (typeof window.Razorpay === 'undefined') {
        setFormError('Payment window is still loading. Wait a moment and try again.');
        return;
      }

      const rzp = new window.Razorpay({
        key: data.keyId,
        amount: data.amount,
        currency: 'INR',
        name: 'Inbit.lol',
        description: `#1 Launch Spotlight: ${name.trim()}`,
        order_id: data.orderId,
        theme: { color: '#10b981' },
        handler: () => {
          setBanner('Payment received. Your product goes live at #1 in a few seconds.');
          setName(''); setTagline(''); setUrl(''); setFeedbackAsk(''); setPerk(''); setBidAmount('');
          setTimeout(fetchBids, 5000);
          setTimeout(fetchBids, 12000);
        },
      });
      rzp.open();
    } catch (err) {
      console.error('Checkout error:', err);
      setFormError('Something went wrong. Please try again.');
    } finally {
      setSubmitting(false);
    }
  };

  const sendFeedback = async (e) => {
    e.preventDefault();
    if (!feedbackText.trim() || !leader || isDemo) return;
    setFeedbackStatus('Sending...');
    try {
      const res = await fetch('/api/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ bidId: leader.id, message: feedbackText.trim() }),
      });
      const out = await res.json().catch(() => ({}));
      if (!res.ok) {
        setFeedbackStatus(out.error || 'Could not send feedback. Try again.');
        return;
      }
      setFeedbackText('');
      setFeedbackStatus('Feedback sent to the maker.');
    } catch {
      setFeedbackStatus('Could not send feedback. Try again.');
    }
  };

  const inputClass =
    'w-full bg-zinc-950 border border-zinc-800 rounded-xl p-3 text-white text-sm focus:outline-none focus:border-emerald-500';
  const labelClass = 'block text-xs font-semibold text-gray-400 mb-1';

  return (
    <main className="min-h-screen bg-zinc-950 text-white p-6 md:p-24 font-sans">
      <div className="max-w-3xl mx-auto">
        {banner && (
          <p className="mb-6 rounded-xl border border-emerald-500/30 bg-emerald-500/10 px-4 py-3 text-sm text-emerald-300" role="status">
            {banner}
          </p>
        )}
        {/* Hero */}
        <div className="text-center mb-12">
          <span className="text-xs bg-emerald-500/10 text-emerald-400 border border-emerald-500/20 px-3 py-1 rounded-full font-semibold">
            Launch board for indie apps, startup tools and digital products
          </span>
          <h1 className="text-4xl md:text-5xl font-extrabold tracking-tight mt-4 mb-3 bg-gradient-to-r from-emerald-400 to-cyan-500 bg-clip-text text-transparent">
            Inbit.lol
          </h1>
          <p className="text-gray-300 max-w-xl mx-auto text-sm md:text-base">
            Not just a ranking. Win the #1 spotlight and get targeted traffic, real first users and honest feedback from people who actually build and use new products.
          </p>
        </div>

        {/* Value points */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-12">
          {VALUE_POINTS.map((v) => (
            <div key={v.title} className="bg-zinc-900/60 border border-zinc-800/80 rounded-2xl p-5">
              <h3 className="text-sm font-bold text-white mb-1">{v.title}</h3>
              <p className="text-xs text-gray-400 leading-relaxed">{v.text}</p>
            </div>
          ))}
        </div>

        {loadError && (
          <p className="text-xs text-amber-400 mb-4" role="status">{loadError}</p>
        )}
        {isDemo && !loading && (
          <p className="text-xs text-gray-500 mb-4">
            These are sample listings. The first paid bid becomes the real #1.
          </p>
        )}

        {/* Spotlight */}
        {leader && (
          <div className="bg-gradient-to-r from-emerald-950/30 to-cyan-950/30 border border-emerald-500/40 rounded-2xl p-6 md:p-8 mb-8 shadow-2xl">
            <div className="flex flex-wrap items-center gap-2 mb-3">
              <span className="bg-emerald-600 text-xs font-bold px-3 py-1 rounded-full text-white">
                #1 Launch Spotlight
              </span>
              {leader.category && (
                <span className="text-xs border border-zinc-700 text-gray-300 px-3 py-1 rounded-full">
                  {leader.category}
                </span>
              )}
            </div>
            <h2 className="text-3xl font-bold text-white">{leader.name}</h2>
            <p className="text-gray-300 text-sm mt-1">{leader.tagline}</p>

            {leader.perk && (
              <div className="mt-5 rounded-xl border border-emerald-500/30 bg-emerald-500/5 p-4">
                <span className="text-xs font-semibold text-emerald-400">Launch perk for Inbit visitors</span>
                <p className="text-sm text-white mt-1">{leader.perk}</p>
              </div>
            )}

            {leader.feedbackAsk && (
              <div className="mt-4 rounded-xl border border-zinc-700/70 bg-zinc-900/60 p-4">
                <span className="text-xs font-semibold text-cyan-400">The maker wants your feedback on</span>
                <p className="text-sm text-white mt-1">{leader.feedbackAsk}</p>
                <form onSubmit={sendFeedback} className="mt-3 flex flex-col sm:flex-row gap-2">
                  <input
                    type="text"
                    value={feedbackText}
                    onChange={(e) => setFeedbackText(e.target.value)}
                    placeholder="Your honest answer"
                    aria-label="Your feedback for the maker"
                    className={inputClass}
                    disabled={isDemo}
                  />
                  <button
                    type="submit"
                    disabled={isDemo}
                    className="bg-zinc-800 hover:bg-zinc-700 disabled:opacity-40 text-white font-semibold px-5 py-3 rounded-xl text-sm transition"
                  >
                    Send feedback
                  </button>
                </form>
                {feedbackStatus && (
                  <p className="text-xs text-gray-400 mt-2" role="status">{feedbackStatus}</p>
                )}
              </div>
            )}

            <div className="mt-6 flex items-center justify-between flex-wrap gap-4">
              <div className="text-sm text-gray-400 space-x-4">
                <span>
                  Winning bid: <span className="text-emerald-400 font-bold text-lg">₹{leader.amount}</span>
                </span>
                {typeof leader.clicks === 'number' && <span>{leader.clicks} visits</span>}
                {typeof leader.feedbackCount === 'number' && <span>{leader.feedbackCount} feedback</span>}
              </div>
              <a
                href={safeHref(leader.url)}
                target="_blank"
                rel="noopener noreferrer"
                className="bg-emerald-500 hover:bg-emerald-400 text-black font-bold px-6 py-2.5 rounded-xl text-sm transition shadow-lg shadow-emerald-500/20"
              >
                Try {leader.name}
              </a>
            </div>
          </div>
        )}

        {/* Queue with niche filter */}
        <div className="bg-zinc-900/60 border border-zinc-800/80 rounded-2xl p-4 backdrop-blur-md mb-12">
          <div className="flex justify-between items-center px-4 mb-3 flex-wrap gap-2">
            <h3 className="text-sm font-semibold text-gray-400">Live product queue</h3>
            <span className="text-xs text-gray-500">Updates when a payment succeeds</span>
          </div>

          <div className="flex flex-wrap gap-2 px-4 mb-3" role="tablist" aria-label="Filter by niche">
            {CATEGORIES.map((c) => (
              <button
                key={c}
                type="button"
                role="tab"
                aria-selected={filter === c}
                onClick={() => setFilter(c)}
                className={`text-xs px-3 py-1.5 rounded-full border transition ${
                  filter === c
                    ? 'bg-emerald-500 text-black border-emerald-500 font-semibold'
                    : 'border-zinc-700 text-gray-400 hover:text-white'
                }`}
              >
                {c}
              </button>
            ))}
          </div>

          <div className="divide-y divide-zinc-800/60">
            {loading && <p className="text-sm text-gray-500 px-4 py-6">Loading the board...</p>}
            {!loading && queue.length === 0 && (
              <p className="text-sm text-gray-500 px-4 py-6">
                {filter === 'All'
                  ? 'No products in the queue yet. Bid below to be next.'
                  : `No ${filter} listed yet. Bid below to lead this niche.`}
              </p>
            )}
            {queue.map((item, index) => (
              <div
                key={item.id || index}
                className="flex items-center justify-between gap-4 py-4 px-4 hover:bg-zinc-800/40 rounded-xl transition"
              >
                <div className="flex items-center gap-4 min-w-0">
                  <span className="text-gray-500 font-bold w-6 shrink-0">#{item.rank || bids.indexOf(item) + 1}</span>
                  <div className="min-w-0">
                    <a
                      href={safeHref(item.url)}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="font-semibold text-white block hover:text-emerald-400 truncate"
                    >
                      {item.name}
                    </a>
                    <span className="text-xs text-gray-400 block truncate">{item.tagline}</span>
                    {item.perk && <span className="text-xs text-emerald-400 block truncate">Perk: {item.perk}</span>}
                  </div>
                </div>
                <div className="text-right shrink-0">
                  <span className="text-emerald-400 font-semibold text-sm block">₹{item.amount}</span>
                  {item.category && <span className="text-xs text-gray-500">{item.category}</span>}
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* How it works */}
        <div className="mb-12">
          <h3 className="text-xl font-bold text-white mb-4">How a launch works here</h3>
          <ol className="space-y-3">
            {HOW_IT_WORKS.map((step, i) => (
              <li key={step} className="flex gap-4 text-sm text-gray-300">
                <span className="text-emerald-400 font-bold w-5 shrink-0">{i + 1}</span>
                <span>{step}</span>
              </li>
            ))}
          </ol>
        </div>

        {/* Submit form */}
        <div className="bg-zinc-900 border border-zinc-800 rounded-2xl p-6 md:p-8">
          <h3 className="text-xl font-bold text-white mb-2">Launch your product at #1</h3>
          <p className="text-xs text-gray-400 mb-6">
            For indie apps, startup tools and digital products only. Bid at least{' '}
            <span className="text-white font-semibold">₹{minBid}</span> to take the top spot after payment.
          </p>

          <form onSubmit={handleCheckout} className="space-y-4" noValidate>
            <div>
              <label htmlFor="name" className={labelClass}>Project name</label>
              <input id="name" type="text" placeholder="e.g., My Awesome SaaS" value={name}
                onChange={(e) => setName(e.target.value)} className={inputClass} />
            </div>
            <div>
              <label htmlFor="tagline" className={labelClass}>What it does, in one line</label>
              <input id="tagline" type="text" placeholder="e.g., Build apps 10x faster" value={tagline}
                onChange={(e) => setTagline(e.target.value)} className={inputClass} />
            </div>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              <div>
                <label htmlFor="url" className={labelClass}>Project URL</label>
                <input id="url" type="url" placeholder="https://yourwebsite.com" value={url}
                  onChange={(e) => setUrl(e.target.value)} className={inputClass} />
              </div>
              <div>
                <label htmlFor="category" className={labelClass}>Niche</label>
                <select id="category" value={category} onChange={(e) => setCategory(e.target.value)} className={inputClass}>
                  {FORM_CATEGORIES.map((c) => (
                    <option key={c} value={c}>{c}</option>
                  ))}
                </select>
              </div>
            </div>
            <div>
              <label htmlFor="feedbackAsk" className={labelClass}>The one question you want visitors to answer</label>
              <input id="feedbackAsk" type="text" placeholder="e.g., Is the pricing page clear?" value={feedbackAsk}
                onChange={(e) => setFeedbackAsk(e.target.value)} className={inputClass} maxLength={140} />
            </div>
            <div>
              <label htmlFor="perk" className={labelClass}>Launch perk for visitors (optional, but it brings real signups)</label>
              <input id="perk" type="text" placeholder="e.g., 30% off for 3 months" value={perk}
                onChange={(e) => setPerk(e.target.value)} className={inputClass} maxLength={100} />
            </div>
            <div>
              <label htmlFor="bid" className={labelClass}>Your bid amount (₹)</label>
              <input id="bid" type="number" min={minBid} placeholder={`At least ${minBid}`} value={bidAmount}
                onChange={(e) => setBidAmount(e.target.value)} className={inputClass} />
            </div>

            {formError && (
              <p className="text-sm text-red-400" role="alert">{formError}</p>
            )}

            <button
              type="submit"
              disabled={submitting}
              className="w-full mt-4 bg-gradient-to-r from-emerald-500 to-cyan-600 hover:opacity-90 disabled:opacity-50 text-black font-extrabold py-4 px-8 rounded-2xl shadow-xl transition transform active:scale-95 text-base cursor-pointer"
            >
              {submitting ? 'Opening payment...' : 'Pay with UPI or card and take #1'}
            </button>
          </form>
        </div>
      </div>
    </main>
  );
}