---
title: Your first open source contribution - termux
date: 2026-03-08
tags: ["termux", "coding", "open source"]
---

This guide is supposed to guide/inspire you and not fully help u to make your first open source software contributions
I will be using my own github commits as an example

**Github: https://github.com/pranav10780**

## What project?

Here i am using the [termux/termux-packages](https://github.com/termux/termux-packages) as an example specifically this [issue](https://github.com/termux/termux-packages/issues/28458)

If you look at the log file generated and skip to the end you will see

```Downloading https://github.com/jarun/nnn/archive/refs/tags/v5.2.tar.gz
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
Dload  Upload   Total   Spent    Left  Speed
0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100  263k  100  263k    0     0  1055k      0 --:--:-- --:--:-- --:--:-- 1055k
aarch64-linux-android-clang  -isystem/data/data/com.termux/files/usr/include/c++/v1 -isystem/data/data/com.termux/files/usr/include  -fstack-protector-strong -Oz -std=c11 -Wall -Wextra -Wshadow -O3 -DNCURSES_WIDECHAR -I/data/data/com.termux/files/usr/include -L/data/data/com.termux/files/usr/lib -Wl,-rpath=/data/data/com.termux/files/usr/lib -Wl,--enable-new-dtags -Wl,--as-needed -Wl,-z,relro,-z,now -Wl,--no-as-needed,-landroid-support,--as-needed -o nnn  src/nnn.c -lreadline -L/data/data/com.termux/files/usr/lib -Wl,-rpath=/data/data/com.termux/files/usr/lib -Wl,--enable-new-dtags -lncursesw -lpthread
src/nnn.c:6492:2: error: call to undeclared function 'pthread_setaffinity_np'; ISO C99 and later do not support implicit function declarations [-Wimplicit-function-declaration]
6492 |         pthread_setaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);
|         ^
1 error generated.
make: *** [Makefile:219: nnn] Error 1
ERROR: failed to build.
```

You can see that the error is trying to call an undeclared function called `pthread_setaffinity_np`, lets search and learn more about it

**It sets that thread to execute on the specified cpuset forever. It is not somehow method-scoped.** [Source](https://stackoverflow.com/questions/8153664/how-to-use-pthread-setaffinity-np)

So the function exists in glibc but since termux is compiling for android we need to check in the bionic library.

We can see this function exists in the [bionic changelog](https://android.googlesource.com/platform/bionic/+/HEAD/docs/status.md)

```
New libc functions in API level 36:

    qsort_r, sig2str/str2sig (POSIX Issue 8 additions).
    GNU/BSD extension lchmod.
    GNU extensions pthread_getaffinity_np/pthread_setaffinity_np.
    New system call wrapper: mseal (<sys/mman.h>).
```

We can see that the function was introduced in API level 36 but termux compiles for API level 24 so the function does not exist

Fortunately for us instead of creating our own patch we can make use existing patch made by [licy183](https://github.com/licy183) : [0001-dummy-pthread-affinity.patch](https://github.com/termux/termux-packages/blob/5c37d88db64609aa2b1294d6a71e98718c4beda4/packages/oidn/0001-dummy-pthread-affinity.patch#L8-L12)

Now in this patch the `pthread_getaffinity_np` is also defined which we don’t need so we can remove it in the final patch

We also need `_GNU_SOURCE`, `assert.h` and `sched.h` for it to compile properly. [Source](https://man7.org/linux/man-pages/man3/pthread_setaffinity_np.3.html)

After making a diff with the source code with our modified pthread patch it should look something like this

```diff
diff --git a/src/nnn.c b/src/nnn.c
index 108e2995..9f7a832e 100644
--- a/src/nnn.c
+++ b/src/nnn.c
@@ -30,6 +30,9 @@
 
 #define _FILE_OFFSET_BITS 64 /* Support large files on 32-bit glibc */
 
+#define _GNU_SOURCE
+#include <assert.h>
+#include <sched.h>
 #if defined(__linux__) || defined(MINGW) || defined(__MINGW32__) \
  || defined(__MINGW64__) || defined(__CYGWIN__)
 #ifndef _GNU_SOURCE
@@ -978,6 +981,13 @@ static uchar_t xchartohex(uchar_t c)
  * Source: https://elixir.bootlin.com/linux/latest/source/arch/alpha/include/asm/bitops.h
  * Optimized: atomic test-and-set per word so different inodes (different words) don't contend.
  */
+#if defined(__BIONIC__)
+static inline int pthread_setaffinity_np(pthread_t thread, size_t cpusetsize,
+                                         cpu_set_t *cpuset) {
+  assert(pthread_equal(pthread_self(), thread));
+  return sched_setaffinity(0, cpusetsize, cpuset);
+}
+#endif
 static inline bool test_set_bit(uint_t nr)
 {
  nr &= HASH_BITS;
```

We have only imported `pthread_setaffinity_np` from the original patch

For the final touches we need to modify the `build.sh` file

- We need to change the `TERMUX_PKG_VERSION` to 5.1 to 5.2
- Remove the `TERMUX_PKG_REVISION` because we are bumping a package/upgrading it
- Update the checksum/hash inside the `TERMUX_PKG_SHA256` variable

The final diff of build should look like this

```diff
 TERMUX_PKG_DESCRIPTION="Free, fast, friendly file browser"
 TERMUX_PKG_LICENSE="BSD 2-Clause"
 TERMUX_PKG_MAINTAINER="@termux"
-TERMUX_PKG_VERSION="5.1"
-TERMUX_PKG_REVISION=2
+TERMUX_PKG_VERSION="5.2"
 TERMUX_PKG_SRCURL=https://github.com/jarun/nnn/archive/refs/tags/v${TERMUX_PKG_VERSION}.tar.gz
-TERMUX_PKG_SHA256=9faaff1e3f5a2fd3ed570a83f6fb3baf0bfc6ebd6a9abac16203d057ac3fffe3
+TERMUX_PKG_SHA256=f166eda5093ac8dcf8cbbc6224123a32c53cf37b82c5c1cb48e2e23352754030
 TERMUX_PKG_AUTO_UPDATE=true
 TERMUX_PKG_DEPENDS="file, findutils, readline, wget, libandroid-support, lzip"
 TERMUX_PKG_BUILD_IN_SRC=true
```

Thank you for reading along this isn’t meant for an serious guide to be used rather as a one time read if you have any doubts please ask

The final commit: https://github.com/termux/termux-packages/pull/28474

## See also

- [Medium Story](https://medium.com/@pranavsuresh107/your-first-open-source-contribution-1a9d898802a7) - same story in medium
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
