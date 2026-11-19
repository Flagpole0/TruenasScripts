# `truenas_config_dr.py` — Explained Like You're 10

## The big idea in one sentence

This script reads the **"save file"** of your TrueNAS storage box and turns it into a **printable instruction book**, so that if the box ever breaks, you (or someone else) know exactly how to rebuild it.

---

## 🏠 A story to picture it

Imagine you built an amazing LEGO castle. It has secret doors, a drawbridge, rooms with names, and a few locked treasure chests.

One day, your little brother knocks it over. 😱

If you'd written down **how** you built it (which rooms go where, which doors connect, which chest has which key), you could rebuild it exactly the same.

- **The castle** = your TrueNAS server (the computer that stores all your files).
- **The "how I built it" notes** = the TrueNAS **configuration file**.
- **This script** = a helper that reads those messy notes and writes a **neat, easy-to-read instruction book** called a *Disaster Recovery Record*.

"Disaster recovery" just means **"how to fix things after something really bad happens."**

---

## 📦 What goes in?

TrueNAS lets you download a backup of its settings. It comes as a `.tar` file (think of it as a zipped-up lunchbox) with two things inside:

| Thing in the lunchbox | What it is, simply |
|---|---|
| `freenas-v1.db` | A little database: a big organized list of **every setting** on the box. |
| `pwenc_secret` | The **"secret seed"**: a special key TrueNAS uses to lock up passwords inside that list. |

Without the secret seed, the passwords in the list look like scrambled gibberish, and **nobody** can unscramble them, not even TrueNAS when you restore. That's why the script nags you if it's missing.

---

## 📖 What comes out?

A document (Markdown, HTML, or both) that has:

1. **A cover page**: the server's name, when the book was made, and a **fingerprint** (SHA-256) of the backup file, so you can prove later you're using the right one.
2. **A step-by-step rebuild guide** ("Restore runbook"): install TrueNAS, upload the config, plug in the disks, unlock things, check that everything works.
3. **A "what's NOT in here" list**: your actual files, apps' data, passphrases you typed by hand, and so on. (The settings book isn't the treasure itself, just the map.)
4. **Chapters about every part of the server:**
   - 🌐 **Network**: its addresses and cables
   - 💾 **Storage**: the disk pools, snapshots (photos of your data at a moment in time), and copies sent to other machines
   - 📂 **Shares**: the folders other computers can open
   - 👤 **Users & groups**: who's allowed in
   - ⚙️ **Services**: which features are switched on
   - ☁️ **Cloud & offsite**: backups sent to the internet, plus their login info
   - 🖥️ **System**: time, email alerts, certificates, scheduled jobs
   - 📦 **Virtual machines & containers**: mini-computers running inside the server
5. **An appendix**: "everything else we found that wasn't in a chapter," so nothing gets forgotten.

---

## 🔒 The really careful part: keeping secrets secret

This script is **very** careful with passwords. Here's how, in kid terms.

### It never phones home
It doesn't talk to your server at all. It only reads the backup file. You can run it on a computer with **no internet** and the server never even knows. 🤫

### It reads, it doesn't write
It opens the settings list in **"look, don't touch"** mode. It can't change anything by accident.

### It doesn't leave crumbs
The settings list is loaded into the computer's **short-term memory** (RAM) instead of being unpacked onto the hard drive. When the script ends, it's gone.

### Two modes: 🙈 Redacted vs. 👀 Unredacted

| | **Redacted** (the default) | **Unredacted** |
|---|---|---|
| Passwords look like | `[set]` or `[empty]` | the real password |
| Safe to print and share? | ✅ Yes | ❌ No, lock it in a fire safe |
| How you get it | Just run it | You must **type the server's name** to prove you mean it, then say "yes" again |

It's like a diary with a lock: the normal version shows "there's a secret here," and only someone who really, *really* means it gets to see what the secret says.

### How does it know what's a secret?
It looks at the **name** of each setting. If the name has words like `password`, `secret`, `token`, `key`, or `psk` in it, it treats it as secret. If TrueNAS stored something scrambled, the script assumes it's secret **unless** it's on a short "this is harmless" list (like `host`, `port`, `username`). When in doubt, it hides it. That's called **failing closed**: if unsure, keep the door shut.

### ⚠️ The one thing it can't catch
If you typed a password **inside a command** (for example, a scheduled job like `curl --password hunter2 ...`), the script can't tell that's a password. It'll show it. So don't put passwords in command lines!

### Safe saving
- Output files are made so **only you** can open them (permission `0600`).
- It writes to a hidden temporary file first, then swaps it into place all at once, so you never get a half-written file.
- Optionally, it can **lock the file with GPG** (a strong encryption tool) *as it's being written*, so the readable version never touches the disk at all.

