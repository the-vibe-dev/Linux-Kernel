<p align="center">
  <img src="./assets/kernelbanner.png" alt="Linux kernel security contributions by Charles Vosburgh / the-vibe-dev" width="100%" />
</p>

# Linux Kernel Security Contributions

<p align="center">
  <img src="https://img.shields.io/badge/Accepted%20Mainline%20Fixes-4-0A66C2?style=for-the-badge" alt="4 accepted mainline security fixes" />
  <img src="https://img.shields.io/badge/Authored%20Patches-2-2EA44F?style=for-the-badge" alt="2 authored patches" />
  <img src="https://img.shields.io/badge/Reported--by%20Fixes-2-6F42C1?style=for-the-badge" alt="2 Reported-by fixes" />
  <img src="https://img.shields.io/badge/Verified%20Stable%20Backports-7-F59E0B?style=for-the-badge" alt="7 verified stable-tree backports of the first authored SCTP fix" />
</p>

Upstream Linux kernel security work by **Charles Vosburgh** / **[`the-vibe-dev`](https://github.com/the-vibe-dev)**.

This repository is the authoritative public portfolio for accepted Linux security fixes carrying Charles's author or reporter credit. CVE and GitHub Security Advisory research is documented separately in [`the-vibe-dev/CVE-GHSA`](https://github.com/the-vibe-dev/CVE-GHSA).

## Current contribution snapshot

| Metric | Verified public count |
|---|---:|
| Accepted mainline security fixes carrying Charles's credit | **4** |
| Patches authored and signed off by Charles | **2** |
| Fixes carrying `Reported-by: Charles Vosburgh` | **2** |
| Verified stable-tree backports of the first authored SCTP fix | **7** |
| Subsystems represented | **2** |

The seven-backport count applies only to `sctp: validate Adaptation Indication parameter length`. It is not a total across all four fixes.

## Accepted mainline contribution index

| Mainline commit | Subsystem | Contribution | Role | Stable status | Write-up |
|---|---|---|---|---|---|
| [`74b21f52c5c5`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) | SCTP networking (`net/sctp`) | `sctp: validate Adaptation Indication parameter length` (`CVE-2026-80717`, `GHSA-fq3m-cvqw-wrvh`) | Patch author + `Signed-off-by` | Seven verified stable-tree backports | [Read](contributions/74b21f52-sctp-adaptation-indication-length/) |
| [`6cfc1b90cb86`](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a) | SCTP networking (`net/sctp`) | `sctp: validate chunk length in the inqueue parser` | Patch author + `Signed-off-by` | No distinct stable backport verified | [Read](contributions/6cfc1b90-sctp-inqueue-chunk-length/) |
| [`e148e567a925`](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a) | KSMBD / SMB server (`fs/smb/server`) | `ksmbd: preserve VFS inherited POSIX ACL mask` | `Reported-by`; patch authored by Namjae Jeon | [Selected for an AUTOSEL series](https://lkml.iu.edu/2608.3/13998.html); no distinct stable commit verified | [Read](contributions/e148e567-ksmbd-posix-acl-mask/) |
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

## Credit roles

The upstream commits preserve two different contribution roles:

- Charles authored and signed off both SCTP fixes. Xin Long acknowledged them, and Jakub Kicinski integrated them through the networking tree.
- Charles reported both KSMBD issues. Namjae Jeon authored the accepted fixes, and Steve French signed them into the SMB tree.

Reporting credit is not presented as patch authorship.

## Submitted / under review

`sctp: validate Cookie Preservative parameter length` is publicly submitted and reviewed, but no mainline commit is verified. It is therefore not included in the four accepted fixes.

- [Patch submission](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/10866.html)
- [Maintainer review](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/10931.html)

## Repository layout

Each directory under `contributions/` corresponds to one accepted mainline fix and documents the security boundary, accepted change, validation evidence available in public sources, impact limits, credit, and upstream references. Private reproducers, raw lab evidence, credentials, and unpublished material are excluded.

## Contribution approach

The write-ups distinguish:

- patch authorship, `Signed-off-by`, `Reported-by`, review, and maintainer integration;
- public upstream evidence from non-public research artifacts;
- direct demonstrated behavior from broader inferred impact;
- mainline inclusion from distinct stable-tree backports; and
- accepted fixes from work that remains submitted or under review.

## Contributor

**Charles Vosburgh** — [`the-vibe-dev`](https://github.com/the-vibe-dev)

Independent security researcher and Linux kernel contributor working across protocol validation, filesystem/ACL behavior, SMB/KSMBD security boundaries, and low-level vulnerability analysis.

## Related research

- [CVE & GHSA Security Research](https://github.com/the-vibe-dev/CVE-GHSA)
- [SecHive.ai](https://sechive.ai) — operator-driven security research and validation tooling

## Responsible use

Technical material in this repository is intended for defensive research, regression testing, patch verification, and systems the reader owns or is explicitly authorized to assess.
