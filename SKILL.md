---
name: x-player-card
description: Put a live, clickable mini web page inside a tweet using X's player card (twitter:card=player). Use when someone wants a signup form, search box, calculator, game or demo that runs inside the X timeline, says "website inside a tweet", "player card", "tweet card embed", "interactive tweet", or wants to copy the tmaker.io /subscribe trick. Covers the meta tags, the frame-ancestors header, the embed page built for X's sandboxed iframe, the poster image, mobile layout, bot firewalls, and how to check it before posting.
---

# Website inside a tweet (X player card)

X still renders `twitter:card = player`: on desktop the tweet shows your poster image with a play button, and a click loads your page in a 640x480 iframe right in the timeline. On mobile, a tap opens your page in a panel that covers the bottom ~30%. People type, click and submit without leaving X.

Working references (checked 2026-10-16): `https://www.tmaker.io/subscribe` (Tibo's newsletter card) and `https://www.rpdoyle.com/x` (Next.js pages router, email capture). `curl -s https://www.tmaker.io/subscribe | grep twitter:` shows the full tag set.

## The 6 parts

### 1. One public URL with two modes

- `https://www.example.com/x` is the URL you tweet. Normal visitors get a full page.
- `https://www.example.com/x?embed=1` is the `twitter:player` URL. It renders the compact 640x480 card only: no nav, no footer, no cookie banner, no chat widget, no analytics popups.
- Same route, same code, one query flag. Fewer moving parts than a second route.

### 2. Meta tags on the tweeted URL (server-rendered, in the initial HTML)

```html
<meta name="twitter:card" content="player">
<meta name="twitter:site" content="@yourhandle">
<meta name="twitter:title" content="What the card does, in plain words">
<meta name="twitter:description" content="One sentence">
<meta name="twitter:image" content="https://www.example.com/x-poster.png">
<meta name="twitter:player" content="https://www.example.com/x?embed=1">
<meta name="twitter:player:width" content="640">
<meta name="twitter:player:height" content="480">
<meta property="og:title" content="...">
<meta property="og:description" content="...">
<meta property="og:image" content="https://www.example.com/x-poster.png">
<meta property="og:url" content="https://www.example.com/x">
```

- Absolute HTTPS URLs everywhere. Tags must be in the HTML X's crawler gets, never injected by client JS.
- Poster: 1200x900 PNG/JPG (4:3, same ratio as 640x480), under 5MB. It is what people see before they click and the whole card on clients that don't play it. Make it a screenshot of the real card so the click feels like nothing changed.

### 3. Let X frame the page

Send on the `/x` route (both modes):

```
Content-Security-Policy: frame-ancestors 'self' https://x.com https://*.x.com https://twitter.com https://*.twitter.com
```

and remove `X-Frame-Options` from that route (it overrides nothing useful and some browsers honor it first). Keep your stricter headers on every other route.

### 4. Post the exact final URL

`https://`, the canonical host (www or apex, whichever does not redirect), no trailing redirects. `curl -sI <url>` must answer `200` directly. A redirect anywhere and X falls back to a plain link.

### 5. Build for the sandbox

X loads the player in a sandboxed iframe:

- Forms don't navigate. Submit with `fetch()` and render the result in place (`e.preventDefault()`).
- No login, no OAuth popups, no third-party cookies (they're blocked in the iframe).
- Every link opens a new tab: `target="_blank" rel="noopener"`. Never `window.top.location`.
- No `alert()`/`confirm()`. Errors show inline.
- The API the card calls must accept the request from the iframe (same origin is easiest; if cross-origin, send CORS headers).
- Rate-limit and validate the endpoint: a viral tweet means bots too.

### 6. Design for two screens

- Desktop: exactly 640x480, `html,body{margin:0;height:100%;overflow:hidden}`. Nothing scrolls. Test it in a 640x480 iframe.
- Mobile: the panel covers the bottom ~30%, so put the input and button in the top 60% of the page.
- Inputs at 16px+ (iOS zoom), big tap targets, a visible result in the same spot as the form.
- The fun part of Tibo's card: a bottom row "or support this tweet · 1 tap each" with a green "repost" and a pink "like" label, each with a hand-drawn arrow pointing down at X's own repost and like buttons under the card. No links needed. Measured from Tibo's tweet on desktop: X's repost button sits about 27% across the card (≈175px of 640) and like about 50% (≈320px).

## Bot firewalls

Vercel BotID, Cloudflare Bot Fight Mode, Attack Mode and custom WAF rules often block `Twitterbot`. Then X gets a 403/429 and shows no card. Allow it:

- Allowlist user agent `Twitterbot` on `/x` and the poster image.
- Check: `curl -s -A "Twitterbot/1.0" https://www.example.com/x | grep -c 'twitter:player'` prints 3 (player, width, height). Run it from a machine your firewall hasn't flagged.

## Framework snippets

Next.js App Router (`app/x/page.tsx`):

```ts
export const metadata = {
  title: "...",
  openGraph: { title: "...", images: [{ url: "https://www.example.com/x-poster.png", width: 1200, height: 900 }] },
  twitter: {
    card: "player",
    site: "@yourhandle",
    images: ["https://www.example.com/x-poster.png"],
    players: [{ playerUrl: "https://www.example.com/x?embed=1", streamUrl: "https://www.example.com/x?embed=1", width: 640, height: 480 }],
  },
};
```

Header in `next.config.js`:

```js
async headers() {
  return [{ source: "/x", headers: [{ key: "Content-Security-Policy", value: "frame-ancestors 'self' https://x.com https://*.x.com https://twitter.com https://*.twitter.com" }] }];
}
```

Next.js Pages Router: put the meta tags in `<Head>` in `pages/x.js`, read `embed` in `getServerSideProps` (or `router.query`), same `headers()` block.

Express:

```js
app.get("/x", (req, res) => {
  res.removeHeader("X-Frame-Options");
  res.set("Content-Security-Policy", "frame-ancestors 'self' https://x.com https://*.x.com https://twitter.com https://*.twitter.com");
  res.sendFile(path.join(__dirname, "public/x.html"));
});
```

If you use helmet, its `frameguard` sets `X-Frame-Options: SAMEORIGIN` on every route: turn it off for `/x`.

## Check before you tweet

1. `curl -sI https://www.example.com/x` → `200`, has `frame-ancestors ... https://x.com`, no `x-frame-options`.
2. `curl -s -A "Twitterbot/1.0" https://www.example.com/x | grep -E 'twitter:(card|player|image)'` → card=player, player URL, 640, 480, image.
3. `curl -sI https://www.example.com/x-poster.png` → `200`, `image/png`.
4. Load `?embed=1` in a 640x480 iframe on a scratch page: nothing scrolls, the form works, links open new tabs.
5. Load it at 375px wide: the form sits in the top 60%.
6. Post the tweet with the URL. X caches cards for about a week, so fix tags before posting, or add `?v=2` to force a fresh fetch.

## Gotchas

- X caches the first crawl. A broken first tweet stays broken for that URL: change the query string.
- Card shows as a plain link: the URL redirected, the crawler got blocked, or the tags were client-rendered.
- Blank iframe on click: `X-Frame-Options` still set, or `frame-ancestors` missing `https://x.com`.
- Form "does nothing": it tried a normal submit inside the sandbox. Use `fetch`.
- The X app on phones may open the page in its in-app browser rather than inline. That's fine if the top of the page carries the action.
