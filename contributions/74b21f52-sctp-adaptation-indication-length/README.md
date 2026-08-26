# sctp: validate Adaptation Indication parameter length

**Linux mainline:** [`74b21f52c5c5a71a05c0ff70e513f4f04ff28b17`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17)  
**Subsystem:** `net/sctp`  
**Role:** **Patch author + `Signed-off-by`**  
**Author:** Charles Vosburgh (`the-vibe-dev`)

---

## At a glance

| Field | Value |
|---|---|
| Mainline commit | `74b21f52c5c5a71a05c0ff70e513f4f04ff28b17` |
| Commit title | `sctp: validate Adaptation Indication parameter length` |
| File changed | `net/sctp/sm_make_chunk.c` |
| Upstream author credit | Charles Vosburgh |
| Upstream trailers | `Signed-off-by: Charles Vosburgh`, `Acked-by: Xin Long`, `Signed-off-by: Jakub Kicinski` |
| Fixes lineage | `1da177e4c3f4 ("Linux-2.6.12-rc2")` |
| Mainline fix | Linux 7.2 |
| Verified stable releases carrying the change | 6.18.44, 6.12.103, 6.6.151, 6.1.183, 5.15.216, 5.10.265 |
| Security class | Type-specific length under-validation leading to a bounded out-of-parameter read |

## Summary

The SCTP Adaptation Layer Indication parameter has a fixed layout: a generic four-byte SCTP parameter header followed by a 32-bit Adaptation Code Point. Before this fix, `sctp_verify_param()` accepted a header-only Adaptation Indication because the generic parameter walker verified that the parameter header existed but did not enforce the size of the complete typed structure.

Later processing treated the same pointer as a full Adaptation Indication structure and read the missing `adaptation_ind` field. If the malformed parameter was the final parameter in an INIT, that read began immediately beyond the peer-declared parameter, at receive-buffer tailroom. The resulting 32-bit value was stored in SCTP association state and copied into the state cookie returned to the peer in the INIT ACK.

The accepted fix adds the missing type-specific size check and aborts malformed association setup through the kernel's existing invalid-parameter-length path.

**Security invariant:** A fixed-layout SCTP parameter must be validated against the size of its complete typed structure before code reads fields beyond the generic parameter header.

---

## The vulnerable boundary

SCTP INIT parameters are TLVs. Generic TLV validation can prove that a parameter header is present and internally parseable, but that does **not** prove that a fixed-layout parameter contains all fields required by its concrete type.

For Adaptation Layer Indication, the peer controls the declared parameter length. The receiver must not interpret bytes beyond that declared length as the required 32-bit Adaptation Code Point.

Before the patch, the relevant switch grouped Adaptation Layer Indication with parameters that required no additional type-specific validation:

```c
case SCTP_PARAM_UNRECOGNIZED_PARAMETERS:
case SCTP_PARAM_ECN_CAPABLE:
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	break;
```

Later, normal parameter processing assumed the full structure existed:

```c
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	asoc->peer.adaptation_ind = ntohl(param.aind->adaptation_ind);
	break;
```

That creates a mismatch between two parser layers:

1. the generic walker accepts a four-byte header; and
2. the type-specific consumer reads a field that begins after those four bytes.

When the malformed parameter is last in the INIT, the missing field is not attacker-supplied parameter data at all—it begins at the receive skb tail.

---

## Why the value becomes observable

The issue is more than a local out-of-bounds read inside the parser. The value read as `adaptation_ind` becomes peer association state and is incorporated into the SCTP state cookie returned in the INIT ACK.

That gives the remote peer an observable path for the four-byte value.

The demonstrated behavior was deliberately bounded. In an owned KVM/QEMU test environment using a non-clearing allocation profile (`init_on_alloc=0`), malformed requests produced non-zero four-byte values in a subset of responses. A controlled A/B test also primed the relevant tailroom with a known four-byte marker and observed that marker before the patch.

An exact-default virtio control produced zero values across the corresponding sample, which is an important limitation: the source bug is reachable regardless, but observable non-zero disclosure depends on receive-buffer/allocation behavior. This work therefore does **not** claim arbitrary-address reads, selected-secret extraction, a write primitive, code execution, or privilege escalation.

---

## Root cause

The generic SCTP parameter validator established only structural TLV validity. It did not enforce the minimum size of the concrete fixed-layout parameter before a later code path cast the pointer to `struct sctp_adaptation_ind` and dereferenced its payload field.

| Boundary | Result |
|---|---|
| Untrusted input | SCTP INIT parameter type and declared length |
| Existing check | Generic parameter-header/TLV validation |
| Missing check | Concrete `struct sctp_adaptation_ind` size validation |
| Unsafe use | Read of `adaptation_ind` beyond the declared parameter |
| Observable consequence | Four-byte value propagated into the returned SCTP state cookie under an affected allocation profile |

The broader parser lesson is simple: **generic TLV validity is not type validity**.

---

## Accepted upstream fix

Mainline commit [`74b21f52c5c5`](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17) separates Adaptation Layer Indication from the no-extra-validation cases and checks the complete structure size:

```c
case SCTP_PARAM_ADAPTATION_LAYER_IND:
	if (ntohs(param.p->length) != sizeof(*param.aind)) {
		sctp_process_inv_paramlength(asoc, param.p,
					     chunk, err_chunk);
		retval = SCTP_IERROR_ABORT;
	}
	break;
```

A malformed parameter now follows the existing invalid-parameter-length handling path and association processing is aborted before `sctp_process_param()` can read the missing field.

The public commit message records:

```text
Fixes: 1da177e4c3f4 ("Linux-2.6.12-rc2")
Cc: stable@vger.kernel.org
Signed-off-by: Charles Vosburgh <the-vibe-dev@users.noreply.github.com>
Acked-by: Xin Long <lucien.xin@gmail.com>
Signed-off-by: Jakub Kicinski <kuba@kernel.org>
```

GitHub's upstream mirror also attributes the commit author to [`the-vibe-dev`](https://github.com/the-vibe-dev).

---

## Validation approach

The research used three controls together rather than relying on a crash or source inspection alone:

### 1. Baseline INIT

A normal INIT establishes that the SCTP listener and packet path are working.

### 2. Valid Adaptation Indication

A correctly sized eight-byte parameter verifies that the intended feature continues to work and that the supplied code point is preserved.

### 3. Header-only malformed parameter

A four-byte header-only Adaptation Indication exercises the missing type-size boundary.

The strongest local A/B comparison used the same kernel base, configuration, guest setup, probe, and input shape with only the patch state changed. Before the patch, the malformed case could reflect the controlled four-byte tail marker. After the patch, the malformed request no longer reached the response path while the baseline and valid controls remained functional.

No executable reproducer is published in this repository.

---

## Impact and limits

The demonstrated primitive is a **bounded four-byte read immediately beyond the malformed parameter** that can be reflected to an unauthenticated SCTP peer through the state cookie under an affected non-clearing receive-buffer profile.

The following stronger claims are **not** supported by the validation and are intentionally excluded:

- arbitrary-address read;
- attacker-selected kernel memory disclosure;
- deterministic disclosure of a particular secret;
- memory write;
- code execution;
- local or remote privilege escalation.

The exact-default allocation control is also important: parser correctness cannot depend on allocator initialization masking an invalid access, but deployment-level confidentiality impact can vary with memory initialization behavior.

---

## Version and backport information

The accepted commit carries:

```text
Fixes: 1da177e4c3f4 ("Linux-2.6.12-rc2")
```

That is the public upstream vulnerable lineage. This write-up does not claim that every intervening release was independently executed.

Directly reviewed vulnerable source included Linux 7.1.4, Linux 7.2-rc4, and the pre-fix networking-tree base. The first final mainline release containing the fix is **Linux 7.2**.

The same change was independently tracked into the following stable releases during this research:

- Linux **6.18.44**
- Linux **6.12.103**
- Linux **6.6.151**
- Linux **6.1.183**
- Linux **5.15.216**
- Linux **5.10.265**

Kernel vendors and downstream distributions should still verify the exact patch status of the branch they ship rather than infer it only from version numbering.

---

## Regression guidance

A durable regression test should keep all three parser cases together:

- baseline INIT succeeds;
- a valid eight-byte Adaptation Indication succeeds and retains the supplied value;
- a header-only Adaptation Indication is rejected before any typed payload read.

Testing under both clearing and non-clearing allocation profiles is useful. The malformed input should be rejected for parser correctness regardless of whether the surrounding allocator would otherwise make the invalid bytes appear to be zero.

**Regression invariant:** No fixed-layout SCTP field may be read until the parameter length has been validated against the complete structure required by that parameter type.

---

## Timeline

| Date / release | Event |
|---|---|
| July 2026 | Issue validated and patch prepared for Linux networking review. |
| July 2026 | Patch submitted upstream. |
| July 29, 2026 | Commit `74b21f52c5c5` recorded in upstream Git history. |
| Linux 7.2 | First final mainline release containing the fix. |
| August 2026 | Stable inclusion verified across the listed 6.18, 6.12, 6.6, 6.1, 5.15, and 5.10 releases. |

---

## Upstream references

- [Linux mainline commit — 74b21f52c5c5](https://github.com/torvalds/linux/commit/74b21f52c5c5a71a05c0ff70e513f4f04ff28b17)
- [Patch submission / review thread](https://patch.msgid.link/20260727-sctp-adaptation-length-v1-1-0ab58b2810a5@gmail.com)

## Credit

**Charles Vosburgh (`the-vibe-dev`)** — patch author and `Signed-off-by` contributor.

The accepted mainline commit was acknowledged by **Xin Long** and signed into the networking tree by **Jakub Kicinski**.

## Research conclusion

The bug came from trusting generic TLV validity where the consumer required type-specific structural validity. A header-only SCTP Adaptation Indication could reach code that dereferenced the absent 32-bit payload field and propagate that value into response state. The accepted patch restores the correct parsing boundary by requiring the complete fixed structure before any typed access.

**Broader lesson:** Protocol parsers should validate the structure they are actually about to dereference—not merely the smallest generic container that can hold it.
