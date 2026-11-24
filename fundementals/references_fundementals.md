# Ghost — Fundamentals

The learning companion to the Ghost method. This is the thing to read (or teach
from) when you want to *understand* the moves, not just run them. Scope is the
same: **authorized practice targets** — CTF platforms, wargames, lab VMs, boxes
you own or are allowed to attack.

Teach it in this order for any technique: **concept → where to look → command →
why it works.** If you understand the "why," you can rebuild the move when the
usual tool is missing.

---

## The mental model

Every level of a box is the same puzzle:

> Something nearby has a power you don't have — it can read a file you can't,
> run as a user you aren't, or hand out a secret if you ask correctly. Your job
> is to **find that thing and borrow its power.**

Solving a box is doing this once per level and feeding each level's loot into the
next. "Being stuck" almost always means you haven't finished *looking* at the
level you're on.

Two habits make the whole thing work:

- **Look before you transform.** Inspect raw bytes before decoding or running
  anything. You avoid wrecking evidence and you avoid running something hostile.
- **Let evidence pick the tool.** The file extension is a suggestion, not a
  fact. `file` tells you what something actually is from its magic bytes.

---

## Phase 1 — Recon (what is this thing?)

**Concept:** before you act, identify. Most wasted time comes from guessing a
file's type instead of checking it.

| Look at | Command | Why |
|---|---|---|
| The directory, fully | `ls -la` | `-a` shows hidden `.files`, where challenges stash things |
| What a file really is | `file X` | reads magic bytes; a `.txt` can be gzip, a `.log` can be an ELF |
| Text inside a binary | `strings X` | pulls printable runs out of a blob — tokens, paths, hints |
| The raw head | `xxd X \| head` | shows the magic number and structure before you transform it |

**Why it works:** files announce themselves in their first few bytes (`PK` =
zip, `\x1f\x8b` = gzip, `7fELF` = executable, `fd 37 7a` = xz). `file` reads
those; you should learn to recognize the common ones by eye in `xxd`.

---

## Phase 2 — Decoding (peel the layers)

**Concept:** CTF data is often wrapped several times — base64 over gzip over a
hexdump, etc. You unwrap **one layer at a time** and re-identify after each.

| Clue | Command | Why |
|---|---|---|
| "encoded for transport", `A-Za-z0-9+/=` text | `base64 -d` | base64 is the default transport encoding |
| lines start `00000000:` | `xxd -r` | that's a hexdump; `-r` reverses it to bytes |
| `file` says gzip / xz | `gunzip` / `xz -d` | decompress, then re-check |
| `file` says tar | `tar tf f` then extract | `tf` lists contents first — preview, and don't blindly extract junk |

**The loop:** `decode → file → decode → file …` until you reach readable
plaintext. The number of layers is unknown; stop only when `file` says plain
text and it reads like the answer.

**Why it works:** each encoding has a signature in the decoded output, so you're
never guessing — you decode, then let `file`/`xxd` tell you what you got.

---

## Phase 3 — Finding secrets (where they hide)

**Concept:** secrets leak into predictable places — memory, environment,
history, config, version control. Learn the list and sweep it every time.

| Place | Command | Why it leaks |
|---|---|---|
| Core dumps | `strings core.dump \| grep -i token` | a crash dumps live RAM to disk |
| Environment | `env` | apps pass secrets as env vars (decode base64 ones) |
| Live processes | `cat /proc/<pid>/cmdline /proc/<pid>/environ` | args + env of a running process |
| Shell history | `cat ~/.bash_history` | people type passwords on the command line |
| SSH | `cat ~/.ssh/* ~/.ssh/known_hosts` | private keys + a map of hosts reached |
| Git | `git log -p --all` | a secret "deleted" in a later commit still sits in history |
| Odd-one-out | `sort f \| uniq -u` | `sort` groups dupes; `uniq -u` prints the single unique line |

Also sweep: `.env`, `config.php`, `*.conf`, database config files.

**Why it works:** these are side effects of normal operation — nobody *means* to
leave a token in a core dump or an env var, which is exactly why it's there.

