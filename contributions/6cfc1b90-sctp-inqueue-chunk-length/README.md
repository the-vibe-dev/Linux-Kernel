# sctp: validate chunk length in the inqueue parser

**Linux mainline:** [`6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a`](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)

**CVE:** [CVE-2026-99128](https://www.cve.org/CVERecord?id=CVE-2026-99128) — published October 28, 2026; public record updated October 30, 2026.

**Subsystem:** `net/sctp`

**Role:** **Patch author + `Signed-off-by`**

**Author:** Charles Vosburgh (`the-vibe-dev`)

## At a glance

| Field | Value |
|---|---|
| Mainline commit | `6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a` |
| Commit title | `sctp: validate chunk length in the inqueue parser` |
| File changed | `net/sctp/inqueue.c` |
| Upstream author credit | Charles Vosburgh |
| Upstream trailers | `Signed-off-by: Charles Vosburgh`; `Acked-by: Xin Long`; `Signed-off-by: Jakub Kicinski` |
| Fixes lineage | `bbd0d59809f9` — SCTP AUTH receive and verification |
| Accepted status | Present in `torvalds/linux`; first fixed mainline release Linux 7.3 |
| Stable status | Seven released backports verified as of November 2, 2026; see the release table below |
| CVE status | CVE-2026-99128, published by the Linux CNA on October 28, 2026 |

## Executive summary

Every SCTP chunk includes a four-byte generic header, but the shared inqueue parser previously accepted a declared chunk length smaller than that header. A zero-length chunk left the computed chunk end at the current header.

Under the SCTP-AUTH processing path documented by the public commit, an ASCONF chunk could continue before the state machine's later length validation. The inqueue then returned the same malformed chunk repeatedly, keeping receive softirq processing busy and causing a soft lockup.

The accepted patch rejects undersized chunks at the shared parser boundary and marks the packet for discard before either caller can continue.

**Security invariant:** A parser must validate the minimum size of the generic structure it has already consumed before computing the next record boundary or returning that record to any caller.

## Security boundary

`sctp_inq_pop()` is shared parser infrastructure. Callers rely on it to advance the packet cursor and return a chunk whose declared extent is internally coherent. If a malformed zero-length chunk is returned without advancing the effective boundary, a caller that continues processing can receive the same chunk indefinitely.

The vulnerable route described upstream requires:

- a kernel built with `CONFIG_IP_SCTP`;
- an established SCTP association;
- `net.sctp.addip_enable=1`; and
- `net.sctp.auth_enable=1`.

ASCONF must be covered by the association's SCTP-AUTH policy. The public commit and CVE record state that the reproducer required neither application credentials nor a shared SCTP AUTH key, and did not require `net.sctp.addip_noauth_enable=1`. Association establishment is a prerequisite; possession of an AUTH key is not.

The public commit demonstrates availability impact in a controlled KVM guest. It does not claim memory corruption, information disclosure, code execution, or privilege escalation.

## Vulnerability details

Before the fix, the parser computed `chunk_end`, consumed the generic header from the socket buffer, and continued without first proving that the peer-declared chunk length was at least `sizeof(struct sctp_chunkhdr)`.

The accepted change adds the minimum-length check before singleton/packet-boundary handling:

```c
if (unlikely(ntohs(ch->length) < sizeof(*ch))) {
    chunk->pdiscard = 1;
} else if (chunk->chunk_end + sizeof(*ch) <=
           skb_tail_pointer(chunk->skb)) {
    /* This is not a singleton */
    chunk->singleton = 0;
```

This preserves the valid four-byte minimum while rejecting declared lengths zero through three. Marking the packet for discard prevents the malformed chunk from being returned repeatedly through the continuing AUTH/ASCONF path.

## Impact

The public commit records repeated watchdog soft-lockup reports and loss of post-trigger health probes in a two-vCPU KVM guest before the patch. With the patch applied, the same post-trigger probe set remained healthy and no equivalent soft-lockup signature appeared.

The demonstrated capability is remote denial of service against a specifically configured SCTP endpoint after association establishment. Deployment exposure depends on SCTP availability and the stated ADD-IP/AUTH configuration.

## Affected and fixed versions

The commit carries:

```text
Fixes: bbd0d59809f9 ("[SCTP]: Implement the receive and verification of AUTH chunk")
```

That change introduced SCTP AUTH receive and verification, including the caller's early continuation for chunks requiring authentication. The [Linux CNA record](https://raw.githubusercontent.com/CVEProject/cvelistV5/main/cves/2026/99xxx/CVE-2026-99128.json) identifies Linux 2.6.24 as the first affected release and Linux 7.3 as the first mainline release containing the fix. The Linux 2.6.24 parser source lacks the minimum-length guard; Linux 7.3 contains it.

The following seven stable fixes are listed in the updated CNA record. Each linked released source snapshot contains the minimum-length guard and packet-discard assignment, and each linked backport contains the same fix as mainline.

| Stable series | First fixed release | Backport commit |
|---|---|---|
| 5.10 | [5.10.273](https://github.com/gregkh/linux/blob/v5.10.273/net/sctp/inqueue.c) | [`cc4a4f1d6b6d`](https://git.kernel.org/stable/c/cc4a4f1d6b6d725438dc37b97f03d95ed9feb63f) |
| 5.15 | [5.15.224](https://github.com/gregkh/linux/blob/v5.15.224/net/sctp/inqueue.c) | [`b0dbd53b2426`](https://git.kernel.org/stable/c/b0dbd53b242609c88e81b0a49093bdbef5d5a418) |
| 6.1 | [6.1.191](https://github.com/gregkh/linux/blob/v6.1.191/net/sctp/inqueue.c) | [`d28ea1a62e27`](https://git.kernel.org/stable/c/d28ea1a62e27210d8ea3cf496dfae85dbdd7b84f) |
| 6.6 | [6.6.161](https://github.com/gregkh/linux/blob/v6.6.161/net/sctp/inqueue.c) | [`5a7ee52589da`](https://git.kernel.org/stable/c/5a7ee52589dae36c93dc64d7a97c5ded84618572) |
| 6.12 | [6.12.115](https://github.com/gregkh/linux/blob/v6.12.115/net/sctp/inqueue.c) | [`e0fb7fe18048`](https://git.kernel.org/stable/c/e0fb7fe180488a5afa2e91509ea838e8df8cb771) |
| 6.18 | [6.18.59](https://github.com/gregkh/linux/blob/v6.18.59/net/sctp/inqueue.c) | [`40507f59f230`](https://git.kernel.org/stable/c/40507f59f230dcf60ae13cb48d34f8c6e9f40659) |
| 7.2 | [7.2.13](https://github.com/gregkh/linux/blob/v7.2.13/net/sctp/inqueue.c) | [`9ac57e4c62e9`](https://git.kernel.org/stable/c/9ac57e4c62e93c287b36080e5da3645690549990) |

The immediately preceding releases—5.10.272, 5.15.223, 6.1.190, 6.6.160, 6.12.114, 6.18.58, and 7.2.12—lack this parser guard. These checks establish the released fix boundaries, rather than relying on stable-review announcements. Intermediate patch releases and other branch tips were not individually inspected or executed; distribution kernels may carry their own backports.

## Remediation

Upgrade to Linux 7.3 or a fixed stable release listed above, or apply a vendor patch containing the same minimum-length validation. Regression tests should cover declared chunk lengths zero through four, normal kernel-generated ASCONF traffic, both IPv4 and IPv6 SCTP, and the caller path that can continue before later state-machine validation.

## Validation approach

The public commit documents pre-trigger health probes, a malformed zero-length ASCONF input, post-trigger health probes, watchdog observations, and a patched differential. It also records healthy controls for declared lengths one through four and successful SCTP self-tests after the fix.

The November 2, 2026 refresh checked the public CNA record, introducing and fixing patches, and the released source snapshots described above. It did not rerun the trigger or repeat the runtime tests; the runtime observations remain those documented in the public upstream commit.

No private reproducer, packet fixture, or non-public validation artifact is included in this repository.

## Disclosure / upstream status

Jakub Kicinski applied the patch to the networking tree on August 30, 2026. The commit is now present in `torvalds/linux`, carries Charles's author and `Signed-off-by` credit, and includes Xin Long's `Acked-by` review.

The Linux CNA published CVE-2026-99128 on October 28, 2026 and updated its record on October 30 to include the seven stable backports listed above. Those backports preserve Charles's authorship and sign-off, Xin Long's acknowledgment, and Jakub Kicinski's sign-off, with an additional stable-maintainer sign-off from Greg Kroah-Hartman.

## Credit

**Charles Vosburgh (`the-vibe-dev`)** — patch author and `Signed-off-by` contributor.

**Xin Long** — `Acked-by` reviewer.

**Jakub Kicinski** — networking maintainer sign-off and integration.

## Public references

- [CVE-2026-99128 public record](https://www.cve.org/CVERecord?id=CVE-2026-99128)
- [Linux CNA record and fixed-version mappings](https://raw.githubusercontent.com/CVEProject/cvelistV5/main/cves/2026/99xxx/CVE-2026-99128.json)
- [SCTP AUTH introducing change](https://github.com/torvalds/linux/commit/bbd0d59809f923ea2b540cbd781b32110e249f6e)
- [Linux 2.6.24 parser source](https://github.com/torvalds/linux/blob/v2.6.24/net/sctp/inqueue.c)
- [Linux 7.3 fixed parser source](https://github.com/torvalds/linux/blob/v7.3/net/sctp/inqueue.c)
- [Linux mainline commit](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)
- [git.kernel.org commit view](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)
- [Public mailing-list review](https://lists.openwall.net/linux-kernel/2026/08/29/754)

## Responsible use

Use SCTP test traffic only in an isolated lab or against systems you are explicitly authorized to assess.
