**Local admin/root on MS01 achieved** → before touching MS02, in order:
    ├── Dump LSASS / LSA secrets / mscache (Windows) or hunt keytabs+ccache (Linux)
    ├── Check who's logged in / recently logged in — `quser`, `net session`, `.rdp`/`.rdg` files
    ├── Grep configs/history for plaintext creds (web.config, PS history, bash_history)
    ├── Re-run BloodHound with any NEW creds found — new user = new attack paths
    └── **Now pivot to MS02 with whatever you looted (hash → PtH, ticket → PtT, plaintext → direct login)**