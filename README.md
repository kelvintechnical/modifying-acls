# Lab: Modifying ACLs — `setfacl -m` for named users and groups

- **Series:** linux-ops-mastery — RHCSA Permissions, Special Bits & ACLs
- **Subjects covered:** `setfacl -m` mutation syntax, `u:name:perms` and `g:name:perms`, combining multiple rows in one invocation, interaction with `umask` and base mode bits, verifying with `getfacl`, when `chmod` still matters vs when ACL rows dominate for non-owners
- **Career arcs covered:** RHCSA (collaboration directories on shared servers), RHCE (`ansible.posix.acl` mirrors these operations), SRE (break-glass grants without retagging primary groups), DevOps (release artifact folders with per-service readers), AI/MLOps (per-team read ACLs on evaluation metrics shares)
- **Prerequisite:** Labs 47–48 — capable filesystem + ability to read `getfacl`
- **Time Estimate:** 30 to 45 minutes
- **Difficulty arc:** Task 1 sandbox · 2–3 single-row grants · 4–5 combined user+group rows · 6 capstone multi-principal file

---

## Objective

`chmod` assigns one owner and one owning group. Real teams need **N humans** with different rights on the same file without creating **N Unix groups**. `setfacl -m` adds or replaces **named rows** while preserving the inode unless you ask for more destructive operations. By the end of this lab your hands know the muscle memory: `setfacl -m u:alice:rw file`, `getfacl` to verify, repeat for groups, undo with precision (preview Lab 53).

The capstone is a single artifact file with **three different principals** (owner, named user, named group) each with distinct effective rights you can prove.

> **Lab safety note:** Work under `/root/modify-acl-lab`. Create local users if your VM is pristine.

---

## Concept: `-m` Means "Merge Rows" — Not "Replace the World"

`setfacl -m` applies **modifications** to the ACL set. Unmentioned rows survive. That is different from wiping (`setfacl -b`, Lab 53) and different from replacing the entire ACL text at once (`setfacl --set`, advanced).

```
   Before:   user::rw-   group::r--   other::---
   Command:  setfacl -m u:alice:r--
   After:    user::rw-   user:alice:r--   group::r--   mask::r--   other::---
```

The kernel synthesizes `mask::` when needed so named rows have a ceiling.

> **Why this matters:** Junior admins assume `-m` replaces everything — then they panic when `group::` still shows old bits. Teach your fingers: **-m merges**.

---

## 📜 Why `setfacl` Exists — The Story

As Linux entered the enterprise file-and-print era of the late 1990s, administrators borrowed ACL semantics that had already proven themselves in proprietary Unix. The userland pair `getfacl`/`setfacl` became the stable text interface — easier to script than ioctl-based tools and clearer in email than raw `ls`.

RHEL integrated ACL-ready xfs and ext4 defaults so thoroughly that many engineers never learn the tools until the day a ticket says *"two departments share `/data/releases` but Legal may not write."* That day, `setfacl` stops being optional trivia and becomes the smallest change with the least blast radius.

> **The point of the story:** `setfacl` is how you express **relationship-shaped permissions** on a filesystem that still pretends to be POSIX.

---

## 👪 The `setfacl -m` Family — Who Lives There

### Principal letters

| Letter | Meaning | Example |
|---|---|---|
| `u` | User | `-m u:alice:r-x` |
| `g` | Group | `-m g:devs:rw-` |
| `m` | Mask | `-m m::r--` (Lab 52) |
| `o` | Other | `-m o::---` |

### Common forms

| Form | Effect |
|---|---|
| `-m u::rwx` | Replace **base owner** triple |
| `-m g::rwx` | Replace **owning group** triple |
| `-m o::rwx` | Replace **other** triple |
| `-m u:name:perms` | Add/replace **named user** row |

### Verification pair

| Reader | After every mutation |
|---|---|
| `getfacl` | Human truth |
| `ls -l` | Quick `+` indicator |

> **The point of the family tree:** `-m` accepts comma-separated clauses: `setfacl -m u:a:r,g:b:rx file`.

---

## 🔬 The Anatomy of `setfacl -m u:alice:rw- FILE` — In One Diagram

```
$ setfacl -m u:alice:rw- report.txt
  │        │  │    │  │    └── target inode
  │        │  │    │  └─ permission triplet (r w x position or `-`)
  │        │  │    └─ colon separator (required)
  │        │  └─ login name resolved via getpwnam(3)
  │        └─ principal type user
  └─ modify (merge) existing ACL set

Kernel flow (simplified):
  1. VFS permission check: caller needs CAP_FOWNER or ownership rules satisfied.
  2. ACL parser validates triplet characters.
  3. xattr backing store updates; ctime changes.
  4. `ls` begins showing `+` if not already.
```

