==**AD in plain English**==

**Active Directory is the central "user database" for a Windows company network.** One server — the **Domain Controller (DC)** — holds every user, computer, password hash, and permission. Own the DC and you own the entire network.

**Why the chain works:** companies almost never give a normal user direct admin on the DC. So you climb sideways and upward: start as a low-priv user → enumerate to find a misconfiguration or a second set of creds → move to another machine (lateral movement) → repeat until you find a path to **Domain Admin**. [BloodHound](https://anshu19981.github.io/OscpCheckList2026/?utm_source=chatgpt.com#s16) literally draws this path for you.

**Key terms you'll see below:** _Kerberoasting_ and _AS-REP Roasting_ = stealing crackable password hashes from the way Windows logins work. _Pass-the-Hash_ = logging in with a stolen hash without ever knowing the plaintext password. _DCSync_ = impersonating a DC to dump everyone's hashes. Full definitions are in the [Glossary (Section 15)](https://anshu19981.github.io/OscpCheckList2026/?utm_source=chatgpt.com#s15).

==**Kerberos in plain English — read this before the "roasting" attacks**==

**Kerberos is how Windows logs you in without passing your password around.** Picture a cinema: you show ID once at the entrance and get a wristband (a _ticket_). After that, every screen lets you in by checking the wristband — you never show ID again.

**Why hackers care:** some of those tickets are encrypted with a _user's password hash_. If you can request such a ticket and take it offline, you can keep guessing passwords until one decrypts it. That's exactly what **Kerberoasting** and **AS-REP Roasting** (below) do — you're not breaking Kerberos, you're abusing a normal feature that hands you crackable material.


==How the 40 AD points actually work==

**The AD set is 3 chained machines worth 40 points total** — the highest-value and most predictable part of the exam. Points are banked as you progress: **MS01 local.txt = 10**, **MS02 local.txt = 10**, **DC proof.txt = 20**. So even if you don't fully own the DC, reaching MS01 and MS02 still banks 20 points.

**The one insight that makes AD click:** it's a _chain_, not three separate boxes. The credentials and hashes you loot on one machine are the keys to the next. Every phase is the same rhythm — enumerate → find a quick win → get a foothold → loot creds → reuse them to move forward → finally reach Domain Admin and own the DC.