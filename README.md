<p align="center">
  <img src="./assets/kernelbanner.png" alt="Linux kernel security contributions by Charles Vosburgh / the-vibe-dev" width="100%" />
</p>

# Linux Kernel Security Contributions

<p align="center">
  <img src="https://img.shields.io/badge/Accepted%20Mainline%20Fixes-4-0A66C2?style=for-the-badge" alt="4 accepted mainline security fixes" />
  <img src="https://img.shields.io/badge/Authored%20Patches-2-2EA44F?style=for-the-badge" alt="2 authored patches" />
  <img src="https://img.shields.io/badge/Reported--by%20Fixes-2-6F42C1?style=for-the-badge" alt="2 Reported-by fixes" />
  <img src="https://img.shields.io/badge/Verified%20Stable%20Backports-18-F59E0B?style=for-the-badge" alt="18 verified stable backports: fourteen SCTP and four KSMBD ACL fixes" />
</p>

I'm **Charles Vosburgh** / **[`the-vibe-dev`](https://github.com/the-vibe-dev)**. My Linux kernel work focuses on protocol validation, filesystem permissions, and SMB security boundaries.

This portfolio brings together my accepted upstream patches and reported findings, with technical write-ups and links to mainline commits and stable backports. My wider vulnerability research is in [`CVE-GHSA`](https://github.com/the-vibe-dev/CVE-GHSA).

## Current contribution snapshot

| My upstream contributions | Count |
|---|---:|
| Accepted mainline security fixes | **4** |
| Patches I authored and signed off | **2** |
| Fixes crediting me as reporter | **2** |
| Verified stable-tree backports | **18** |
| Subsystems represented | **2** |
| Public CVEs mapped to accepted fixes | **3** |

Seven stable backports carry my SCTP Adaptation Indication fix, seven carry the SCTP inqueue fix, and four carry the KSMBD inherited-ACL fix. My accepted work includes **CVE-2026-80717**, **CVE-2026-99128**, and **CVE-2026-93786**. Release mappings were refreshed November 2, 2026.

## Accepted mainline contribution index

| Mainline commit | Subsystem | Contribution | Role | Stable status | Write-up |
|---|---|---|---|---|---|
| [`74b21f52c5c5`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) | SCTP networking (`net/sctp`) | `sctp: validate Adaptation Indication parameter length` (`CVE-2026-80717`, `GHSA-fq3m-cvqw-wrvh`) | Patch author + `Signed-off-by` | Seven verified stable-tree backports | [Read](contributions/74b21f52-sctp-adaptation-indication-length/) |
| [`6cfc1b90cb86`](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a) | SCTP networking (`net/sctp`) | `sctp: validate chunk length in the inqueue parser` (`CVE-2026-99128`) | Patch author + `Signed-off-by` | Seven verified released stable backports | [Read](contributions/6cfc1b90-sctp-inqueue-chunk-length/) |
| [`e148e567a925`](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a) | KSMBD / SMB server (`fs/smb/server`) | `ksmbd: preserve VFS inherited POSIX ACL mask` (`CVE-2026-93786`) | `Reported-by`; patch authored by Namjae Jeon | Released in 6.18.53, 6.12.111, 6.6.158, and 6.1.191 | [Read](contributions/e148e567-ksmbd-posix-acl-mask/) |
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

The SCTP inqueue fix has seven released backports, mapped by the [Linux CNA record](https://raw.githubusercontent.com/CVEProject/cvelistV5/main/cves/2026/99xxx/CVE-2026-99128.json) and verified against released source:

| Stable release | Stable commit |
|---|---|
| Linux 7.2.13 | [`9ac57e4c62e9`](https://git.kernel.org/stable/c/9ac57e4c62e93c287b36080e5da3645690549990) |
| Linux 6.18.59 | [`40507f59f230`](https://git.kernel.org/stable/c/40507f59f230dcf60ae13cb48d34f8c6e9f40659) |
| Linux 6.12.115 | [`e0fb7fe18048`](https://git.kernel.org/stable/c/e0fb7fe180488a5afa2e91509ea838e8df8cb771) |
| Linux 6.6.161 | [`5a7ee52589da`](https://git.kernel.org/stable/c/5a7ee52589dae36c93dc64d7a97c5ded84618572) |
| Linux 6.1.191 | [`d28ea1a62e27`](https://git.kernel.org/stable/c/d28ea1a62e27210d8ea3cf496dfae85dbdd7b84f) |
| Linux 5.15.224 | [`b0dbd53b2426`](https://git.kernel.org/stable/c/b0dbd53b242609c88e81b0a49093bdbef5d5a418) |
| Linux 5.10.273 | [`cc4a4f1d6b6d`](https://git.kernel.org/stable/c/cc4a4f1d6b6d725438dc37b97f03d95ed9feb63f) |

The inherited-ACL fix also has four released backports:

| Stable release | Stable commit |
|---|---|
| Linux 6.18.53 | [`0093909becbd`](https://git.kernel.org/stable/c/0093909becbda22b62b39b654dbc607faed59cd0) |
| Linux 6.12.111 | [`591acf171644`](https://git.kernel.org/stable/c/591acf171644de590cda245fd8595b473a1beb70) |
| Linux 6.6.158 | [`4a19bb1c4566`](https://git.kernel.org/stable/c/4a19bb1c45667f5a5f6eff9cf6648e0f692d046f) |
| Linux 6.1.191 | [`8493da704275`](https://git.kernel.org/stable/c/8493da704275ca5ab67fb88c0ae7b71576e91951) |

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
