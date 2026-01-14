---
title: I am thinking about salad
date: 2025-09-12
description: salad thoughts
summary: idk salad stuff
tags:
  - testg
  - welcome
  - salad
---
love that vibe. let’s lean into “gloriously over-engineered for a very small problem.” here’s a revamped outline + punchier copy you can drop in.

  

# **Title ideas (with a wink)**

- “Command-Line Aquatics: Massively Over-Engineered Lane Booking for One Guy”
    
- “Swim Fast, Click Slow: Why I Built a CLI to Beat a Website”
    
- “Chelsea Piers, But Make It Bash: A Needlessly Robust Lane-Booking Tool”
    

  

# **Tone rules (for the whole post)**

- Self-aware, a little cheeky, never mean.
    
- Sprinkle faux-serious engineering phrases for tiny wins (“five nines of… reservations?”).
    
- Use footnotes for jokes and humblebrags.
    
- Short paras, lots of headings; screenshots or terminal GIFs where it helps.
    

---

# **Outline (tongue-in-cheek edition)**

1. Cold open: the 5:59 a.m. Hunger Games
    
    - Paint the scene: caffeine, countdown, spinning throbber.
        
    - “I wrote a CLI so I could stop losing to a button.”
        
    
2. Scope, allegedly
    
    - What a “lane assignment” is, in human terms.
        
    - Constraints that sound important but mostly justify the project (release times, caps).
        
    - Failure modes: page timeouts, fat-fingered clicks, plot twists from CSRF.
        
    
3. Goals & non-goals (a.k.a. “what this is not… yet”)
    
    - Goals: faster than clicking, logs that tattle on me, dry-run to protect my dignity.
        
    - Non-goals: violating ToS, DOSing the gym, “ML for lanes.”*
        
    
4. Architecture (dramatically oversized)
    
    - auth/, slots/, book/, notify/, schedule/, observability/—yes really.
        
    - Sequence diagram for a three-step operation because diagrams are fun.
        
    - Idempotency locks so I don’t book Lane 4… four times.
        
    
5. Reverse-engineering (ethically, caffeinated)
    
    - DevTools sleuthing: endpoints, headers, CSRF crumbs.
        
    - Rate limits, polite backoff, jitter (because I’m not a monster).
        
    - How to break nothing while learning everything.
        
    
6. The booking “race strategy”
    
    - Pre-warm session 60–90s before drop.
        
    - NTP drift paranoia and why my clock is more accurate than my swim splits.
        
    - Retry story: 409/429/5xx and when to back away slowly.
        
    
7. Implementation highlights (a tour of the toys)
    
    - Python + typer, httpx, tenacity, pydantic, rich.
        
    - Config: .env for mortals, Keychain for grown-ups.
        
    - CLI UX: lanes slots, lanes book, --dry-run, --today, --prefs weekday-early.
        
    
8. Scheduling & automation (because I will forget)
    
    - Cron/launchd/systemd one-liners.
        
    - Presets: “Mon/Wed/Fri 6:30 a.m.” with vacation kill-switch.
        
    - What gets logged vs what gets pinged.
        
    
9. Notifications that spark just enough joy
    
    - Slack/webhooks, macOS notifications, email.
        
    - Success, partial success, and the walk of shame (failure with reasons).
        
    
10. Testing without angering the front desk
    

  

- Mock server, recorded fixtures (VCR-style), schema drift checks.
    
- “Simulated drop” so I can rehearse like it’s a Broadway show.
    

  

11. Observability (for a hobby project, lol)
    

  

- JSON logs, trace IDs, tiny metrics (p95 book time, success rate).
    
- Health checks: “can we auth?” “is CSRF fresh?” “do we even have internet?”
    

  

12. Security & ethics (the serious bit)
    

  

- Secret storage, no plaintext creds in logs.
    
- Respectful rate limits and ToS, personal-use guardrails.
    

  

13. Results & war stories
    

  

- Before/after: success %, average book time, how often it saves my morning.
    
- Funny edge cases: “booked the sauna by accident” (didn’t, but you get it).
    

  

14. How to install & use (so you can copy me)
    

  

- uv tool install lanes-cli (or pipx).
    
- Quickstart commands, uninstall/cleanup, sample config.
    

  

