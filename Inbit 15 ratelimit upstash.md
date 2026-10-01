import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

// Rate limiting is optional: if the Upstash env vars are missing, requests are allowed.
const enabled = Boolean(process.env.UPSTASH_REDIS_REST_URL && process.env.UPSTASH_REDIS_REST_TOKEN);
const redis = enabled ? Redis.fromEnv() : null;

function make(prefix, requests, window) {
  if (!redis) return null;
  return new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(requests, window),
    prefix: `inbit:${prefix}`,
  });
}

const limiters = {
  feedback: make('feedback', 5, '1 m'),   // 5 feedback messages per minute per IP
  checkout: make('checkout', 10, '1 h'),  // 10 checkout attempts per hour per IP
};

export function getIp(req) {
  const fwd = req.headers.get('x-forwarded-for');
  return (fwd ? fwd.split(',')[0].trim() : req.headers.get('x-real-ip')) || 'unknown';
}

// Returns true if the request is allowed.
export async function allow(name, req) {
  const limiter = limiters[name];
  if (!limiter) return true;
  try {
    const { success } = await limiter.limit(getIp(req));
    return success;
  } catch (err) {
    console.error('ratelimit error:', err);
    return true; // do not block real users if Upstash is down
  }
}