> **Reading rule:** Perms are **positional**. `rw-` means read yes, write yes, execute no. Order matters.

---

## 📚 `setfacl -m` Reference Table

| Task | Command | Notes |
|---|---|---|
| Grant user read | `setfacl -m u:alice:r-- FILE` | |
| Grant group rx | `setfacl -m g:devs:r-x DIR` | Directory common |
| Chain updates | `setfacl -m u:a:rwx,g:b:r-- FILE` | One syscall batch |
| Copy ACL A → B | `getfacl A \| setfacl --set-file=- B` | Advanced pattern |
| Re-hit baseline mode | `chmod` on base bits | Still valid alongside ACLs |

> **Rule one of modification:** After every `setfacl`, run `getfacl`. No exceptions on exams.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | "Grant user X rw on file Y" is a literal exam verb family. |
| **RHCE candidate** | Maps 1:1 to `ansible.posix.acl` state=present etype/entity/permissions. |
| **SRE / Platform** | Smallest grant for break-glass read during incidents. |
| **DevOps** | GitOps repos rarely model ACLs — imperative `setfacl` still fills gaps. |
| **AI / MLOps** | Fine-grained read on evaluation logs without duplicating data. |

---

## 🔧 The 6 Tasks

---

### Task 1 — Lab tree and users

**Purpose:** Prepare principals.

```bash
sudo -i
mkdir -p /root/modify-acl-lab
cd /root/modify-acl-lab
id alice &>/dev/null || useradd alice
id bob &>/dev/null || useradd bob
getent group devs >/dev/null || groupadd devs
usermod -aG devs bob 2>/dev/null || true
echo 'quarterly' > revenue.txt
chmod 640 revenue.txt
ls -l revenue.txt
```

**Human-Readable Breakdown:** Create file `revenue.txt` with tight base perms so ACL differences are obvious later.

**Reading it left to right:** `groupadd devs` creates a true secondary group. `usermod -aG` adds bob to devs without removing other groups.

**The story:** Shared finance-ish filenames make the mental model sticky — replace with boring `file1.txt` if you prefer.

**Expected output:**

```text
-rw-r----- 1 root root 10 May 26 10:05 revenue.txt
```

**Switches**

| Token | Meaning |
|---|---|
| `chmod 640` | Base tight perms |
| `usermod -aG devs bob` | Append group membership |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `groupadd: group 'devs' exists` | Safe — continue |

---

### Task 2 — Grant Alice read-write via named user row

**Purpose:** Classic `u:name:perms` pattern.

```bash
setfacl -m u:alice:rw- revenue.txt
ls -l revenue.txt
getfacl revenue.txt
```

**Human-Readable Breakdown:** Alice is not owner — she needs a named user row to reach the data despite `640` group being `root`.

**Reading it left to right:** Named user row intersects future mask (auto here).

**The story:** This is the bread-and-butter RHCSA grant.

**Expected output:**

```text
-rw-rw----+ 1 root root 10 May 26 10:05 revenue.txt
user:alice:rw-
mask::rw-
```

**Switches**

| Token | Meaning |
|---|---|
| `-m u:alice:rw-` | Named user merge |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Operation not supported | Return to Lab 47 mount checks |

---

### Task 3 — Grant `devs` group read-only while preserving Alice

**Purpose:** Named **group** row independent of owning group.

```bash
setfacl -m g:devs:r-- revenue.txt
getfacl -c revenue.txt
```

**Human-Readable Breakdown:** `g:devs:r--` lets any process with effective GID `devs` read — bob qualifies after `newgrp devs` or login refresh, but `runuser` tests are crisper (Task 4).

**Reading it left to right:** Two named rows coexist; mask widens to cover max needed bits.

**The story:** This models "whole department may read, only Alice may write" **if** mask and other rows cooperate — tune in Task 5 if you tighten.

**Expected output:**

```text
user::rw-
user:alice:rw-
group::r--
group:devs:r--
mask::rw-
other::---
```

**Switches**

