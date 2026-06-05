# Chapter 1.1 — SSH Port Forwarding and Connecting to RDHPCS (Hera)

> *How a single YubiKey tap in one terminal can quietly let an editor like
> Kiro reach a high-performance computing cluster sitting behind a security
> perimeter — without ever asking you for a password again.*

This chapter starts from zero. If you have never thought hard about what a
"port" is, or why anyone would SSH to their own laptop, that is exactly the
right starting point. By the end you should be able to read an
`~/.ssh/config` file like the one used to reach NOAA's Hera cluster and
explain, line by line, what each piece is doing.

---

## 1. What is a "port" anyway?

Forget computers for a moment. Picture an apartment building.

```
   ┌──────────────────────────┐
   │      Apartment Block     │   <- one computer (one IP address)
   │                          │
   │  Apt 22  Apt 80  Apt 443 │   <- ports
   │   SSH    Web    HTTPS    │      (different services at different doors)
   └──────────────────────────┘
            │
       Street address              <- the IP address
       (e.g. 10.0.0.5)
```

- The **street address** (IP address) tells the mail carrier which building.
- The **apartment number** (port number) tells them which door inside.

A computer can run many programs at once — a web server, an SSH server, a
database — and each one listens at its own apartment number. A few you will
see over and over:

| Port | Service        |
|-----:|----------------|
|   22 | SSH (login)    |
|   80 | HTTP (web)     |
|  443 | HTTPS (secure web) |

When you type `ssh somehost`, your computer is really saying *"deliver this
to `somehost`, apartment 22."*

---

## 2. What is `localhost`?

`localhost` is a special name that always means **"this very computer I am
sitting at."** Same machine, no network involved.

```
   Your laptop
   ┌──────────────────────────────────┐
   │                                  │
   │   localhost  =  "me, myself"     │
   │                                  │
   │   localhost:8080  =  apt 8080    │
   │                      inside me   │
   │                                  │
   └──────────────────────────────────┘
```

So `localhost:50000` means *"go knock on apartment 50000 inside my own
laptop."* Normally that apartment would be empty. The clever part of port
forwarding is that we put a **secret tube** in that apartment that comes out
somewhere else entirely.

---

## 3. The problem we are solving

Reaching Hera involves three buildings, not two.

```
   Your laptop (GFE)        RDHPCS Bastion             Hera
   ┌───────────┐            ┌───────────┐         ┌───────────┐
   │           │            │  YubiKey  │         │           │
   │           │            │  required │         │           │
   │           │ ────────►  │  at door  │ ──────► │           │
   │           │            │           │         │           │
   └───────────┘            └───────────┘         └───────────┘
```

The bastion (`hera-rsa.princeton.rdhpcs.noaa.gov`) is a guarded gate.
Anything that wants to reach Hera has to pass through there, and the gate
demands a YubiKey tap.

That is fine for a human in a terminal. But Kiro is a program — it cannot
tap a YubiKey for you. We need a way to authenticate **once** with the
YubiKey and then leave a hands-free path that any tool can reuse.

That hands-free path is the **SSH tunnel**.

---

## 4. What `LocalForward` actually does

The magic line in the SSH config is:

```
LocalForward MY_FORWARD_PORT localhost:MY_FORWARD_PORT
```

In English: *"Open apartment number `MY_FORWARD_PORT` on my laptop. Anything
that knocks on that door gets transported through the SSH tunnel to the same
apartment number on the far side, which RDHPCS has already wired to land on
Hera."*

Picture a pneumatic tube between two desks:

```
   YOUR LAPTOP                                   RDHPCS / HERA
   ┌──────────────────────┐                  ┌──────────────────────┐
   │                      │                  │                      │
   │  Apt MY_FORWARD_PORT │                  │  Apt MY_FORWARD_PORT │
   │  ┌────────────────┐  │   SSH tunnel     │  ┌────────────────┐  │
   │  │  secret tube   │  │ ════════════════►│  │  Hera SSH door │  │
   │  │                │  │  (encrypted)     │  │   (port 22)    │  │
   │  └────────────────┘  │                  │  └────────────────┘  │
   │                      │                  │                      │
   └──────────────────────┘                  └──────────────────────┘
            ▲                                            │
            │  knock here on localhost                   ▼
            │  =                                    you arrive on Hera
            │  knock on Hera
```

