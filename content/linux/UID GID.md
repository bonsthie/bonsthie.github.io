## 🪪 What Are UID and GID?

### ✅ **UID (User ID)**

The **User ID** is a number that identifies a user on a Unix-like system.

- Each user (like `root`, `alice`, `bob`) has a unique UID.
    
- The **root user** always has:

```bash
UID = 0
```

> 🧠 UID controls **what files and actions a process is allowed to access**.

---

### ✅ **GID (Group ID)**

The **Group ID** is a number that identifies a **group of users**.

- Each user can be part of one or more groups.
    
- Each file or process can belong to a group (via GID), and group permissions apply.
    

> 🧠 GID controls **access rights shared across users** in the same group.

---

## 🔧 Example:

Let’s say you have a user named `alice`:

```bash
$ id alice uid=1001(alice) gid=1001(alice) groups=1001(alice),27(sudo)
```

- **UID = 1001** → This is Alice’s identity.
    
- **GID = 1001** → Alice’s main group.
    
- **Groups** → Alice is also in the `sudo` group (which lets her run admin commands).