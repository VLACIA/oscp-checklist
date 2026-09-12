**Pivoting = using a machine you own as a stepping stone to reach machines you can't see.** In the AD set, you can reach MS01 directly — but MS02 and the DC sit on an internal network your Kali box has no route to. MS01 _can_ talk to them, so you turn MS01 into a tunnel and route your tools _through_ it.

**Simple analogy:** you can't enter a locked office building, but you've befriended someone inside who passes your messages back and forth. That insider is your pivot.

**For OSCP, learn one tool well — Ligolo-ng.** It's the simplest: start the proxy on Kali, run the agent on MS01, and suddenly the internal subnet behaves like it's directly reachable. Don't drown in five different tools; master Ligolo and keep Chisel/SSH as backups.