Two facts to lock in:

1. The tube **only exists while the SSH session that built it is alive**.
   Close that terminal, the tube vanishes.
2. From your laptop's point of view, the tube looks like a normal apartment
   door at `localhost:MY_FORWARD_PORT`. Any program — `ssh`, Kiro, a
   browser — can walk up to that door without knowing or caring there is a
   tunnel behind it.

---

## 5. The two `Host` blocks, side by side

The `~/.ssh/config` file you set up has two named shortcuts. They look
similar but do very different jobs.

### 5.1 `Host Hera` — the tunnel BUILDER

```
Host Hera
    HostName hera-rsa.princeton.rdhpcs.noaa.gov   ← the real bastion address
    User My.Hera.Username
    IdentityFile ~/.ssh/id_rsa
    LocalForward MY_FORWARD_PORT localhost:MY_FORWARD_PORT   ← builds the tube
    ServerAliveInterval 60        ← send a heartbeat every 60s
    ServerAliveCountMax 30        ← give up only after 30 missed heartbeats
```

What happens, step by step, when you run `ssh -F ~/.ssh/config Hera`:

```
   Step 1                      Step 2                     Step 3
   ─────────                   ─────────                  ─────────
   You: ssh Hera        ───►   Bastion: "YubiKey?"  ───►  Tunnel is open.
   Laptop dials the            You tap it.                Apt MY_FORWARD_PORT
   bastion.                    Bastion: "Welcome."        on your laptop now
                                                          secretly leads to Hera.
```

The `ServerAlive*` lines are why the tunnel does not quietly die after a few
minutes of inactivity. Your laptop pings the bastion every 60 seconds saying
*"still here?"* If 30 pings in a row get no answer, only then does it give
up. That is roughly thirty minutes of grace, which absorbs normal network
hiccups.

You leave this terminal window open. It is **not** idle — it is the live
wire that holds the tunnel up.

### 5.2 `Host Hera-port` — the tunnel RIDER

```
Host Hera-port
    HostName localhost            ← yes, your own laptop
    Port MY_FORWARDED_PORT        ← the magic apartment
    User My.Hera.Username
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 30
```

When you run `ssh -F ~/.ssh/config Hera-port`:

```
   ssh asks: "Where to?"
       │
       ▼
   localhost, port MY_FORWARDED_PORT
       │
       ▼
   ┌──────────────────────────────────┐
   │  Apartment MY_FORWARDED_PORT     │
   │  on your laptop                  │
   │                                  │
   │  Inside it: the secret tube      │   ← built by 5.1 a moment ago
   │  going to Hera                   │
   └──────────────────────────────────┘
       │
       ▼
   You pop out on Hera, get asked
   for your SSH key (id_rsa), and
   because id_rsa.pub is in Hera's
   authorized_keys, you are let in
   instantly. No YubiKey.
```

This is the part that trips beginners up: you literally tell SSH to
"connect to localhost," which sounds like nonsense — why would you SSH to
your own laptop? The answer is that thanks to the tunnel, that specific
apartment on your laptop **is Hera in disguise**.

---

## 6. The two terminals, the full picture

```
   ┌──────────────────────────────  YOUR LAPTOP  ──────────────────────────────┐
   │                                                                           │
   │   Terminal #1                              Terminal #2 (or Kiro)          │
   │   ┌─────────────────────┐                  ┌─────────────────────┐        │
   │   │ ssh -F config Hera  │                  │ ssh -F config       │        │
   │   │                     │                  │     Hera-port       │        │
   │   │ [YubiKey tap]       │                  │                     │        │
   │   │ Connected.          │                  │ Connected instantly │        │
   │   │ (keep this open!)   │                  │ (no prompt)         │        │
   │   └──────────┬──────────┘                  └──────────┬──────────┘        │
   │              │                                        │                   │
   │              │ holds tunnel open                      │ rides tunnel      │
   │              ▼                                        ▼                   │
   │     ┌──────────────────────────────────────────────────────────┐          │
   │     │   Apartment MY_FORWARD_PORT on localhost                 │          │
   │     │   (the secret tube to Hera lives here)                   │          │
   │     └─────────────────────────┬────────────────────────────────┘          │
   │                               │                                           │
   └───────────────────────────────┼───────────────────────────────────────────┘
                                   │ encrypted SSH tunnel
                                   ▼
                           ┌──────────────────┐
                           │  RDHPCS bastion  │
                           │ (already passed) │
                           └────────┬─────────┘
                                    │
                                    ▼
                              ┌───────────┐
                              │   HERA    │
                              └───────────┘
```

