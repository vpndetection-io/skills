---
name: block-vpn-signups
description: Adds a VPNDetection check to an app's sign-up, so accounts from VPNs, Tor exits and relays (and residential proxies and datacenters, with a paid key) are refused, using the official library for Next.js, Express, Better Auth, Django, Laravel or Rails. Use when someone wants to block VPN or proxy sign-ups, stop fake or duplicate accounts, or add an IP check to a registration form.
---

# Block VPN and proxy sign-ups

Check the address only where an account is created, never on every page, and let the sign-up through when the lookup fails: an outage of ours should not stop anyone registering.

1. Find the handler that creates the account (a form action, a `POST` route, an auth library's sign-up endpoint) and the framework, then add the matching block below. Read the key from `VPNDETECTION_API_KEY`. Without one, lookups run on the free plan, which answers `is_vpn`, `is_tor` and `is_relay` with no cap.
2. Work out where the client's address comes from before trusting it. Behind a proxy or CDN the socket peer is the proxy, so set the framework's trusted-proxy option, or read the header the edge overwrites (`CF-Connecting-IP` on Cloudflare). Never take the left-most `X-Forwarded-For` an edge of yours doesn't overwrite: the caller wrote it.
3. With a paid key, offer to add residential proxies and datacenters (`is_resproxy` and `is_hosting`, `isResproxy` and `isHosting` in JavaScript and PHP). The free plan doesn't answer them, so on it they never match.
4. Tell the user what now happens to a refused sign-up, and that the default refuses with a `403`, which they may prefer to replace with a message on the form or a flag for review.

## Next.js

`npm install vpndetection-next`, then run the proxy on the sign-up page alone (`proxy.ts` on Next 16, `middleware.ts` exporting `middleware` on 15), and refuse in the server action:

```ts
// proxy.ts
import { createProxy } from 'vpndetection-next';

export const proxy = createProxy({ apiKey: process.env.VPNDETECTION_API_KEY });
export const config = { matcher: ['/signup'] };
```

```ts
// in the sign-up server action
import { headers } from 'next/headers';
import { readLookup } from 'vpndetection-next';

const lookup = readLookup(await headers());
// `result` is the answer as the API serves it, in snake_case.
if (lookup?.isVpn || lookup?.result?.is_tor || lookup?.result?.is_relay) {
    return { error: 'Please turn off your VPN or proxy to create an account.' };
}
```

## Express

`npm install vpndetection-express`, and put it on the sign-up route alone:

```js
import { vpndetection } from 'vpndetection-express';

app.set('trust proxy', 'loopback'); // the proxy in front of you, if any
app.post('/signup', vpndetection({
    apiKey: process.env.VPNDETECTION_API_KEY,
    blockCondition: [{ isVpn: true }, { isTor: true }, { isRelay: true }],
}), signup);
```

## Better Auth

`npm install vpndetection`, then a `before` hook on the sign-up endpoints:

```ts
import { betterAuth } from 'better-auth';
import { APIError, createAuthMiddleware } from 'better-auth/api';
import { VPNDetection } from 'vpndetection';

const vpndetection = new VPNDetection({ apiKey: process.env.VPNDETECTION_API_KEY, timeoutMs: 2000 });

export const auth = betterAuth({
    hooks: {
        before: createAuthMiddleware(async (ctx) => {
            if (!ctx.path.startsWith('/sign-up')) return;
            const ip = ctx.headers?.get('cf-connecting-ip') ?? ctx.headers?.get('x-real-ip');
            if (!ip) return;
            const result = await vpndetection.lookup(ip).catch(() => null);
            if (result?.isVpn || result?.isTor || result?.isRelay) {
                throw new APIError('FORBIDDEN', { message: 'Please turn off your VPN or proxy to create an account.' });
            }
        }),
    },
});
```

Use the header the app's own edge sets: `cf-connecting-ip` on Cloudflare, `x-real-ip` from an nginx you run.

## Django

`pip install vpndetection-django`, add `"vpndetection_django.VPNDetectionMiddleware"` to `MIDDLEWARE`, skip every path but the sign-up's, and decorate the sign-up view:

```python
# settings.py
VPNDETECTION = {
    "api_key": os.environ.get("VPNDETECTION_API_KEY"),
    "skip": lambda request: not request.path.startswith("/signup"),
}
```

```python
from vpndetection_django import block_if


@block_if([{"is_vpn": True}, {"is_tor": True}, {"is_relay": True}])
def signup(request): ...
```

## Laravel

`composer require vpndetection/laravel`, `php artisan vendor:publish --tag=vpndetection-config`, set `VPNDETECTION_API_KEY` in `.env`, and in `config/vpndetection.php`:

```php
'block_condition' => [['isVpn' => true], ['isTor' => true], ['isRelay' => true]],
```

Then put the middleware on the registration route alone, not in `bootstrap/app.php`, where it would apply the condition to every page:

```php
Route::post('register', [RegisteredUserController::class, 'store'])
    ->middleware(\VPNDetection\Laravel\VPNDetectionMiddleware::class);
```

## Rails

`bundle add vpndetection-rails`, configure it with no `block_condition` and a `skip` for every path but the sign-up's, and refuse in the registrations controller:

```ruby
# config/application.rb
config.vpndetection = {
  api_key: ENV["VPNDETECTION_API_KEY"],
  skip: ->(request) { !request.path.start_with?("/signup") },
}
```

```ruby
before_action :refuse_vpn, only: :create

def refuse_vpn
  result = VPNDetection::Rails.lookup(request)&.result
  return unless result&.vpn? || result&.tor? || result&.relay?

  redirect_back fallback_location: root_path, alert: "Please turn off your VPN or proxy to create an account."
end
```

## What it doesn't do

It judges the address, not the person: a VPN user may be honest, and a fraudster on a home connection passes. It doesn't stop disposable email addresses or bots that sign up from clean addresses.
