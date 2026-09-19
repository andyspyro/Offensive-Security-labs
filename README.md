# Offensive Security Labs | Hack The Box

This repo contains my authorized offensive security lab work.

I keep the parts that matter to me later: how I enumerated the target, what assumption changed, how I got access, what I checked after access, and what the defensive fix would be.

## Machines with retained notes

| Machine | Status I can support | What I worked on | Evidence I kept |
|---|---|---|---|
| [Blue](blue/README.md) | **Completed: user and root** | Windows 7, SMB enumeration, MS17-010 validation, EternalBlue, Meterpreter, WSL payload/networking decisions | Full penetration test report with attack chain and lessons learned |
| [Paperwork](paperwork/README.md) | **Completed: user and root** | Source review, command injection, reverse shell, internal service discovery, arbitrary file write, SSH access, UNIX socket privilege escalation | Full retained penetration test report |
| [Orion](orion/README.md) | In progress | Craft CMS 5.6.16, public PoC troubleshooting, Python, session state, CSRF, MySQL credential recovery | HTB screenshots and Python exploit work |
| [Valentine](valentine/README.md) | User access obtained | SSH key handling, old RSA compatibility, Bash history, tmux lead | Screenshot showing SSH access as `hype` and retained shell history |
| [CrossFitTwo](crossfittwo/README.md) | Enumeration and web analysis | OpenBSD, virtual hosts, password reset behavior, ffuf account enumeration | Retained ffuf and curl screenshot |
| [DanglingTree](danglingtree/README.md) | In progress | Windows, IIS, Burp, SmarterMail, log review, credential discovery, account pivoting | Retained notes and screenshots; no confirmed administrator result |

## How I work through a box

My longer process is in [METHODOLOGY.md](METHODOLOGY.md).

The short version is:

1. Confirm the VPN and target.
2. Enumerate services before choosing an exploit.
3. Fix hostname and virtual host issues early.
4. Learn what a normal request looks like.
5. Follow controlled input.
6. Test one idea at a time.
7. Enumerate again after access.
8. Treat privilege escalation as a separate problem.
9. Write down the root cause and fix.

## Tools I have actually used

Nmap, Burp Suite, curl, ffuf, Python, Ncat, OpenSSH, Searchsploit, smbclient, Metasploit Framework, Meterpreter, PowerShell, WSL, Bash, Linux process tools, service inspection, and browser developer tools.

## Earlier HTB activity

I also have earlier activity records for Jarmis, Spooktrol, Scanned, Rainbow, Zero, Reaper, ReaperTwo, and OneTwoSeven.

I do not have enough retained technical evidence to write reliable walkthroughs for those machines, so I am not attaching findings or completion claims to them.

## Scope

Everything in this repo comes from authorized Hack The Box lab systems.

I do not publish flags, live credentials, private keys, reusable passwords, or session tokens.

## Related work

* [Application Security Labs](https://github.com/andyspyro/Application-security-labs)
* [Security Engineering Projects](https://github.com/andyspyro/Security-engineering-projects)
* [Cybersecurity Portfolio](https://github.com/andyspyro/Cybersecurity-Portfolio)