---

## 🔑 Unscrambling passwords (the decryption bit)

TrueNAS scrambles stored passwords using **AES**, a very strong lock. The `pwenc_secret` file is the key.

When you give the script both the list and the key, it:
1. Spots values that **look** scrambled (they have a very specific size and shape).
2. Tries to unlock them with the key.
3. Checks the result looks like real readable text. If it's still gibberish, it says "couldn't unlock this" and explains why (missing key? wrong server's key? missing helper library?).

Even when it *can* unlock passwords, it only **shows** them in unredacted mode. In redacted mode it just counts them.

---

## 🧭 How you use it

### Option A: the friendly wizard (just run it)
```bash
python3 truenas_config_dr.py
```
It asks you questions one at a time:
1. **Which backup file?** (it looks in your current folder and Downloads for you)
2. **Redacted or unredacted?**
3. **Which chapters?** (all, by default)
4. **Short summary or every single detail?**
5. **Markdown, HTML, or both?**
6. **Where to save it, and should it be encrypted?**

### Option B: no questions (batch mode, for automation)
```bash
# Safe, redacted copy
python3 truenas_config_dr.py -c backup.tar --batch

# Full secret copy, encrypted to your GPG key
python3 truenas_config_dr.py -c backup.tar --batch --unredacted \
    --confirm-host faultline --gpg-recipient you@example.com
```

### Handy switches
| Switch | What it does |
|---|---|
| `-c FILE` | Which backup file to read |
| `--batch` | Don't ask questions |
| `--unredacted` + `--confirm-host NAME` | Show real secrets (needs the server name as a safety check) |
| `--summary` | Short tables only |
| `--no-appendix` | Skip the "everything else" chapter |
| `--builtin` | Include TrueNAS's own built-in system users |
| `--sections network,storage` | Only some chapters |
| `--format md` / `html` / `md,html` | What kind of file to make |
| `-o FOLDER` | Where to save |
| `--gpg-recipient` / `--gpg-symmetric` | Encrypt with a key / with a passphrase |

---

## 🧩 How the code is organized (the "rooms" of the script)

Think of the script as a little factory with stations on a conveyor belt:

```
 backup.tar
     │
     ▼
┌──────────────┐   Opens the lunchbox, finds the database and the secret seed,
│ load_config  │   checks it's not too big or fake, loads it into memory.
└──────┬───────┘
       ▼
┌──────────────┐   Holds everything while working: the database, the
│     Ctx      │   unscrambler (Pwenc), redacted/unredacted mode, counters.
└──────┬───────┘
       ▼
┌──────────────┐   For each setting: is it empty? yes/no? scrambled?
│ value/process│   secret? JSON? Turns raw data into friendly labels.
└──────┬───────┘
       ▼
┌──────────────┐   Walks through the chapter list (SECTIONS) and builds
│  build_doc   │   the cover page, rebuild steps, chapters, appendix.
└──────┬───────┘
       ▼
┌──────────────┐   Turns the book into Markdown and/or HTML
│ render_md /  │   (with safe escaping so weird text can't break it).
│ render_html  │
└──────┬───────┘
       ▼
┌──────────────┐   Saves the files safely (private, all-at-once),
│     emit     │   or pipes them straight into GPG to lock them.
└──────┬───────┘
       ▼
┌──────────────┐   Prints "Done!", file sizes, how many secrets were found,
│    report    │   and a warning if plaintext secrets are sitting on disk.
└──────────────┘
```

### A few clever little tricks worth knowing
- **It survives TrueNAS version changes.** If a table or setting doesn't exist in your version, it just skips it instead of crashing.
- **It turns ugly names into nice ones.** `cifs_ro` becomes `ro`, and five separate schedule columns (minute, hour, day, month, weekday) get glued into one readable schedule like `0 3 * * *`.
- **It follows links.** If a setting says "user #2," it looks up and writes `alex (#2)` instead.
- **It reads certificate expiry dates** so you can see when your HTTPS certificates run out.
- **It guesses the TrueNAS version from the file name**, because the database itself doesn't store it.
- **It treats the backup file as untrusted.** Names from inside the database are quoted safely before use, odd control characters are stripped, and Markdown symbols are escaped, so a weird value can't break the document or your terminal.

---

## 🧠 TL;DR

> **It's a librarian for your server's settings.** It reads the backup, writes a tidy rebuild book, hides passwords unless you *really* ask, and never touches the real server.

Keep a **redacted** copy anywhere handy. Keep an **unredacted** copy encrypted, offline, somewhere safe, like a fire safe or (fittingly) etched on a steel plate. 🔐
