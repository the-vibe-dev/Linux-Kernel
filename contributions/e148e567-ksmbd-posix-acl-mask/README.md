# ksmbd: preserve VFS inherited POSIX ACL mask

**Linux mainline:** [`e148e567a9252643baa125cb65d7ae9c2c6cf68a`](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a)  
**Subsystem:** `fs/smb/server` / KSMBD  
**Role:** **`Reported-by`**  
**Reporter:** Charles Vosburgh (`the-vibe-dev`)  
**Patch author:** Namjae Jeon

---

## At a glance

| Field | Value |
|---|---|
| Mainline commit | `e148e567a9252643baa125cb65d7ae9c2c6cf68a` |
| Commit title | `ksmbd: preserve VFS inherited POSIX ACL mask` |
| File changed | `fs/smb/server/vfs.c` |
| Public credit | `Reported-by: Charles Vosburgh` |
| Patch author | Namjae Jeon |
| Upstream integration | Signed off by Steve French |
| First verified mainline tag containing fix | `v7.2-rc5` |
| Final mainline release containing fix | Linux 7.2 |
| Security class | Incorrect permission assignment / post-create ACL widening |

## Summary

KSMBD previously performed its own POSIX ACL inheritance work **after** the VFS had already created a child inode and computed the child's access/default ACLs from the parent ACL and requested creation mode.

The KSMBD helper reloaded the parent directory's default ACL, found the `ACL_MASK` entry, changed that mask to full `rwx`, and then installed the resulting ACL on the newly created object. That could silently broaden effective permissions that the VFS had intentionally narrowed.

In a configuration with a named user or group entry whose recorded permissions were constrained by a restrictive ACL mask, a new object created through SMB could therefore grant effective access that the equivalent VFS inheritance result would not have granted. For directories, the widened ACL could also become the new default ACL and propagate to later descendants.

The accepted mainline patch removes the KSMBD post-create rewrite and preserves the ACL state already computed by the VFS.

**Security invariant:** KSMBD must preserve the child ACL and effective mask produced by the VFS; it must not reload the parent ACL after creation and broaden the result.

---

## POSIX ACL background

POSIX ACLs can express permissions for named users and groups in addition to the traditional owner/group/other mode bits. The `ACL_MASK` entry is the effective permission ceiling applied to named-user, named-group, and owning-group entries.

For example, an ACL may contain a named entry that records `rwx` while a restrictive mask removes those permissions in practice:

```text
user:someuser:rwx
mask::---
```

The VFS inheritance path takes the parent's default ACL and the requested file mode into account when constructing a new child. The resulting child ACL is the security result that downstream filesystem/server code should preserve.

---

## Vulnerable KSMBD behavior

Before the accepted patch, `ksmbd_vfs_inherit_posix_acl()` obtained the parent default ACL and then modified its mask entry:

```c
acls = get_inode_acl(parent_inode, ACL_TYPE_DEFAULT);
if (IS_ERR_OR_NULL(acls))
	return -ENOENT;
pace = acls->a_entries;

for (i = 0; i < acls->a_count; i++, pace++) {
	if (pace->e_tag == ACL_MASK) {
		pace->e_perm = 0x07;
		break;
	}
}
```

`0x07` is full `rwx` permission.

The helper then installed that ACL on the child as its access ACL and, for directories, as its default ACL:

```c
rc = set_posix_acl(idmap, dentry, ACL_TYPE_ACCESS, acls);

if (S_ISDIR(inode->i_mode))
	rc = set_posix_acl(idmap, dentry, ACL_TYPE_DEFAULT, acls);
```

The problem is not merely that KSMBD handled ACLs. The problem is that this happened **after the VFS had already performed inheritance and mode-based normalization**. KSMBD effectively discarded the VFS-computed effective mask and replaced it with a broader one.

---

## Security consequence

The bounded research scenario uses two distinct authenticated principals:

1. a creator who is allowed to create a child through the SMB share; and
2. another named principal whose ACL entry exists but is intentionally suppressed by a restrictive mask.

With ordinary local/VFS creation under the equivalent parent ACL, the restrictive mask remains effective and the second principal is denied.

With the vulnerable KSMBD post-create rewrite, the child ACL can instead acquire `mask::rwx`, activating permissions that were previously recorded but ineffective. The named principal can then gain read and/or write access to the new object.

For an SMB-created directory, the widened ACL can also be installed as the directory's default ACL, allowing the broader permissions to flow into subsequently created descendants.

The validated scope is intentionally narrow: this is a permission-widening problem for affected newly created objects and descendants. It is not evidence of arbitrary filesystem path access, kernel code execution, UID escalation, or modification of unrelated pre-existing objects.

---

## Root cause

The VFS had already produced the authoritative child ACL. KSMBD then re-read the **parent** default ACL and transformed it as if inheritance still needed to happen.

| Boundary | Result |
|---|---|
| Administrator policy | Parent default POSIX ACL with restrictive mask |
| Trusted kernel result | VFS-computed child access/default ACL |
| KSMBD error | Post-create parent ACL reload + forced `ACL_MASK = rwx` |
| Consequence | Effective permissions on new SMB-created objects can be broader than intended |

This is a good example of a post-processing bug: the initial security mechanism works, but a later subsystem helper overwrites the correct result.

---

## Accepted upstream fix

Mainline commit [`e148e567a925`](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a) removes the mask mutation and the subsequent `set_posix_acl()` calls:

