Image references for 03-Network-Forensics

| File | Contents |
|---|---|
| `private-key-access-results.png` | Splunk `private-key-access.spl` results — 21 events, all `xrdp` denied reads of `/etc/xrdp/key.pem` (service noise, not credential theft) |
| `chroot_query_results.png` | Splunk `chroot-attempts.spl` results — 97 events, all `rtkit-daemon: Successfully called chroot` (expected background, not container escape) |
| `vulners_terminal_output.png` | Nmap NSE `vulners` output against `scanme.nmap.org` — OpenSSH 6.6.1p1 CVE cross-reference (leads, not confirmed findings; see README caveat) |