15. What’s next (things I’ll probably never do)
    

  

- Apple Watch glance, calendar sync, tiny web dashboard.
    
- “Family plan” for couples diplomacy.
    

  

16. Appendix
    

  

- Example configs, cron snippets, sequence diagram, terminal GIFs.
    
- Disclaimer: not affiliated with Chelsea Piers; affiliate with good vibes.
    

  

* yet.

---

## **Drop-in copy you can use (first few sections)**

  

### **Cold open: the 5:59 a.m. Hunger Games**

  

At 5:59 a.m., I become a professional button-clicker. Chelsea Piers drops new swim lanes, the site slows to a dignified crawl, and my mouse suddenly develops stage fright. After losing one too many morning slots to the spinning wheel of fate, I did the only rational thing: I built a command-line tool so I could out-click a website.

  

### **Scope, allegedly**

  

A “lane assignment” is 45 minutes of chlorinated bliss, doled out on a predictable schedule with some fairness rules so nobody hogs the water. The UI is fine for humans at noon; it’s less fine at 6:00:00 a.m. when 200 of us try to be the same human. My needs were simple: log in, find slots that match my preferences, book at drop, and tell me if it worked. My solution was… not simple. But it is delightful.

  

### **Goals & non-goals**

  

**Goals:** be faster than clicking, keep receipts (logs), and include a dry-run so I can practice without actually stealing my own lane.

**Non-goals:** breaking rules, hammering endpoints, or introducing machine learning to the problem of “click button.”

---

## **Architecture (dramatically oversized)**

  

The project has more folders than the problem deserves:

- auth/ grabs a session and handles CSRF like it’s priceless.
    
- slots/ lists candidate times; book/ commits with an idempotency key so I don’t double-dunk.
    
- notify/ pings Slack/email/macOS; schedule/ wires cron/launchd; observability/ pretends this isn’t a hobby.
    

  

Data flow (because we love a diagram): **Login → Fetch → Filter → Book → Confirm → Notify**.

There’s a tiny lock file so simultaneous runs don’t book me four lanes and a lifeguard chair.

---

## **Reverse-engineering (ethically, caffeinated)**

  

Step one: open DevTools like a responsible adult. Watch the network tab while booking in the browser. Copy nothing blindly; observe everything carefully. You’ll meet CSRF tokens, cookies, and a modest set of JSON endpoints. Add polite rate limits and randomized jitter so you act like a person who respects other people’s mornings.

---

## **CLI usage (fun but real)**

```
# one-time login (stores token in Keychain)
lanes auth login

# preview slots for tomorrow between 6–8am (no booking)
lanes slots --date +1 --start 06:00 --end 08:00 --pool 50m --dry-run

# book the earliest 45-min slot at/after 6:30am, then notify me
lanes book --date +1 --earliest 06:30 --duration 45 --pool 50m --notify

# schedule on weekdays at 5:59:55 (yes, the five seconds matter)
# crontab -e
59 5 * * 1-5 lanes book --today --prefs weekday-early --notify
```

---

## **Installation (short + friendly)**

- If you use **uv**:
    

```
uv tool install lanes-cli
```

-   
    
- Or with **pipx**:
    

```
pipx install lanes-cli
```

-   
    
- Minimal config:
    

```
# ~/.config/lanes/config.toml
pool = "50m"
earliest = "06:30"
duration = 45
notify = "slack:webhook_url_goes_here"
```

-   
    
- Uninstall:
    

```
uv tool uninstall lanes-cli   # or: pipx uninstall lanes-cli
```

  

---

## **Callout boxes you can sprinkle in**

  

> **Isn’t this overkill?**

> Absolutely. It’s also faster, calmer, and weirdly fun.

  

> **What if the site changes?**

> The tool fails loudly with a link to a “schema drift” checklist. I sigh, fix a selector, and pretend I’m on SRE rotation.

  

> **Will I get banned?**

> I’m polite: low request counts, backoff, no scraping beyond what I’d click myself. If in doubt, I use dry-run and go back to being a normal person.

---

If you want, I’ll draft the next chunk (“Booking strategy,” “Testing without angering the front desk,” and “Observability, but tiny”) in this same tone and give you a final pass section with numbers you can collect from your logs (success rate, p95 book time, etc.) to make it feel legit.