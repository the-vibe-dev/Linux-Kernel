# sctp: validate chunk length in the inqueue parser

**Linux mainline:** [`6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a`](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)

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
| Accepted status | Present in `torvalds/linux` mainline |
| Stable status | No distinct stable-tree backport verified as of September 5, 2026 |
| CVE status | No public CVE assignment verified as of September 5, 2026 |

## Executive summary

Every SCTP chunk includes a four-byte generic header, but the shared inqueue parser previously accepted a declared chunk length smaller than that header. A zero-length chunk left the computed chunk end at the current header.

Under the SCTP-AUTH processing path documented by the public commit, an ASCONF chunk could continue before the state machine's later length validation. The inqueue then returned the same malformed chunk repeatedly, keeping receive softirq processing busy and causing a soft lockup.

The accepted patch rejects undersized chunks at the shared parser boundary and marks the packet for discard before either caller can continue.

**Security invariant:** A parser must validate the minimum size of the generic structure it has already consumed before computing the next record boundary or returning that record to any caller.

## Security boundary

`sctp_inq_pop()` is shared parser infrastructure. Callers rely on it to advance the packet cursor and return a chunk whose declared extent is internally coherent. If a malformed zero-length chunk is returned without advancing the effective boundary, a caller that continues processing can receive the same chunk indefinitely.

The vulnerable route described upstream requires:

- a kernel built with SCTP support;
- an established SCTP association;
- `net.sctp.addip_enable=1`; and
- `net.sctp.auth_enable=1`.

The public commit demonstrates availability impact in a controlled KVM guest. It does not claim memory corruption, information disclosure, code execution, or privilege escalation.

## Vulnerability details

Before the fix, the parser computed `chunk_end`, consumed the generic header from the socket buffer, and continued without first proving that the peer-declared chunk length was at least `sizeof(struct sctp_chunkhdr)`.

The accepted change adds the minimum-length check before singleton/packet-boundary handling:

```c
if (unlikely(ntohs(ch->length) < sizeof(*ch))) {
    chunk->pdiscard = 1;
} else if (...) {
    /* normal packet-boundary handling */
}
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

That trailer defines the public upstream lineage. This write-up does not claim every intervening release was independently executed. The fix is mainline commit `6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a`.

No public CVE or distinct stable-tree cherry-pick was found during the September 5, 2026 re-check.

## Remediation

Apply the mainline fix or a vendor patch containing the same minimum-length validation. Regression tests should cover declared chunk lengths zero through four, normal kernel-generated ASCONF traffic, both IPv4 and IPv6 SCTP, and the caller path that can continue before later state-machine validation.

## Validation approach

The public commit documents pre-trigger health probes, a malformed zero-length ASCONF input, post-trigger health probes, watchdog observations, and a patched differential. It also records healthy controls for declared lengths one through four and successful SCTP self-tests after the fix.

No private reproducer, packet fixture, or non-public validation artifact is included in this repository.

## Disclosure / upstream status

Jakub Kicinski applied the patch to the networking tree on August 30, 2026. The commit is now present in `torvalds/linux`, carries Charles's author and `Signed-off-by` credit, and includes Xin Long's `Acked-by` review.

## Credit

**Charles Vosburgh (`the-vibe-dev`)** — patch author and `Signed-off-by` contributor.

**Xin Long** — `Acked-by` reviewer.

**Jakub Kicinski** — networking maintainer sign-off and integration.

## Public references

- [Linux mainline commit](https://github.com/torvalds/linux/commit/6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)
- [git.kernel.org commit view](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6cfc1b90cb86f4aabc69fb8e30128e07e2cdfa3a)
- [Public mailing-list review](https://lists.openwall.net/linux-kernel/2026/08/29/754)

## Responsible use

Use SCTP test traffic only in an isolated lab or against systems you are explicitly authorized to assess.
