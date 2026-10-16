# x-player-card

A Claude Code skill that puts a live mini website inside a tweet: a signup form, search box, calculator or game that people use without leaving the X timeline.

It uses X's player card (`twitter:card=player`). The skill covers the meta tags, the `frame-ancestors` header, building for X's sandboxed iframe, the poster image, mobile, bot firewalls that block Twitterbot, and a check list to run before you post.

## Install

```bash
git clone https://github.com/rdoyle99/x-player-card ~/.claude/skills/x-player-card
```

Then in Claude Code: "put a newsletter signup inside a tweet for my site".

## Live examples

- https://www.rpdoyle.com/x (email signup, Next.js)
- https://www.tmaker.io/subscribe (the card that started it, by @tibo_maker)

Made by [@doooyle](https://x.com/doooyle). Credit to @tibo_maker and @tarasshyn for the trick.
