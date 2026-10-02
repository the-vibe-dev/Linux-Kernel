<p align="center">
  <img src="./assets/kernelbanner.png" alt="Linux kernel security contributions by Charles Vosburgh / the-vibe-dev" width="100%" />
</p>

# Linux Kernel Security Contributions

<p align="center">
  <img src="https://img.shields.io/badge/Accepted%20Mainline%20Fixes-4-0A66C2?style=for-the-badge" alt="4 accepted mainline security fixes" />
  <img src="https://img.shields.io/badge/Authored%20Patches-2-2EA44F?style=for-the-badge" alt="2 authored patches" />
  <img src="https://img.shields.io/badge/Reported--by%20Fixes-2-6F42C1?style=for-the-badge" alt="2 Reported-by fixes" />
  <img src="https://img.shields.io/badge/Verified%20Stable%20Backports-9-F59E0B?style=for-the-badge" alt="9 verified stable backports: seven SCTP and two KSMBD ACL fixes" />
</p>

I'm **Charles Vosburgh** / **[`the-vibe-dev`](https://github.com/the-vibe-dev)**. My Linux kernel work focuses on protocol validation, filesystem permissions, and SMB security boundaries.

This portfolio brings together my accepted upstream patches and reported findings, with technical write-ups and links to mainline commits and stable backports. My wider vulnerability research is in [`CVE-GHSA`](https://github.com/the-vibe-dev/CVE-GHSA).

## Current contribution snapshot

| My upstream contributions | Count |
|---|---:|
| Accepted mainline security fixes | **4** |
| Patches I authored and signed off | **2** |
| Fixes crediting me as reporter | **2** |
| Verified stable-tree backports | **9** |
| Subsystems represented | **2** |
| Public CVEs mapped to accepted fixes | **2** |

Seven stable backports carry my SCTP Adaptation Indication fix; two carry the KSMBD inherited-ACL fix. My accepted work includes **CVE-2026-80717** and **CVE-2026-93786**.

## Accepted mainline contribution index

| Mainline commit | Subsystem | Contribution | Role | Stable status | Write-up |
|---|---|---|---|---|---|
| [`74b21f52c5c5`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) | SCTP networking (`net/sctp`) | `sctp: validate Adaptation Indication parameter length` (`CVE-2026-80717`, `GHSA-fq3m-cvqw-wrvh`) | Patch author + `Signed-off-by` | Seven verified stable-tree backports | [Read](contributions/74b21f52-sctp-adaptation-indication-length/) |
| [`6cfc1b90cb86`](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a) | SCTP networking (`net/sctp`) | `sctp: validate chunk length in the inqueue parser` | Patch author + `Signed-off-by` | No distinct stable backport verified | [Read](contributions/6cfc1b90-sctp-inqueue-chunk-length/) |
| [`e148e567a925`](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a) | KSMBD / SMB server (`fs/smb/server`) | `ksmbd: preserve VFS inherited POSIX ACL mask` (`CVE-2026-93786`) | `Reported-by`; patch authored by Namjae Jeon | Released in 6.18.53 and 6.12.111; 6.6 stable review announced September 30 | [Read](contributions/e148e567-ksmbd-posix-acl-mask/) |
| [`2bebf2470af1`](https://github.com/torvalds/linux/commit/2bebf2470af1a72f87754a5c7b21e86af32b9c8f) | KSMBD / SMB server (`fs/smb/server`) | `ksmbd: enforce signing required by the session` | `Reported-by`; patch authored by Namjae Jeon | No stable backport verified | [Read](contributions/2bebf247-ksmbd-session-signing/) |

## Stable-tree backports

The first authored SCTP fix is present in seven verified stable branches:

| Stable branch | Stable commit |
|---|---|
| Linux 7.1.y | [`bfa28cf99eb4d096c87da939f54233444d209ca5`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=bfa28cf99eb4d096c87da939f54233444d209ca5) |
| Linux 6.18.y | [`17b412468c7a44f66a385bda48cdc1e94e39bd6d`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=17b412468c7a44f66a385bda48cdc1e94e39bd6d) |
| Linux 6.12.y | [`5fd7cfc708dfc988ae9920c21075e6121bc89926`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=5fd7cfc708dfc988ae9920c21075e6121bc89926) |
| Linux 6.6.y | [`93942b5772e0eee4147d4799cc1b936ae12fa615`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=93942b5772e0eee4147d4799cc1b936ae12fa615) |
| Linux 6.1.y | [`7b7e4e3640d57bd8857f0052c8b0d8ed4e5e954a`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=7b7e4e3640d57bd8857f0052c8b0d8ed4e5e954a) |
| Linux 5.15.y | [`4c92c601c061e5602db2edeea54fef74aa304027`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=4c92c601c061e5602db2edeea54fef74aa304027) |
| Linux 5.10.y | [`fa7861ddbe3b525b5d541c15c3953d3569e6eb0e`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=fa7861ddbe3b525b5d541c15c3953d3569e6eb0e) |

The inherited-ACL fix also has released backports:

| Stable release | Stable commit |
|---|---|
| Linux 6.18.53 | [`0093909becbd`](https://git.kernel.org/stable/c/0093909becbda22b62b39b654dbc607faed59cd0) |
| Linux 6.12.111 | [`591acf171644`](https://git.kernel.org/stable/c/591acf171644de590cda245fd8595b473a1beb70) |

## Upstream collaboration

I authored both SCTP fixes, acknowledged by Xin Long and integrated by Jakub Kicinski through the networking tree. I reported the two KSMBD findings; Namjae Jeon authored their fixes, with Steve French providing SMB tree integration.

## Submitted / under review

My `sctp: validate Cookie Preservative parameter length` patch is publicly submitted and reviewed, with mainline integration pending.

- [Patch submission](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/10866.html)
- [Maintainer review](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/10931.html)

## Repository layout

Browse `contributions/` for each fix's technical explanation, upstream history, credit, and public references.

## Contribution approach

I trace protocol and permission decisions to the kernel code that enforces them, then work with maintainers on focused fixes. My write-ups explain the failure, its practical limits, and how the accepted patch restores the intended behavior.

## Contributor

**Charles Vosburgh** — [`the-vibe-dev`](https://github.com/the-vibe-dev)

I'm an independent security researcher and builder working across protocol validation, filesystem ACLs, SMB/KSMBD security, and low-level vulnerability analysis.

## Related research

- [CVE & GHSA Security Research](https://github.com/the-vibe-dev/CVE-GHSA)
- [SecHive.ai](https://sechive.ai) — operator-driven security research and validation tooling

## Responsible use

Technical material in this repository is intended for defensive research, regression testing, patch verification, and systems the reader owns or is explicitly authorized to assess.