```diff
-	pace = acls->a_entries;
-
-	for (i = 0; i < acls->a_count; i++, pace++) {
-		if (pace->e_tag == ACL_MASK) {
-			pace->e_perm = 0x07;
-			break;
-		}
-	}
-
-	rc = set_posix_acl(idmap, dentry, ACL_TYPE_ACCESS, acls);
-	...
+	posix_acl_release(acls);
+	return 0;
```

The fixed helper still detects whether the parent has a default ACL, but it no longer re-applies or broadens that ACL after the VFS creation path has done the correct inheritance work.

The upstream commit message states these credit roles:

```text
The VFS initializes a child's POSIX ACL from the parent's default ACL and
the requested creation mode. Do not mutate the parent ACL or overwrite the
child's VFS-computed access and default ACLs afterwards.

This preserves restrictive ACL_MASK entries and prevents SMB object creation
from widening effective permissions.

Reported-by: Charles Vosburgh
Signed-off-by: Namjae Jeon
Signed-off-by: Steve French
```

That credit is important: **Charles Vosburgh reported the issue; Namjae Jeon authored the accepted patch.**

---

## Validation approach

The original research used an owned disposable Linux/KSMBD environment and compared the behavior of local creation with SMB creation under the same restrictive default ACL.

A useful control matrix is:

### Local/VFS control

Create an object locally beneath the parent directory and verify that the named principal remains restricted by the inherited ACL mask.

### SMB creation path

Create the equivalent object through KSMBD and inspect the resulting access ACL. On the vulnerable behavior, the effective mask is widened.

### Separate-principal access

Use a second mapped principal whose named ACL entry is constrained by the parent's mask. Confirm that the local control remains denied while the vulnerable SMB-created object becomes readable/writable.

### Directory propagation

Repeat with a directory and a descendant to establish whether the broadened default ACL propagates beyond the directly created directory.

No executable private harness or raw lab evidence is included in this repository.

---

## Impact and limits

The demonstrated impact is a **cross-principal confidentiality and integrity boundary failure** for affected newly created objects.

Practical prerequisites include:

- POSIX ACL support on the backing filesystem;
- a parent default ACL containing a named-user or named-group entry broader than its effective mask;
- creation of a new object through KSMBD; and
- another principal whose access is supposed to remain constrained by that mask.

The research does **not** support claims of:

- access to arbitrary paths outside the share;
- access to unrelated pre-existing objects;
- kernel memory corruption;
- command execution;
- local root or UID escalation;
- universal impact on KSMBD deployments without the relevant ACL configuration.

The upstream commit itself does not assign a CVE or CVSS score. This repository therefore describes the technical boundary and observed effect without inventing a public severity label.

---

## Version information

The accepted commit fixes the pre-existing behavior but does not publish a complete affected-version range.

The original validation reproduced the behavior on Linux **v7.1.3**. Public tag containment checks showed the patch absent from **v7.2-rc4**, present in **v7.2-rc5**, and present in final **Linux 7.2**.

The exact introducing commit was not independently established during the bounded review, so this write-up does not expand the affected range beyond directly reviewed evidence and the upstream fix history.

The patch was selected for a public AUTOSEL series covering Linux 6.18 through 6.6. That is a queued selection, not proof of completed stable-tree backports. No distinct stable-branch cherry-pick IDs were verified during the September 5, 2026 re-check.

---

## Regression guidance

A strong KSMBD regression test should:

1. configure a parent directory with a restrictive default ACL mask and a named principal;
2. create equivalent children locally and through SMB;
3. compare the resulting ACLs and effective permissions;
4. test both files and directories;
5. test a descendant beneath an SMB-created directory; and
6. verify that KSMBD does not mutate the parent cached/on-disk ACL as part of child creation.

**Regression invariant:** The ACL attached to a new inode after VFS creation is authoritative unless a later operation has explicit, policy-backed reason to change it. KSMBD inheritance code must not widen that result.

---

## Timeline

| Date / release | Event |
|---|---|
| July 2026 | Issue reported by Charles Vosburgh to the KSMBD maintainers. |
| July 17, 2026 | Fix authored by Namjae Jeon. |
| July 22, 2026 | Commit `e148e567a925` recorded in upstream history. |
| Linux 7.2-rc5 | First verified mainline tag containing the fix. |
| Linux 7.2 | Final mainline release containing the accepted change. |

---

## Upstream references

- [Linux mainline commit — e148e567a925](https://github.com/torvalds/linux/commit/e148e567a9252643baa125cb65d7ae9c2c6cf68a)
- [git.kernel.org commit view](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e148e567a9252643baa125cb65d7ae9c2c6cf68a)
- [AUTOSEL selection thread](https://lkml.iu.edu/2608.3/13998.html)

## Credit

**Charles Vosburgh (`the-vibe-dev`)** — reporter, credited by the accepted upstream commit's `Reported-by` trailer.

**Namjae Jeon** — patch author.  
**Steve French** — upstream sign-off / SMB tree integration.

## Research conclusion

KSMBD's former post-create ACL helper reloaded the parent default ACL, forced the mask to full permissions, and overwrote the ACL state the VFS had already computed for the child. The accepted patch removes that second inheritance pass and preserves the VFS security result.

**Broader lesson:** Filesystem server code should not second-guess a completed VFS permission decision. Reapplying inheritance after creation can silently transform a correct restrictive ACL into a broader one.
