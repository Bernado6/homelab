# Troubleshooting

Problems I hit along the way, and what fixed them. Ordered roughly by when they came up.

| Problem | Cause / fix | Step |
|---|---|---|
| No Wi-Fi in Windows | OptiPlex Micros usually have no Wi-Fi card. Use Ethernet. | [7](02-hardware/07-display-keyboard-and-network.md) |
| Chrome blocks Rufus | "Uncommon download". `Ctrl + J` → Keep. Check the signature is from Akeo Consulting. | [8](03-ubuntu-server/08-making-the-install-usb.md) |
| F12 → USB goes back to Windows | 1) Check the flash drive has Ubuntu files (`boot`, `casper`, `EFI`); if not, re-flash, trying **DD Image mode**. 2) Use a **rear** USB port. 3) Make sure you selected the USB entry. 4) Last resort: temporarily disable Secure Boot. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| Installer can't see the SSD | BIOS → SATA Operation → **AHCI**. | [9](03-ubuntu-server/09-bios-settings.md) |
| Ubuntu only uses ~100GB | The LVM default. Untick LVM during install, or extend the volume later. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| Installer asks for a proxy | Leave it blank at home. | [10](03-ubuntu-server/10-installing-ubuntu-server.md) |
| `hostname -i` shows `127.0.1.1` | Use `hostname -I` or `ip -4 addr`. | [11](03-ubuntu-server/11-first-login-and-ssh.md) |
| Server IP changed | Set a DHCP reservation in the router. | [14](05-networking/14-dhcp-reservation.md) |
| `helm-install-traefik` pod restarts after K3s install | Normal at first boot. It retries until the cluster is ready, then shows **Completed**. | [12](04-kubernetes/12-installing-k3s.md) |
| `port-forward` says "does not have a service port 80" | With `svc/<name>`, the right-hand port is the **Service** port, not the container port. `svc/nginx-svc 8000:5000` works and kubectl maps 5000 → 80 for you. With `pod/<name>` use the container port directly. | [13](04-kubernetes/13-first-app.md) |
| `port-forward` works on the server but `localhost:8000` on the laptop says connection refused | The forward listens on the **server's** localhost. Either run it with `--address 0.0.0.0` and open `http://<SERVER_IP>:8000`, or tunnel from the laptop: `ssh -L 8000:localhost:8000 user@<SERVER_IP>` then open `http://localhost:8000`. | [13](04-kubernetes/13-first-app.md) |
| `http://quantum:8000` from the laptop gives `DNS_PROBE_FINISHED_NXDOMAIN` | The server's hostname is not known to your laptop or router. Use the IP, or add `<SERVER_IP>  quantum` to `/etc/hosts` on the laptop (only reliable once you have a DHCP reservation). | [14](05-networking/14-dhcp-reservation.md) |
| Browser shows **Not Secure** | Plain HTTP on the home network. Expected until HTTPS with cert-manager is set up (see roadmap). | |
| Router admin page `192.168.100.1` times out from the laptop, but ping works | Huawei HG8145V5 only serves its admin UI to **wired** clients. From the wired server, `curl -I http://192.168.100.1` returns 200. | [14](05-networking/14-dhcp-reservation.md) |
| SSH tunnel to the router bounces to `https://localhost` and refuses | The router's login page redirects to HTTPS without the port. Forward 443 onto the laptop's own 443: `sudo ssh -L 443:192.168.100.1:443 -L 8080:192.168.100.1:80 user@<SERVER_IP>`, then open `https://localhost`. | [14](05-networking/14-dhcp-reservation.md) |
| Router login asks for an **SN** | Serial number on the label on the back of the router, 16 hex characters starting `4857`. | [14](05-networking/14-dhcp-reservation.md) |
| `hostname -I` shows `10.42.0.0 10.42.0.1` as well as the LAN IP | K3s pod-network interfaces (`flannel.1`, `cni0`). Only the `192.168.x.x` address is reachable from the LAN. | [14](05-networking/14-dhcp-reservation.md) |
| `error: no context exists with the name: "homelab"` in a new terminal | `KUBECONFIG` isn't set in this shell, so kubectl only sees `~/.kube/config`. Run the export from the [Daily Checklist](daily-checklist.md), or persist it. | [15](06-working-with-the-cluster/15-kubectl-from-laptop.md) |
| Grafana pod `2/3 CrashLoopBackOff`, `OOMKilled`, exit code 137 | Memory limit too low for startup. Raise `grafana.resources.limits.memory` in `apps/monitoring/values.yaml` and `helm upgrade`. | [18](08-observability/18-prometheus-grafana.md) |
| `strict decoding error: unknown field "spec.rules"` (or `spec.tls`, `spec.ingressClassName`) on apply | Ingress fields pasted into a Deployment, or any snippet pasted into the wrong `kind`. The rejected field names say which kind you meant. | [20](09-tls/20-https-with-cert-manager.md) |
| Certificate stuck at `READY False` | `kubectl -n <ns> describe certificate <name>`; the Events say why. Usually a typo in the issuer name or a host not in the Ingress `tls.hosts`. | [20](09-tls/20-https-with-cert-manager.md) |
| Browser trusts the site but `curl` fails the certificate | Homebrew `curl` has its own CA bundle and ignores the macOS keychain. `curl --cacert ~/homelab-root-ca.crt …`, or use `/usr/bin/curl`. | [20](09-tls/20-https-with-cert-manager.md) |
| `ImagePullBackOff` with `lookup registry-1.docker.io: Try again`, and on the server `dig` says `127.0.0.53#53: connection refused` while `dig @1.1.1.1` works | A `type: LoadBalancer` Service on port 53 made ServiceLB redirect **all** local port-53 traffic, including the node's own stub resolver, into a pod that isn't running, so the node can't resolve the registry to pull that pod's image. Take the node off the stub: `DNSStubListener=no`, fixed upstreams in netplan, symlink `/etc/resolv.conf` to `/run/systemd/resolve/resolv.conf`, restart CoreDNS. Do it **before** creating the Service. | [21](10-dns/21-local-dns-adguard.md) |
| Shell hangs with a `>` prompt after pasting a `tee … <<'EOF'` block | The closing `EOF` was indented; a heredoc terminator must start at column one. `Ctrl+C`, then paste without indentation, or use `printf … \| sudo tee` instead. | [21](10-dns/21-local-dns-adguard.md) |
| `netplan apply`: *networkd requires that use-dns has the same value in both dhcp4_overrides and dhcp6_overrides* | Only `dhcp4-overrides` was set. Add `dhcp6-overrides: {use-dns: false}` too. | [21](10-dns/21-local-dns-adguard.md) |
| `netplan apply`: *expected mapping (check indentation)* pointing at `addresses:[…]` | Missing space after the colon. Fix the line in an editor; the `- item` list form is paste-proof. | [21](10-dns/21-local-dns-adguard.md) |

---

[◀ Previous: Public Access With Cloudflare Tunnel](11-public-access/23-cloudflare-tunnel.md) · [🏠 Home](../README.md) · [Next: Lessons Learned ▶](lessons-learned.md)