---

## Phase 4 — Talking to local services

**Concept:** a box often runs a service on loopback that hands out the next
secret if you connect and say the right thing. First find the port, then speak
to it.

| Need | Command | Why |
|---|---|---|
| Listeners (tools blocked) | `cat /proc/net/tcp` | the kernel's raw table; ports are **hex** |
| Decode a hex port | `printf '%d\n' 0xA15E` | `0xA15E` → `41310` |
| Scan loopback | `nc -zv 127.0.0.1 1-65535 2>&1 \| grep -i succ` | `-z` = scan only, filter to successes |
| Talk to it | `nc host port` / `echo "TOKEN" \| nc host port` | read the prompt, then reply |
| No `nc` | `exec 3<>/dev/tcp/host/port; cat <&3` | bash's built-in TCP — the fallback |
| TLS port | `openssl s_client -connect host:port -quiet` | same idea, wrapped in TLS |

A probe loop when you have a handful of candidate ports:

```bash
for p in 30003 30100 30101 31339 41310; do
  echo "=== $p ==="
  timeout 3 nc 127.0.0.1 $p </dev/null
done
```

**Why it works:** `/dev/tcp` is a bash feature, so you get a TCP client even on a
stripped box with no `nc`, `netcat`, or `ncat`. This is the "drop a level"
principle in action.

---

## Phase 5 — SSH and pivoting

**Concept:** credentials/keys you find are for moving to the next user or host.

| Command | Why |
|---|---|
| `ssh -i key user@host 'cmd'` | run one command remotely without an interactive shell |
| `chmod 600 key` | ssh refuses a key that's group/world-readable |

**Read the metadata.** A key's trailing **comment** says where it was generated
(`user@origin-host`), which tells you the account and box it belongs to. The
brief or note that came with it tells you the real target.

---

## Phase 6 — Privilege escalation

**Concept:** find a program, process, or file that already runs with more
privilege than you, and make it act on your behalf.

| Vector | Command | Why it works |
|---|---|---|
| SUID | `find / -perm -4000 -type f 2>/dev/null` | SUID bit = runs as the file's **owner** (often root) |
| Capabilities | `getcap -r / 2>/dev/null` | fine-grained "modern SUID" (e.g. `cap_setuid`) |
| Sudo rights | `sudo -l` | lists exactly what you may run as root — read it literally |
| Kernel | `uname -a` | match the version to a known local-exploit |
| Cron | `cat /etc/crontab; ls -la /etc/cron.d/` | root running things on a timer = a window to abuse |
| Processes | `ps aux` | what's running and as whom |
| Writable dirs | `find / -writable -type d 2>/dev/null` | places to drop a file something privileged will pick up |

**SUID in practice:** a custom SUID helper (e.g. `ghost-archivist`) exists to do
one privileged thing — run it and it reads a root-only file *for* you.

**Cron race:** root creates a file, sleeps, then deletes it. You win the gap:

```bash
while true; do cat FILE 2>/dev/null && break; done
```

**Why it works:** the privilege is already granted to the binary/job; you're not
breaking in, you're standing in the path of power that's already flowing.

---

## Graduation: assembling a multi-flag finale

The capstone is usually: **N shards, one technique each** — say one found with
`strings`, one behind `base64 -d`, one read via a SUID helper — then assembled
and submitted to a gatekeeper over `nc`. When a challenge says "collect the
pieces," read it as: expect one distinct move per piece, and one final delivery
step.

---

## One-screen recap

1. **Look before you transform** — `file`, `xxd`, `strings` first.
2. **Magic bytes, not extension** — the file tells you what it is.
3. **Peel + re-check every layer** — decode → `file` → repeat.
4. **Blocked? Drop a level** — `/proc/net/tcp`, `/dev/tcp`, raw primitives.
5. **The clue names the technique** — read key comments, file names, briefs.
6. **Follow the chain** — finish looting a level before you climb.

Underneath all six: *something nearby has a power you don't — find it and borrow
it.*
