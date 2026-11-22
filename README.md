# [<img src="https://s3.vpndetection.io/vpndetection-public/brand/mark.svg" alt="VPNDetection" height="28"/>](https://vpndetection.io/) VPNDetection Skills

[![license](https://img.shields.io/github/license/vpndetection-io/skills)](LICENSE)

The official [VPNDetection](https://vpndetection.io) skills for coding agents: Claude Code, Codex, Cursor, Gemini CLI and the rest that read [Agent Skills](https://agentskills.io).

## Getting Started

```console
npx skills add vpndetection-io/skills
```

The [skills](https://skills.sh) CLI asks which agents to install them into, and copies each skill into that agent's skills folder. The skills are plain Markdown and run nothing themselves.

## What's in it

- `block-vpn-signups`: adds a sign-up check to a Next.js, Express, Better Auth, Django, Laravel or Rails app, refusing accounts from VPNs, Tor exits and relays (and residential proxies and datacenters, with a paid key) through our official libraries. It fails open, so an outage of ours never stops a sign-up.

## What it sends

Nothing, until your agent writes code that calls the VPNDetection API, and your app then sends that API the addresses it checks, with the key you give it. Our privacy policy is at https://vpndetection.io/privacy, and questions go to support@vpndetection.io.

## Other Libraries

There are official VPNDetection client libraries available for many languages including PHP, Python, Go, Java, and Ruby. See our GitHub at https://github.com/vpndetection-io for more.

## About VPNDetection

VPN Detection API: Accurate anonymity detection identifying VPNs, residential proxies, hosting servers, Tor nodes, CDNs, relays and more.

[<img src="https://s3.vpndetection.io/vpndetection-public/brand/mark.svg" alt="VPNDetection" height="64"/>](https://vpndetection.io/)

## License

This project is licensed under the [MIT License](LICENSE).
