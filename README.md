# K17 CTF 2026 — PiSigma Writeups

## 🏆 K17 CTF 2026

Our team PiSigma participated in K17 CTF 2026 and finished 68th.

This repository contains some of the challenges we worked on during the CTF, with a focus on Web Security and Cryptography.

I wanted to document these challenges not just by writing down the final payload or exploit, but by explaining how I approached them, what I noticed, what I tried, and how I eventually reached the solution.

---

## 📂 Writeups

### 🔐 Cryptography

* [Blowfish](./Crypto/Blowfish/)
* [Crypto Challenge 2](./Crypto/Crypto-Challenge-2/)

### 🌐 Web Security

* [DriveOne](./Web/DriveOne/)
* [whatsNew](./Web/whatsNew/)

---

## 🔎 How I Approach CTF Challenges

My approach usually starts with understanding what I'm actually dealing with before trying random payloads.

Depending on the challenge, I generally go through something like:

```text
Understand the target
        ↓
Look at the input / data flow
        ↓
Find something unusual
        ↓
Ask "what happens if I control this?"
        ↓
Test the idea
        ↓
If it fails → go back and look again
        ↓
Build the exploit
        ↓
Get the flag
```

A lot of the time, the important part isn't knowing a particular trick beforehand. It's noticing a small assumption in the application and then asking whether that assumption can be broken.

That's what I tried to capture in these writeups.

---

## 🧩 What You'll Find Here

The selected challenges cover things like:

* Cryptographic protocol weaknesses
* Block-level authentication
* Encryption/decryption oracles
* XOR-based manipulation
* TOCTOU vulnerabilities
* SQLite triggers
* File read vulnerabilities
* HTML parser differentials
* DOM clobbering
* Admin-bot exploitation

---

## 👥 Team

Team: PiSigma
K17 CTF Rank: 68th

This was a team effort, and the solutions documented here represent challenges worked on during our participation in the CTF.

---

## ⚠️ Disclaimer

These writeups are for educational and CTF purposes.

The techniques discussed here should only be used against systems where you have permission to test them.

