### Linux PrivEsc Decision Tree.

sudo -l → NOPASSWD?
└── GTFObins → ROOT ✓

SUID binary?
├── Known binary (GTFObins) → ROOT ✓
└── Custom → strings/ltrace → PATH hijack → ROOT ✓

Capabilities?
└── cap_setuid → setuid(0) → ROOT ✓

Cron writable script?
└── Append reverse shell → ROOT ✓

/etc/passwd writable?
└── Add root user → ROOT ✓

NFS no_root_squash?
└── Mount + SUID bash → ROOT ✓

Docker group?
└── docker run -v /:/mnt → chroot → ROOT ✓

glibc 2.34–2.39 (Looney Tunables)?
└── CVE-2023-4911 PoC → ROOT ✓

Nothing?
└── Kernel exploit (check exact version)