| Token | Meaning |
|---|---|
| `-m g:devs:r--` | Named group entry |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Invalid argument` on group name | Typo — `getent group devs` |

---

### Task 4 — Verify effective access as Bob using `runuser`

**Purpose:** Prove ACLs, not theory.

```bash
runuser -u bob -- cat /root/modify-acl-lab/revenue.txt
runuser -u alice -- head -n1 /root/modify-acl-lab/revenue.txt
```

**Human-Readable Breakdown:** Bob should read via `group:devs`. Alice should still read.

**Reading it left to right:** `runuser` (from `util-linux`) runs a single command under target user without full login overhead.

**The story:** If this fails, you debug ACL — not cable ghosts.

**Expected output:**

```text
quarterly
quarterly
```

**Switches**

| Token | Meaning |
|---|---|
| `runuser -u bob -- cmd` | Run cmd as bob |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Permission denied for bob | Confirm `g:devs` row and bob membership: `id bob` |

---

### Task 5 — Tighten **other** and confirm nothing leaked

**Purpose:** Practice base triplet merge via `setfacl -m o::---`.

```bash
setfacl -m o::--- revenue.txt
getfacl revenue.txt
runuser -u nobody -- cat /root/modify-acl-lab/revenue.txt
```

**Human-Readable Breakdown:** `nobody` should fail unless your distro maps odd supplemental groups — expect failure.

**Reading it left to right:** `o::---` closes world ACL path; named rows unaffected.

**The story:** Defense-in-depth: even if someone later mis-chmods directory traversal, other should stay dead on sensitive files.

**Expected output:**

```text
cat: ... Permission denied
```

**Switches**

| Token | Meaning |
|---|---|
| `-m o::---` | Strip other bits |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| nobody can still read | They may match group via supplemental GIDs — test a dedicated user |

---

### Task 6 — Capstone: multi-principal write proof + cleanup

**Purpose:** Alice writes append via ACL without ownership change.

```bash
runuser -u alice -- bash -lc 'echo extra >> /root/modify-acl-lab/revenue.txt'
wc -l /root/modify-acl-lab/revenue.txt
getfacl /root/modify-acl-lab/revenue.txt
```

**Human-Readable Breakdown:** Demonstrate write path worked for non-owner.

**The story:** If append fails, mask or directory write bit on parent directory is the usual culprit — debug with `namei -l`.

**Expected output:**

```text
2 /root/modify-acl-lab/revenue.txt
user:alice:rw-
...
```

**Cleanup**

```bash
rm -rf /root/modify-acl-lab
userdel -r alice 2>/dev/null || true
userdel -r bob 2>/dev/null || true
groupdel devs 2>/dev/null || true
exit
```

**Switches**

| Token | Meaning |
|---|---|
| `bash -lc` | Login-style shell for redirection quoting |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Append permission denied | Check parent dir ACL / perms: `getfacl ..` |

---

## 🔍 Modifying ACL Decision Guide

```
Need to change rights?
  │
  ├── "Everyone in one Unix group shares same rights"
  │       └── chmod/chgrp may suffice
  │
  ├── "Two humans, different rights, same file"
  │       └── setfacl -m u:A:perms,u:B:perms
  │
  ├── "Whole AD/LDAP group without local /etc/group line"
  │       └── sssd realm join lab (outside scope) — local group here
  │
  └── "Undo a mistake"
          └── Lab 53 (-x / -b)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Tree + users + group + baseline file
- [ ] 02 `u:alice:rw-`
- [ ] 03 `g:devs:r--` and `getfacl -c`
- [ ] 04 `runuser` proof for bob + alice
- [ ] 05 `o::---` tighten + nobody test
- [ ] 06 Alice append capstone, cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Typos in `u:name` | Invalid argument | `id name` |
| Expecting `-m` to delete rows | Stale rows linger | Use `-x` (Lab 53) |
| Missing parent directory execute | Cannot traverse | `namei -l path` |
| SELinux type wrong | AVC denials | `ausearch -m avc` |
| Confusing owning group vs named group | Wrong effective access | `getfacl` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize `setfacl -m u:USER:PERMS FILE` and always pair with `getfacl`.

**RHCE candidate**
- Map etype: `user` / `group`, entity: name, permissions: symbolic.

**SRE / Platform interview**
- Narrate merge semantics vs `chmod` overwrite semantics.

**DevOps**
- Wrap grants in idempotent scripts: check `getfacl` first, apply only if drift.

**AI / MLOps**
- ACL grants on metrics shares are reversible artifacts — log the `setfacl` line in change management.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 48 — Viewing ACLs | Reader complement |
| Lab 50 — Deny patterns | Contrasting grant vs effective deny |
| Lab 52 — ACL Masks | Ceiling math |
| Lab 53 — Removing ACLs | teardown |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