Order of events:

1. You start Terminal #1 and run `ssh -F ~/.ssh/config Hera`. YubiKey tap.
   The tunnel is built. Leave it alone.
2. In Terminal #2 — or Kiro, or any tool — you connect to `Hera-port`. It
   hits `localhost:MY_FORWARDED_PORT`, which is really the tunnel, which
   delivers you to Hera. Your `id_rsa` private key on the laptop matches
   the `id_rsa.pub` you copied into Hera's `authorized_keys`, so Hera lets
   you in without a password.
3. Kiro can now open files on Hera, run commands, do AWS authentication,
   and so on. Every SSH call to `Hera-port` simply rides the existing
   tunnel.

---

## 7. Why each ingredient matters

| Ingredient | Without it | With it |
|---|---|---|
| `Host Hera` block | No tunnel exists. Nothing else works. | Tunnel built once per session. |
| `LocalForward` line | Just a normal SSH login to the bastion. No localhost shortcut. | Apt `MY_FORWARD_PORT` on laptop becomes a secret door to Hera. |
| YubiKey tap | The bastion refuses you. | You are authenticated, tunnel goes live. |
| Terminal #1 staying open | Tunnel collapses. `Hera-port` stops working. | Tunnel persists. Kiro keeps working. |
| `id_rsa` on laptop + `id_rsa.pub` in Hera's `authorized_keys` | Hera prompts for password (and Kiro cannot answer). | Hera trusts your laptop, no password. |
| `Host Hera-port` block | You would have to remember `ssh -p MY_FORWARDED_PORT user@localhost` every time. | Tools just say "Hera-port" and it works. |
| `ServerAlive*` | Idle timeouts kill the tunnel mid-day. | Heartbeats keep it alive. |

---

## 8. Common pitfalls

- **"Kiro suddenly cannot reach Hera."** First thing to check: is Terminal #1
  still alive? If the `Hera` session has been closed or has dropped, the
  tunnel is gone, and `Hera-port` will fail until you re-run the YubiKey
  login.
- **"It worked yesterday."** Laptops sleep. When the laptop sleeps long
  enough, the SSH heartbeats stop, the bastion drops the connection, and
  you wake up to a dead tunnel. Re-run Terminal #1.
- **"Kiro is asking for a password."** That means you reached Hera, but the
  passwordless key trust is not set up. Make sure `~/.ssh/id_rsa.pub` from
  the laptop is appended to `~/.ssh/authorized_keys` on Hera, with correct
  permissions (`chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`).
- **Port number mismatch.** `LocalForward` in the `Hera` block and `Port` in
  the `Hera-port` block must use the same number RDHPCS assigned to you. If
  they drift apart, the tunnel exists but no one is riding it.

---

## 9. Mental model to lock in

- A **port** is just an apartment number on a computer. Different services
  live at different apartments.
- **`localhost`** means "my own computer." Normally an empty building. Port
  forwarding puts a secret tube in one of its apartments.
- The **tunnel** is built by the YubiKey-protected `Hera` connection and
  only lives as long as that terminal stays open.
- **`Hera-port` is not really localhost.** It is Hera wearing a localhost
  mask. Kiro uses the mask because Kiro cannot deal with YubiKey prompts.
- **Rule of thumb when something breaks:** check Terminal #1. If the `Hera`
  session died, `Hera-port`, Kiro, AWS auth, the integrated terminal —
  everything that depended on the tunnel — will fail until you reopen it.

That is the whole trick. Once you have internalized *"the second host is
not really localhost, it is Hera in disguise,"* the rest of the RDHPCS
workflow stops feeling weird.
