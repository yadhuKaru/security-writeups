# security-writeup

Hey — I'm Yadhu (aka Homie Hacks), and this is where I document the security rooms and boxes I work through.

I'm heading into cybersecurity as an incoming freshman focused on offensive security, and I built this repo with  **beginners** in mind.  
I want to show how someone actually *thinks* through these boxes, not just copy-paste commands. 
Every writeup here follows the same rule: **show the "why," not just the "what."** 


---

## How These Are Written

- **Beginner-friendly by default.** Jargon gets defined inline or tucked into collapsible sections, so this reads fine whether you already know the tools or you're seeing them for the first time.
- **Reasoning over recipes.** Every step explains *why* I tried something, not just what I typed.
- **No walls of raw output.** Screenshots and short snippets stand in for pages of terminal logs.
- **Fixes included.** Every vulnerability comes with an actual remediation section — what a developer or admin would do to close the hole.

---

## Write-Ups

| # | Room | Platform | What's Inside |

| 1 | [Whiterose](./whiterose-writeup.md) | TryHackMe | Chaining weak creds → an IDOR → a leaked password → an SSTI/RCE bug in EJS → a `sudo` privilege-escalation CVE, all the way to root |

*When new posts come out you can check back here, or watch the repo for updates.*

---

## A Bit More About Me

I'm currently a freshman in college studying Comp Sci and Eng.  with a focus on offensive security
This series doubles as both a learning log and a portfolio soo if you're a beginner, I hope these help something click.
If you're evaluating my work,  thanks for reading, and feel free to open an issue or reach out if you spot something I got wrong. 

---

*⭐ Star this repo if you find it useful — it helps other beginners find it too.*
