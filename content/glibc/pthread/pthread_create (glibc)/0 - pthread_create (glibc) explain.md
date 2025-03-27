#glibc #Breakdown #thread #pthread

This document provides a detailed breakdown of the internal `__pthread_create_2_1` function from **glibc**, which is responsible for thread creation in POSIX-compliant systems.  
`__pthread_create_2_1` is the internal name for the `pthread_create` function. you can find the entire source code in glibc/[nptl](ntpl%20folder)/pthread_create.c or in [here](pthread_create%20(glibc)%20code.md)

Each major segment of the function is listed below with a short explanation and a link to a more detailed page in the same folder.

---

## 🧵 Overview

Function signature:

```c
int __pthread_create_2_1(pthread_t *newthread, const pthread_attr_t *attr,
                         void *(*start_routine)(void *), void *arg);
```

This function creates and starts a new thread with the specified attributes and entry point.

---

## 📦[1. Deferred Initialization](1%20-%20Deferred%20Initialization.md)

- **What it does:** Ensures one-time initialization in case the library starts in a single-threaded state.
    
- **Why it matters:** Prepares glibc internals for multithreading.
    

---

## ⚙️ [2. Thread Attribute Handling](2%20-%20Thread%20Attribute%20Handling.md)

- **What it does:** Uses user-provided thread attributes or retrieves system defaults.
    
- **Why it matters:** Ensures that every thread has a valid and complete set of attributes.
    

---

## 📊 [3. Stack and Thread Descriptor Allocation](3%20-%20Stack%20and%20Thread%20Descriptor%20Allocation.md)

- **What it does:** Allocates memory for the thread stack and sets up the internal thread descriptor (`struct pthread`).
    
- **Why it matters:** This is where the actual memory for the thread is reserved.
    

---

## 🔧 [4. TCB and Thread Descriptor Initialization](4%20-%20TCB%20and%20Thread%20Descriptor%20Initialization.md)

- **What it does:** Initializes fields in the thread descriptor such as start routine, argument, flags, guards, etc.
    
- **Why it matters:** Prepares everything needed before launching the thread.
    

---

## 📊 [5. Scheduling Parameter Determination](5%20-%20Scheduling%20Parameter%20Determination.md)

- **What it does:** Chooses whether to use the parent thread's scheduling policy or the explicitly defined one.
    
- **Why it matters:** Impacts how the OS schedules the new thread.
    

---

## 🔄 [6. Thread Count Management and Signal Blocking](6%20-%20Thread%20Count%20Management%20and%20Signal%20Blocking.md)

- **What it does:** Increments thread count and blocks signals to avoid race conditions during creation.
    
- **Why it matters:** Ensures the new thread starts in a controlled, race-free environment.
    

---

## 🤖 [7. Thread Creation and Debug Event Reporting](7%20-%20Thread%20Creation%20and%20Debug%20Event%20Reporting.md)


- **What it does:** Calls the actual system-level thread creation and notifies any attached debugger.
    
- **Why it matters:** It's where the thread actually starts running.
    

---

## ♻️ [8. Post-Creation Signal Restoration and Error Handling](8%20-%20Post-Creation%20Signal%20Restoration%20and%20Error%20Handling.md)

- **What it does:** Restores signal mask and handles any failures that occurred during creation.
    
- **Why it matters:** Prevents resource leaks and ensures consistent thread state.
    

---

## 🚪 [9. Finalization and Cleanup](9%20-%20Finalization%20and%20Cleanup.md)


- **What it does:** Unlocks and frees resources, sets internal flags, and returns to the caller.
    
- **Why it matters:** Completes the thread creation process cleanly.
    

---