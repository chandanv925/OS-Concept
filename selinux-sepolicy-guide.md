# SELinux & Android sepolicy — A From-Scratch Guide

A structured path from "what even is this" to "I can read and write a basic Android sepolicy rule." Written in plain language, with the jargon explained the first time it shows up.

---

## Table of Contents

1. [The problem SELinux solves](#1-the-problem-selinux-solves)
2. [DAC vs MAC — the core idea](#2-dac-vs-mac--the-core-idea)
3. [How SELinux actually works under the hood](#3-how-selinux-actually-works-under-the-hood)
4. [The building blocks: types, domains, classes, permissions](#4-the-building-blocks-types-domains-classes-permissions)
5. [Reading and writing a policy rule](#5-reading-and-writing-a-policy-rule)
6. [Attributes and macros (writing less repetitive policy)](#6-attributes-and-macros-writing-less-repetitive-policy)
7. [The security context / label](#7-the-security-context--label)
8. [Enforcement modes: permissive vs enforcing](#8-enforcement-modes-permissive-vs-enforcing)
9. [Android's history with SELinux](#9-androids-history-with-selinux)
10. [What Android deliberately does NOT use](#10-what-android-deliberately-does-not-use)
11. [How sepolicy is organized in the Android source tree](#11-how-sepolicy-is-organized-in-the-android-source-tree)
12. [Treble: why vendor and system policy are split](#12-treble-why-vendor-and-system-policy-are-split)
13. [The tools you'll actually use day-to-day](#13-the-tools-youll-actually-use-day-to-day)
14. [Debugging a denial, step by step](#14-debugging-a-denial-step-by-step)
15. [A worked example, start to finish](#15-a-worked-example-start-to-finish)
16. [Cheat sheet](#16-cheat-sheet)
17. [Where to go next](#17-where-to-go-next)

---

## 1. The problem SELinux solves

Normal Linux permissions (the `rwx` bits you see in `ls -l`) are called **DAC** — Discretionary Access Control. "Discretionary" because the *owner* of a file gets to decide who can touch it. This has a well-known weakness: if a process is running as `root`, or an attacker manages to hijack a root process, that process can do essentially *anything* — read any file, write any device, kill any process — because root is above the ownership rules.

SELinux exists to close that gap. It adds a second, independent layer of rules that sits on top of normal Linux permissions. Even a process running as root has to also satisfy this second layer before an action is allowed. On Android, this is what stops a compromised app or a buggy system service from reading another app's private data, writing to raw storage, or reconfiguring the network stack, even if it somehow gets elevated privileges.

## 2. DAC vs MAC — the core idea

| | DAC (traditional Unix permissions) | MAC (SELinux) |
|---|---|---|
| Who decides access? | The file/resource **owner** | A **central, system-wide policy** |
| Can the owner override it? | Yes | No — not even root |
| Granularity | Coarse (user/group/other) | Fine (per type, per action, per object class) |
| Default behavior | Allow unless explicitly denied | **Deny unless explicitly allowed** |

That last row is the important mental shift. SELinux's entire philosophy is **default denial**: nothing is permitted unless a rule says so. This is the opposite of how most people think about permissions day to day, and it's the single most important idea to internalize before anything else makes sense.

## 3. How SELinux actually works under the hood

SELinux is implemented as an **LSM** (Linux Security Module) — a hook framework built into the kernel. Whenever a sensitive action is about to happen (open a file, connect a socket, send a signal, etc.), the kernel calls into SELinux *before* actually doing it. SELinux looks at:

- **Who** is asking (the security label of the process)
- **What** they're asking to do it to (the security label of the target object)
- **What kind of action** it is

...and checks that triple against the loaded policy. If there's a matching `allow` rule, the action proceeds. If not, it's blocked (in enforcing mode) and logged (always, if auditing is on).

This means SELinux decisions never depend on *who owns* something — only on the *labels* involved. That's what makes it "mandatory": the policy, not the object owner, is the single authority.

## 4. The building blocks: types, domains, classes, permissions

Android's SELinux policy is written using **Type Enforcement (TE)** — one specific model within SELinux, and the one that matters most in practice.

- **Type** — every object (a file, a process, a socket, a directory...) is tagged with a type. E.g. an ordinary installed app gets the type `untrusted_app`.
- **Domain** — when the "object" is a *process*, its type is called its domain instead. Same concept, different name, because it's talking about something active rather than passive.
- **Class** — the *kind* of object being accessed: `file`, `dir`, `socket`, `binder`, `capability`, etc. Classes are a fixed, rarely-changing list defined by the kernel/policy — you don't invent new ones casually.
- **Permission** — the specific operation within a class: `read`, `write`, `open`, `execute`, `connect`, `ioctl`, and so on. Each class has its own set of valid permissions (a `file` can be `read`; a `socket` can be `connect` — they're not interchangeable).

Think of it like a sentence: *"Can [this domain] perform [this permission] on [this class] of object labeled [this type]?"*

## 5. Reading and writing a policy rule

The most common rule shape looks like this:

```
allow source_type target_type:class permission_list;
```

- **source** — who's asking (a domain/type)
- **target** — what's being acted on (a type)
- **class** — the kind of object
- **permissions** — the specific action(s), space-separated inside `{ }` if there's more than one

A concrete example:

```
allow untrusted_app app_data_file:file { read write };
```

Read this left to right: *the `untrusted_app` domain is allowed to `read` and `write` objects of class `file` that are labeled `app_data_file`.* That's it — that's the whole grammar. Everything else in sepolicy is variations and shortcuts around this one pattern.

Rule types you'll encounter beyond plain `allow`:

| Rule | Meaning |
|---|---|
| `allow` | Grants the permission |
| `neverallow` | A build-time assertion — "this must never be allowed, fail the build if it is" — used to enforce security invariants |
| `auditallow` | Log the access even though it's allowed (useful for watching a sensitive path) |
| `dontaudit` | Suppress logging for a denial you've decided is noise, without granting it |

## 6. Attributes and macros (writing less repetitive policy)

Writing the same rule for every app-like type would be tedious and error-prone, so SELinux gives you two shortcuts.

**Attributes** — a label you can attach to multiple types, so one rule referencing the attribute applies to all of them:

```
typeattribute untrusted_app appdomain;
typeattribute isolated_app appdomain;

allow appdomain app_data_file:file { read write };
```

Here, `appdomain` isn't a real type — it's a *group tag*. The rule above silently expands to cover every type tagged with `appdomain`. Common built-in attributes include `domain` (every process type) and `file_type` (every file type).

**Macros** — file permissions in particular have a lot of fiddly overlap (e.g. `read` alone isn't enough to actually open a file; you also need `open`). Android ships macro bundles like `rw_file_perms` so you don't have to remember the exact permission set every time:

```
allow appdomain app_data_file:file rw_file_perms;
```

Rule of thumb: **use the macro if one exists** — it reduces the chance of a denial caused by one missing permission in a longer list you typed by hand.

## 7. The security context / label

Every object's actual tag is called a **security context** (or just "label"). You'll see it with `ls -Z` (files) or `ps -Z` (processes). It has this shape:

```
user:role:type:sensitivity[:categories]
```

Example: `u:r:untrusted_app:s0:c15,c256,c513,c768`

Breaking it down:
- `u` — the SELinux user (not a Linux user — see Section 10)
- `r` — the SELinux role
- `untrusted_app` — the **type**, the field that actually matters for Type Enforcement rules
- `s0` — the sensitivity level (Multi-Level Security)
- `c15,c256,...` — **categories**, used on Android to separate one app's data from another's and one device user's data from another's, even when the *type* is otherwise identical

In everyday Android work, the `type` field is what you'll be reading and reasoning about 95% of the time.

## 8. Enforcement modes: permissive vs enforcing

SELinux (and each domain within it) can run in one of two modes:

- **Permissive** — denials are logged but not actually blocked. Useful while developing: you get to see what *would* have been denied without breaking the device.
- **Enforcing** — denials are logged **and** blocked. This is how a production Android device runs.

Android also supports **per-domain permissive mode**: the system overall can be enforcing, while one specific domain (e.g. a new vendor service you're bringing up) is temporarily permissive. This lets you develop and test a new component's policy incrementally without turning off protection everywhere else.

Two commands you'll use constantly during development:
```
getenforce      # show current global mode
setenforce 0    # set permissive (0) — testing only, needs root
setenforce 1    # set enforcing (1)
```

## 9. Android's history with SELinux

Useful context for *why* the policy looks the way it does today:

- **Before Android 4.3**: sandboxing was DAC-only — each app got a unique Linux UID and that was the whole boundary.
- **Android 4.3**: SELinux introduced, but in permissive mode (logging only).
- **Android 4.4**: enforcing mode, but only for a handful of critical domains (`installd`, `netd`, `vold`, `zygote`).
- **Android 5.0**: enforcing mode for *everything* — 60+ domains. From here on, any process left unlabeled generally falls into a generic domain that produces a denial, which is the signal that it needs its own dedicated domain.
- **Android 6.0**: tightened further — better isolation between device users, filtering of `ioctl` commands, and drastically reduced `/proc` visibility.
- **Android 7.0**: the media stack (previously one big `mediaserver` process) was split into smaller, more narrowly-privileged processes specifically to shrink what a media-parsing bug could reach.
- **Android 8.0**: introduced **Treble** compatibility — see Section 12.

The overall trend across every release is the same: fewer broad exceptions, more narrowly scoped domains, less reliance on "generic" catch-all types.

## 10. What Android deliberately does NOT use

Upstream SELinux (as used on servers, e.g. Fedora/RHEL) has several features Android intentionally ignores to keep the policy simpler to build, audit, and debug:

| Feature | Status on Android |
|---|---|
| SELinux **users** | Not used — only one user, `u`, exists at all |
| SELinux **roles** / RBAC | Not used in any meaningful way — just `r` (subjects) and `object_r` (objects) as placeholders |
| **Sensitivity** levels | Not used — always fixed at `s0` |
| **Categories** | *Are* used, but only for app/user data isolation, not full Multi-Level Security |
| **Booleans** (runtime policy toggles) | Not used — Android policy is fixed at build time and never depends on device state, which makes debugging far more predictable |
| Policy language | Mostly the plain Kernel Policy Language; CIL (Common Intermediate Language) shows up in a few specific places |

Knowing this up front will save you time: if you read a general SELinux tutorial that talks about `semanage boolean` or multiple SELinux users, that material doesn't apply to Android policy.

## 11. How sepolicy is organized in the Android source tree

The policy source lives in `system/sepolicy` in AOSP, broadly split into two trees:

- **`public/`** — the stable interface: types and macros that vendor/device-specific policy is allowed to depend on across Android versions.
- **`private/`** — internal platform policy, including the actual list of object classes (`security_classes`) and permissions (`access_vectors`), which are considered implementation detail.

Each domain typically gets its own `.te` (Type Enforcement) file — e.g. `netd.te`, `vold.te` — containing the `type` declaration, its `typeattribute` memberships, and its `allow` rules. Labels are then attached to real files and processes via **context files**, most importantly:

- `file_contexts` — maps filesystem paths to labels (what type a given file/directory gets)
- `service_contexts` — labels for Binder services
- `property_contexts` — labels for system properties
- `seapp_contexts` — how app processes get labeled (e.g. deciding `untrusted_app` vs `isolated_app`)

A device/vendor tree builds its own additional policy on top of the AOSP base, rather than editing AOSP's files directly.

## 12. Treble: why vendor and system policy are split

Starting with Android 8.0, Android adopted **Treble**, which separates the low-level vendor/SoC code from the Android system framework so each can be updated somewhat independently (`vendor.img`/`boot.img` vs `system.img`).

This forced SELinux policy to also be splittable: the platform team ships system policy, the device/SoC vendor ships vendor policy, and the two need to combine correctly even when they're built at different times by different parties. One asymmetric rule falls out of this: a device can run a newer vendor image than platform image, but **not** the reverse — the vendor policy version must never be newer than the platform's. Android provides compatibility mechanisms specifically to prevent this mismatch from forcing unnecessary simultaneous OTA updates for platform and vendor.

## 13. The tools you'll actually use day-to-day

| Tool | What it's for |
|---|---|
| `ls -Z` | Show the security context of files |
| `ps -Z` | Show the security context (domain) of running processes |
| `getenforce` / `setenforce` | Check/change global enforcing mode |
| `dmesg` / `logcat` | Where denials are logged as `avc: denied` messages |
| `audit2allow` | Reads denial logs and *suggests* the `allow` rule(s) that would satisfy them — a starting draft, not a final answer |
| `sesearch` | Query a compiled policy: "what rules allow domain X to do Y to type Z?" |
| `seinfo` | Summarize a compiled policy: list types, attributes, classes it contains |
| `checkpolicy` / `secilc` | Compile source policy (TE / CIL) into the binary policy the kernel loads |

The healthy workflow is: **run in permissive → capture denials → use `audit2allow` as a rough draft → hand-verify and tighten by hand → re-test in enforcing.** Never ship a rule straight out of `audit2allow` without reading it — it tends to over-grant because it only knows about the one denial it saw, not your actual security intent.

## 14. Debugging a denial, step by step

1. **Find the denial.** Search `logcat` or `dmesg` for a line containing `avc: denied`.
2. **Read the five key fields** in the denial:
   - `scontext` — the source domain (who was blocked)
   - `tcontext` — the target's label (what they were trying to reach)
   - `tclass` — the object class (file, socket, binder, ...)
   - the requested permission(s) (e.g. `{ read }`)
   - `comm` — the process name, useful for cross-checking `scontext`
3. **Decide if the denial is expected or a bug.** Ask: *should* this domain ever legitimately need this access? Sometimes a denial reveals an actual bug elsewhere (e.g. code opening a file it shouldn't touch at all) — the fix there is to change the code, not the policy.
4. **If access should be granted**, write the narrowest possible `allow` rule — ideally reusing an existing type/attribute rather than inventing a new overly-broad one.
5. **Rebuild and reload policy, re-test in permissive mode first**, confirm the specific denial is gone, then move that domain back toward enforcing.
6. **Watch for a chain reaction** — granting one permission sometimes reveals the next thing that was also missing. This is normal; keep the loop going until the operation succeeds cleanly.

## 15. A worked example, start to finish

Suppose a new vendor daemon called `mydaemon` needs to read a file `/data/vendor/mydata/config`.

**Step 1 — see the denial (permissive mode):**
```
avc: denied { read } for comm="mydaemon" name="config"
 scontext=u:r:mydaemon:s0 tcontext=u:object_r:vendor_data_file:s0 tclass=file
```

**Step 2 — read it:** domain `mydaemon` wants `read` on a `file` labeled `vendor_data_file`.

**Step 3 — check if a dedicated label makes more sense.** Lumping everything under the generic `vendor_data_file` is usually too broad long-term; better practice is to give this data its own type, e.g. `mydaemon_data_file`, and label the path for it in `file_contexts`:
```
/data/vendor/mydata(/.*)?    u:object_r:mydaemon_data_file:s0
```

**Step 4 — write the narrow rule** in `mydaemon.te`:
```
allow mydaemon mydaemon_data_file:file r_file_perms;
```

**Step 5 — rebuild, reload, retest.** Confirm the denial disappears and no unrelated denial pops up in its place.

This is the same loop every real sepolicy change follows, just scaled up for bigger services.

## 16. Cheat sheet

```
# Rule grammar
allow  <source> <target>:<class> <permissions>;

# Check mode
getenforce

# Toggle mode (dev/debug only)
setenforce 0   # permissive
setenforce 1   # enforcing

# Inspect labels
ls -Z /path/to/file
ps -Z

# Find denials
logcat | grep avc
dmesg  | grep avc

# Draft a rule from a denial (verify before using!)
audit2allow -a

# Query a compiled policy
sesearch --allow -s mydaemon -c file
seinfo -tlist        # list all types
seinfo -alist        # list all attributes
```

**Mental checklist before adding any rule:**
- Is there already an attribute/macro that covers this instead of a one-off rule?
- Am I granting the *narrowest* class/permission set that solves the actual need?
- Does this new access make sense as a permanent security posture, or am I just silencing a denial?

## 17. Where to go next

Once this document feels comfortable, the natural next steps are:
- Set up a local AOSP checkout (or just browse `system/sepolicy` on Android's public source viewer) and read real `.te` files for services you recognize (e.g. `netd.te`, `vold.te`).
- Deliberately break something in a test/emulator build, watch the resulting `avc: denied` message, and walk it through the debugging steps in Section 14 yourself.
- Read the SELinux Notebook (community-maintained, in-depth reference on the policy language itself) once you want detail beyond what Android's docs cover.

**Primary reference used for this guide:** Android Open Source Project, *Security-Enhanced Linux in Android* and *SELinux concepts* — `source.android.com/docs/security/features/selinux` and its `concepts` sub-page.
