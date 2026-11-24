---
name: ghost-ctf
description: >-
  Repeatable method for working Linux capture-the-flag / wargame / boot2root
  boxes (HackTheBox, TryHackMe, OverTheWire/Bandit, VulnHub, OSCP-style labs).
  Use when you have a foothold — a shell, a file, a blob, a service — and need
  the next step: decode layered data, find a leaked token or credential, talk to
  a local service, escalate privileges (SUID / sudo / cron / caps), or pivot to
  the next user. Authorized practice and lab targets only.
---

# Ghost — Linux CTF method

A method for solving Linux CTF / wargame / boot2root challenges. It is scoped to
**authorized practice targets**: CTF platforms, wargames, lab VMs, and boxes you
own or are explicitly permitted to attack. It is enumeration-and-escalation
methodology, not exploit development.

## The one idea underneath all of it

A privileged program, process, or file holds a power you don't have. **Find it,
then use it to borrow that power.** `find` / `getcap` / `sudo -l` locate it; the
helper, service, or binary does the work. Everything below is a way of applying
this one move at each level.

## The arc

```
recon → decode → find leaked secrets → talk to services → escalate → pivot
```

Repeated up the chain. Each level's loot (a token, a key, a password, a file) is
the input to the next level. When you get stuck, you are almost always missing a
recon step on the level you're already standing on.

## The durable method (follow this when a box feels stuck)

1. **Look before you transform.** Inspect raw bytes before you decode, run, or
   convert anything.
2. **Evidence picks the tool — magic bytes, not extension.** `file` it; a
   `.txt` may be gzip, a `.bin` may be a tarball.
3. **Peel and re-check each layer.** Decode one layer → `file` the result →
   repeat until plaintext. Never assume how deep it goes.
4. **Blocked? Drop a level.** Tool missing or filtered → use the primitive
   underneath it (`ss`/`netstat` → `/proc/net/tcp`; `nc` → `/dev/tcp`).
5. **The clue's wording names the technique.** A key comment, a file name, a
   brief, a hint — read it literally; it usually tells you the next move.
6. **Follow the chain.** Don't jump ahead. Finish looting a level before
   climbing; the thing that unlocks the next step is usually right where you are.

## Playbook

### 1. Recon — always first

```bash
ls -la              # everything, including hidden .files
file X              # what it really is (ignore the extension)
strings X           # readable text inside a binary or blob
xxd X | head        # raw bytes / magic number before transforming
```

### 2. Decode / peel layers

```bash
base64 -d          # "encoded for transport" → decode
xxd -r             # a hexdump (lines start 00000000:) back to bytes
xz -d / gunzip     # compressed → decompress, then file the result
tar tf stage.bin   # peek inside an archive BEFORE extracting (preview + safety)
```

Rule: **decode one layer → `file` it → repeat** until you hit plaintext.

### 3. Find secrets — where they hide

```bash
strings core.dump | grep -i token    # core dumps leak RAM contents
env                                   # env vars (then base64 -d anything encoded)
cat /proc/<pid>/cmdline /proc/<pid>/environ   # live process secrets
cat ~/.bash_history                   # past commands, often with creds
cat ~/.ssh/* ~/.ssh/known_hosts       # keys + hosts that were reached
git log -p --all                      # committed-then-deleted secrets
sort file | uniq -u                   # the one odd line in a sea of duplicates
```

Also worth a look: `.env`, `config.php`, `*.conf`, database configs.

### 4. Talk to local services

```bash
# find listeners when ss/netstat are blocked (ports are hex)
cat /proc/net/tcp
printf '%d\n' 0xA15E                  # decode a hex port

# scan loopback for open ports
nc -zv 127.0.0.1 1-65535 2>&1 | grep -i succ

# then talk to the port — read what it wants, reply
nc host port
echo "TOKEN" | nc host port

# no nc? DIY TCP client with bash
exec 3<>/dev/tcp/host/port; cat <&3

# TLS port
openssl s_client -connect host:port -quiet
```

### 5. SSH / pivot

```bash
ssh -i key user@host 'cmd'            # run a remote command non-interactively
chmod 600 key                         # or ssh refuses the key
```

The **key comment tells you where it was made**; the note/brief that came with
it tells you the real target.

### 6. Privilege escalation

```bash
find / -perm -4000 -type f 2>/dev/null   # SUID — runs as its owner
getcap -r / 2>/dev/null                  # capabilities (modern SUID)
sudo -l                                  # what you may run as root
uname -a                                 # kernel version → look up known exploits
cat /etc/crontab; ls -la /etc/cron.d/    # root running things on a timer
ps aux                                   # what's running, and as whom
```

- A **SUID binary** can be run to read root-only files (e.g. a `ghost-archivist`
  helper).
- **Cron race:** root creates a file → sleeps → deletes it. Win the window:
  ```bash
  while true; do cat FILE 2>/dev/null && break; done
  ```

## Enumeration checklist (paste-and-go)

```bash
# Who am I / what can I reach
id; sudo -l

# Files
ls -la ~; ls -la /
find / -writable -type d 2>/dev/null       # writable dirs = injection spots

# Programs that run as someone else
find / -perm -4000 -type f 2>/dev/null      # SUID
getcap -r / 2>/dev/null                      # capabilities

# Scheduled / automated privilege
cat /etc/crontab; ls -la /etc/cron.d/
ps aux

# Processes / leaked secrets
cat /proc/<pid>/cmdline /proc/<pid>/environ

# Network
ss -tlnp 2>/dev/null || cat /proc/net/tcp    # listeners
# then: nc host port  /  openssl s_client -connect host:port -quiet

# Config / history
cat ~/.bash_history ~/.ssh/* ~/.ssh/known_hosts
git log -p --all
```

## Graduation pattern

Multi-flag finales tend to be: several shards, **one technique each**
(e.g. `strings` / `base64 -d` / a SUID helper) — gathered, assembled, and sent
to a gatekeeper over `nc`. When you see "collect N pieces," expect one distinct
move per piece and one final submission step.

## Teaching mode

If the user wants to **learn** rather than solve a specific box — "explain the
fundamentals", "where do I look", "walk me through the commands", "I'm new to
this" — read `references_fundementals.md` and teach from it: concept first, then
where to look, then the command, then why it works. Don't just dump the cheat
sheet; explain the *why* behind each move so they can improvise when a tool is
missing.
