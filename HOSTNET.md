# MySpeed — hostnet variant

> **Unofficial variant** of [MySpeed](https://github.com/gnmyt/myspeed) by [gnmyt](https://github.com/gnmyt), based on upstream **1.0.9**.
> It is not affiliated with or endorsed by the original project. All credit for MySpeed goes to its authors.

This variant is meant to run MySpeed with **`network_mode: host`**, so speed tests are not measured through Docker's bridge/NAT.

## Why

With the default bridge network, every packet of a speed test goes through Docker's NAT (iptables/conntrack). On small hosts such as a Raspberry Pi, this mostly hurts the **download** measurement. On a Raspberry Pi 5 with a 2.5 GbE adapter we measured:

| Setup | Download | Upload |
|---|---|---|
| Official image, bridge network | ~200–500 Mbps | ~2000 Mbps |
| This variant, host network | ~1100–2300 Mbps* | ~2300 Mbps |

\* The upper value was reached with a nearby test server from the same ISP. The test server you pick matters as much as the network mode.

Running the official image with `network_mode: host` mostly works too. However, the container then sees every host interface: docker bridges, veth pairs, VPNs such as Tailscale or WireGuard. If the interface saved in the config does not exist on the host, MySpeed falls back to an arbitrary one. This variant fixes that.

## What changed

The only code change is in `server/util/loadInterfaces.js`:

- Virtual interfaces (`veth*`, `br-*`, `docker*`, `virbr*`, `tailscale*`, `wg*`, `tun*`, `tap*`, `zt*`) are skipped when looking for usable interfaces. This also makes startup faster.
- When the configured interface is missing, MySpeed falls back to the **interface that carries the default route** instead of the first one found.
- The interface list is refreshed every hour without being emptied first, so a test running during the refresh does not lose its interface.

Everything else is identical to upstream. Your existing `data/` folder (database, settings, history) works as is. Database migrations run automatically on first start.

## Usage

```yaml
services:
    myspeed:
        image: ghcr.io/jrobertoma/myspeed-hostnet:latest   # or jrobertoma/myspeed-hostnet:latest
        network_mode: host
        container_name: myspeed
        volumes:
            - ./data:/myspeed/data
        environment:
            - TZ=Etc/UTC
        restart: unless-stopped
```

The web interface is available at `http://<host>:5216`. With host networking there is no `ports:` mapping. To use another port, set `SERVER_PORT`.

After the first start, check *Settings → Interface* and make sure your physical interface (for example `eth0` or `eth1`) is selected.

## Images

| Registry | Image |
|---|---|
| GitHub Container Registry | `ghcr.io/jrobertoma/myspeed-hostnet` |
| Docker Hub | `jrobertoma/myspeed-hostnet` |

Platforms: `linux/amd64`, `linux/arm64`. 32-bit ARM is not supported, because the Bun base image has no 32-bit build.

Tags follow `<upstream version>-hostnet.<revision>` (for example `1.0.9-hostnet.1`), plus `latest`.

Source code: [github.com/JRobertoMA/MySpeed](https://github.com/JRobertoMA/MySpeed) (branch `hostnet`).

## License

MIT, same as upstream. See [LICENSE](https://github.com/JRobertoMA/MySpeed/blob/hostnet/LICENSE).
