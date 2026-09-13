# CVE-2026-80717 — sctp: validate Adaptation Indication parameter length

- **CVE:** [CVE-2026-80717](https://www.cve.org/CVERecord?id=CVE-2026-80717)
- **Global advisory:** [GHSA-fq3m-cvqw-wrvh](https://github.com/advisories/GHSA-fq3m-cvqw-wrvh)
- **Linux mainline:** [`74b21f52c5c5a71a05c0ff70e513f4f04ff28b17`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17)
- **Subsystem:** `net/sctp`
- **Role:** **Patch author + `Signed-off-by`**
- **Author:** Charles Vosburgh (`the-vibe-dev`)

---

## Executive Summary

An unauthenticated remote peer, Mallory, can send an SCTP INIT containing a header-only Adaptation Layer Indication parameter. The generic TLV walker accepts the four-byte header, although the concrete parameter also requires a 32-bit Adaptation Code Point. Later association processing reads that absent field immediately beyond the peer-declared parameter. Under the validated non-clearing receive-buffer profile, the four-byte value can be copied into association state and reflected to Mallory in the state cookie returned with the INIT ACK. Mallory does not need to steal a session, authenticate, or break SCTP cryptography; this is a bounded parser over-read, not an arbitrary-address read or code-execution primitive.

The public fix lineage points back to `1da177e4c3f4 ("Linux-2.6.12-rc2")`, but this report does not claim that every intervening release was independently executed. Vulnerable source was directly reviewed in Linux 7.1.4, Linux 7.2-rc4, and the pre-fix networking-tree base. Linux 7.2 is the first final mainline release containing the fix. The same correction is present in seven verified stable branches, including Linux 7.1.y beginning with 7.1.8.

I reviewed the public vulnerable source, accepted fix, CVE/GHSA record, and stable backports directly. The runtime results below are preserved from the documented owned KVM/QEMU validation; I did not rerun the network trigger for this CVE metadata update.

| Field | Value |
|---|---|
| CVE | [CVE-2026-80717](https://www.cve.org/CVERecord?id=CVE-2026-80717) |
| Global advisory | [GHSA-fq3m-cvqw-wrvh](https://github.com/advisories/GHSA-fq3m-cvqw-wrvh) |
| Published | August 28, 2026 |
| Severity | High — CVSS 3.1: 7.5 |
| Vector | `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` |
| Mainline commit | [`74b21f52c5c5a71a05c0ff70e513f4f04ff28b17`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) |
| File changed | `net/sctp/sm_make_chunk.c` |
| Upstream author credit | Charles Vosburgh |
| Upstream trailers | `Signed-off-by: Charles Vosburgh`, `Acked-by: Xin Long`, `Signed-off-by: Jakub Kicinski` |
| Mainline fix | Linux 7.2 |
| Verified stable branches | 7.1.y, 6.18.y, 6.12.y, 6.6.y, 6.1.y, 5.15.y, and 5.10.y |
| Security class | Type-specific length under-validation leading to a bounded out-of-parameter read |

## Background

SCTP INIT parameters are type-length-value structures. The generic parameter walker is responsible for making sure that a parameter header exists and that its declared length can be parsed. A concrete fixed-layout parameter can require more data than that minimum header, so its type-specific validator must also check the complete structure before a consumer reads any typed field.

The Adaptation Layer Indication parameter contains the generic four-byte SCTP parameter header followed by a 32-bit Adaptation Code Point. Mallory controls the parameter type, declared length, and bytes sent to the listening SCTP endpoint. The receiver must reject a declared length that does not cover both the header and the code point. Alice, the operator of the receiving kernel, should never have bytes outside Mallory's declared parameter interpreted as parameter contents or returned in protocol state.

The security invariant is therefore narrow: **a fixed-layout SCTP parameter must be validated against the size of its complete typed structure before code reads fields beyond the generic parameter header.**

## Vulnerability Details

### Vulnerable parser boundary

Before the patch, `sctp_verify_param()` in `net/sctp/sm_make_chunk.c` grouped Adaptation Layer Indication with parameter types that required no additional type-specific validation:

```c
case SCTP_PARAM_UNRECOGNIZED_PARAMETERS:
case SCTP_PARAM_ECN_CAPABLE:
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	break;
```

The generic walker could therefore accept a four-byte, header-only Adaptation Layer Indication. Later, `sctp_process_param()` treated the same pointer as a complete `struct sctp_adaptation_ind` and read the payload field:

```c
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	asoc->peer.adaptation_ind = ntohl(param.aind->adaptation_ind);
	break;
```

The two parser layers enforced different assumptions:

1. the generic walker accepted a parameter containing only the header; and
2. the type-specific consumer read a field beginning after that header.

When Mallory places the malformed parameter last in the INIT, the supposed `adaptation_ind` field begins immediately beyond the parameter at receive-buffer tailroom. The receiver then stores that 32-bit value in `asoc->peer.adaptation_ind`.

### Why the value becomes observable

The value does not remain solely inside the parser. SCTP association setup incorporates `peer.adaptation_ind` into the state cookie returned to the peer in the INIT ACK. Under an affected buffer-allocation profile, this creates an observable path from the four-byte over-read to Mallory's response.

| Boundary | Result |
|---|---|
| Mallory-controlled input | SCTP INIT parameter type and declared length |
| Existing check | Generic parameter-header/TLV validation |
| Missing check | Complete `struct sctp_adaptation_ind` size validation |
| Unsafe operation | Read of `adaptation_ind` beyond the declared parameter |
| Narrow observable consequence | Four-byte value propagated into the returned SCTP state cookie under an affected allocation profile |

The root cause is the use of generic TLV validity as a substitute for type validity. The public mainline commit records the upstream lineage as:

```text
Fixes: 1da177e4c3f4 ("Linux-2.6.12-rc2")
Cc: Linux stable
Signed-off-by: Charles Vosburgh
Acked-by: Xin Long
Signed-off-by: Jakub Kicinski
```

That `Fixes:` tag identifies the public source lineage. It does not establish that every release or downstream kernel derived from that lineage was independently inspected or dynamically tested.

## Exploitability Analysis

The verified primitive is a **bounded four-byte read immediately beyond a malformed Adaptation Layer Indication parameter**. It is remotely reachable during SCTP association setup without authentication or user interaction. Under the documented non-clearing allocation profile, the resulting value can cross the kernel-to-network confidentiality boundary by appearing in the returned state cookie.

The owned KVM/QEMU validation used `init_on_alloc=0` and observed non-zero four-byte values in a subset of malformed responses. A controlled A/B test primed the relevant tailroom with a known marker and observed that marker before the patch. The same malformed request stopped reaching the response path after the patch, while baseline and correctly sized Adaptation Indication controls remained functional.

An exact-default virtio control produced zero values across the corresponding sample. That result limits the deployment claim: source-level reachability does not guarantee observable non-zero disclosure on every allocation and receive path. Allocator clearing can mask the confidentiality consequence, but it does not make the out-of-parameter read valid.

The evidence does not establish any of the following stronger capabilities:

- arbitrary-address reads;
- attacker selection of a particular kernel address or secret;
- deterministic disclosure of a chosen value;
- a memory write;
- code execution; or
- local or remote privilege escalation.

## Proof of Concept

No executable network reproducer is published in this repository. The original research used three inputs against an owned KVM/QEMU guest so that parser failure could be separated from basic network or feature failure:

1. **Baseline INIT:** confirmed that the SCTP listener and packet path were working.
2. **Valid Adaptation Indication:** used the complete eight-byte parameter and confirmed that the supplied code point was preserved.
3. **Header-only Adaptation Indication:** used a four-byte parameter to exercise the missing type-specific size check.

The strongest recorded comparison kept the kernel base, configuration, guest setup, probe, and input shape constant while changing only the patch state. Before the fix, the malformed case could reflect a controlled four-byte receive-tail marker. After the fix, the malformed request was rejected before response construction, while the baseline and valid controls still worked.

These are preserved results from the documented controlled environment, not a new run performed for this write-up update. Reproduction requires an isolated disposable kernel guest and raw SCTP traffic generation. It should not be attempted against third-party systems, and a test guest should be discarded after use if its memory or network state is no longer trusted.

## Remediation

### Accepted upstream fix

Mainline commit [`74b21f52c5c5`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) separates Adaptation Layer Indication from the no-extra-validation cases and requires the declared length to match the complete typed structure:

```c
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	if (ntohs(param.p->length) != sizeof(*param.aind)) {
		sctp_process_inv_paramlength(asoc, param.p,
					     chunk, err_chunk);
		retval = SCTP_IERROR_ABORT;
	}
	break;
```

A malformed parameter now follows the existing invalid-parameter-length path, and association setup is aborted before `sctp_process_param()` can read the missing field.

### Fixed releases and stable backports

Linux 7.2 is the first final mainline release containing the fix. Seven stable branches were verified to carry the same correction:

| Stable branch | Verified stable commit | First verified stable release carrying the change |
|---|---|---|
| Linux 7.1.y | [`bfa28cf99eb4d096c87da939f54233444d209ca5`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=bfa28cf99eb4d096c87da939f54233444d209ca5) | 7.1.8 |
| Linux 6.18.y | [`17b412468c7a44f66a385bda48cdc1e94e39bd6d`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=17b412468c7a44f66a385bda48cdc1e94e39bd6d) | 6.18.44 |
| Linux 6.12.y | [`5fd7cfc708dfc988ae9920c21075e6121bc89926`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=5fd7cfc708dfc988ae9920c21075e6121bc89926) | 6.12.103 |
| Linux 6.6.y | [`93942b5772e0eee4147d4799cc1b936ae12fa615`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=93942b5772e0eee4147d4799cc1b936ae12fa615) | 6.6.151 |
| Linux 6.1.y | [`7b7e4e3640d57bd8857f0052c8b0d8ed4e5e954a`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=7b7e4e3640d57bd8857f0052c8b0d8ed4e5e954a) | 6.1.183 |
| Linux 5.15.y | [`4c92c601c061e5602db2edeea54fef74aa304027`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=4c92c601c061e5602db2edeea54fef74aa304027) | 5.15.216 |
| Linux 5.10.y | [`fa7861ddbe3b525b5d541c15c3953d3569e6eb0e`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/commit/?id=fa7861ddbe3b525b5d541c15c3953d3569e6eb0e) | 5.10.265 |

Kernel vendors and downstream distributions should verify the exact patch status of the branch they ship rather than infer it solely from a version number.

### Regression guidance

A durable regression test should keep all three parser cases together:

- a baseline INIT succeeds;
- a valid eight-byte Adaptation Indication succeeds and retains the supplied value; and
- a header-only Adaptation Indication is rejected before any typed payload read.

The malformed case should also be tested with clearing and non-clearing allocation profiles. Parser correctness must not depend on whether allocator initialization happens to make the invalid bytes appear as zero.

**Regression invariant:** No fixed-layout SCTP field may be read until the parameter length has been validated against the complete structure required by that parameter type.

## Summary

CVE-2026-80717 is a remotely reachable SCTP parameter-validation error. A header-only Adaptation Layer Indication passed generic TLV validation and reached code that read the absent 32-bit payload field. Under the validated non-clearing receive-buffer profile, that bounded value could be propagated into the state cookie returned to an unauthenticated peer. The evidence does not support arbitrary reads, writes, code execution, or privilege escalation.

The accepted upstream patch enforces the complete typed parameter length before the payload is accessed. It is included in Linux 7.2 and has been verified across seven stable branches, with the newest addition being Linux 7.1.y commit `bfa28cf99eb4` in release 7.1.8.

---

## Timeline

| Date / release | Event |
|---|---|
| July 2026 | Issue validated and patch prepared for Linux networking review. |
| July 2026 | Patch submitted upstream. |
| July 29, 2026 | Mainline commit `74b21f52c5c5` recorded in upstream Git history. |
| Linux 7.2 | First final mainline release containing the fix. |
| August 2026 | Stable inclusion verified across the listed 7.1, 6.18, 6.12, 6.6, 6.1, 5.15, and 5.10 releases. |
| August 28, 2026 | CVE-2026-80717 and global advisory GHSA-fq3m-cvqw-wrvh published. |

## Upstream References

- [CVE-2026-80717 record](https://www.cve.org/CVERecord?id=CVE-2026-80717)
- [GitHub global advisory — GHSA-fq3m-cvqw-wrvh](https://github.com/advisories/GHSA-fq3m-cvqw-wrvh)
- [Linux mainline commit — 74b21f52c5c5](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17)
- [Original patch submission](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/06549.html)
- [Maintainer review and `Acked-by`](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/08340.html)
- [Patchwork bot acceptance notice](https://lkml.iu.edu/hypermail/linux/kernel/2607.3/11779.html)

## Credit

**Charles Vosburgh (`the-vibe-dev`)** — finder, patch author, and `Signed-off-by` contributor.

The accepted mainline commit was acknowledged by **Xin Long** and signed into the networking tree by **Jakub Kicinski**.

**Broader lesson:** Protocol parsers should validate the structure they are about to dereference, not merely the smallest generic container that can hold it.
