---
title: Await v2.10 in termux packages
description: "Showing the behind the scenes of upgrading packages in termux"
date: 2026-09-27
tags: ["github", "termux", "open source"]
---

## Introduction

I was scrolling throught the issues section of [termux-package](https://github.com/termux/termux-packages/issues/) when i stumpled upon this [issue](https://github.com/termux/termux-packages/issues/31984), upon a closer inspection of the logs i figured out it was a file not found error.

```
-- Build files have been written to: /home/builder/.termux-build/await/build
/home/builder/.termux-build/await/src/await.c:19:10: fatal error: 'spawn.h' file not found
19 | #include <spawn.h>
|          ^~~~~~~~~
1 error generated.
ERROR: failed to build.
```

I first thought it was a case of the developer changing the header files to another directory or something, so i cloned the source code locally and check it.

```
pranav@server:~/termux-packages/packages/await/await-2.10.0.mod$ find . -iname "spawn.h" -type f
pranav@server:~/termux-packages/packages/await/await-2.10.0.mod$ grep "spawn.h" -r
await.c:#include <spawn.h>
```

The `spawn.h` file literally had only one mention in the entire code base and it took me a really long time to found that it was a posix header file and is available on bionic c library but i was only implemented in API level 28 and termux compiles stuff for API level 27 and below.

After a little digging around to find how to implement it, i eventually found this [issue](https://github.com/termux/termux-packages/issues/4634) which gave lead me to `libandroid-spawn`.

```
pranav@server:~/termux-packages$ grep -r "libandroid-spawn" | grep build.sh
--snip--
packages/bazel/build.sh:TERMUX_PKG_BUILD_DEPENDS="libandroid-spawn-static, patch, unzip, zip"
packages/ninja/build.sh:TERMUX_PKG_DEPENDS="libandroid-spawn, libc++"
packages/lua-language-server/build.sh:TERMUX_PKG_DEPENDS="binutils, libandroid-spawn, libc++"
packages/rgbds/build.sh:TERMUX_PKG_DEPENDS="libandroid-spawn, libandroid-support, libc++, libpng"
packages/llama-cpp/build.sh:TERMUX_PKG_DEPENDS="libandroid-spawn, libc++, libcurl"
packages/cups/build.sh:TERMUX_PKG_BUILD_DEPENDS="libandroid-spawn"
packages/openjdk-25/build.sh:TERMUX_PKG_DEPENDS="libandroid-shmem, libandroid-spawn, libiconv, libjpeg-turbo, zlib, littlecms, alsa-plugins"
--snip--
```

We can see that inorder to use `libandroid-spawn` we need to import it via `TERMUX_PKG_BUILD_DEPENDS` so we do exactly that.

```
# in build.sh of await
TERMUX_PKG_SRCURL=https://github.com/slavaGanzin/await/archive/refs/tags/${TERMUX_PKG_VERSION}.tar.gz
TERMUX_PKG_SHA256=49acabc0c4859e4f0527cf40c0b06f88240c5dd70e662a63bf3eb853043917f9
TERMUX_PKG_AUTO_UPDATE=true
TERMUX_PKG_DEPENDS="libandroid-spawn"
```

Now we can try to compile this.

```
builder@c2ccab953c88:~/termux-packages$ ./build-package.sh -f -I await
--snip--
ld.lld: error: undefined symbol: posix_spawnattr_setflags
>>> referenced by await.c
>>>               /tmp/await-0a6d22.o:(shell)

ld.lld: error: undefined symbol: posix_spawnattr_setpgroup
>>> referenced by await.c
>>>               /tmp/await-0a6d22.o:(shell)

ld.lld: error: undefined symbol: posix_spawn
>>> referenced by await.c
>>>               /tmp/await-0a6d22.o:(shell)

ld.lld: error: undefined symbol: posix_spawnattr_destroy
>>> referenced by await.c
>>>               /tmp/await-0a6d22.o:(shell)

ld.lld: error: undefined symbol: posix_spawn_file_actions_destroy
>>> referenced by await.c
>>>               /tmp/await-0a6d22.o:(shell)
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

As you can see even though we added `libandroid-spawn` the linker can't still find the function symbols.

I decided to check how other packages work around this issue using `posix_spawn` as a search term (cus most functions in `spawn.h` start with `posix_spawn_something`).

```
pranav@server:~/termux-packages$ grep "posix\_spawn" -r | grep build.sh
--snip--
packages/guile/build.sh:ac_cv_func_posix_spawn=yes
packages/bison/build.sh:ac_cv_have_decl_posix_spawn=no
packages/gettext/build.sh:ac_cv_have_decl_posix_spawn=no
packages/libandroid-spawn/build.sh:	$CXX $CFLAGS $CPPFLAGS -I$TERMUX_PKG_BUILDER_DIR -c $TERMUX_PKG_BUILDER_DIR/posix_spawn.cpp
--snip--

pranav@server:~/termux-packages$ nvim packages/guile/build.sh
--snip--
# https://github.com/termux/termux-packages/issues/14806
TERMUX_PKG_NO_STRIP=true
TERMUX_PKG_EXTRA_CONFIGURE_ARGS="
LIBS=-landroid-spawn
ac_cv_func_posix_spawn=yes
ac_cv_func_posix_spawnp=yes
gl_cv_func_posix_spawn_works=yes
--snip--
```

As we can see the `guile` package uses `TERMUX_PKG_EXTRA_CONFIGURE_ARGS` to include it via `LIBS=-landroid-spawn`.

We can also implement this in our `await` package.

```
# in build.sh of await
--snip--
TERMUX_PKG_SHA256=49acabc0c4859e4f0527cf40c0b06f88240c5dd70e662a63bf3eb853043917f9
TERMUX_PKG_AUTO_UPDATE=true
TERMUX_PKG_DEPENDS="libandroid-spawn"
TERMUX_PKG_EXTRA_CONFIGURE_ARGS="LIBS=-landroid-spawn"

termux_step_make() {
	$CC $CPPFLAGS $CFLAGS "$TERMUX_PKG_SRCDIR"/await.c -o await $LDFLAGS
}
--snip--
```

We now try to build this package, but **suprisee** it won't work.

We actually need to insert the `-landroid-spawn` flag to the `$CC` line int the `termux_step_make()`.

This is because the `TERMUX_PKG_EXTRA_CONFIGURE_ARGS` only supplies the flags when the build process runs the configure script but in this `build.sh` it completely replaces the configure step with it's own custom `termux_step_make`, so there is no compiling and it is directly compiling via `$CC`, so the simple fix is to add the `-landroid-spawn` directly at the end of the compiling line.

```
pranav@server:~/termux-packages$ cat packages/await/build.sh
--snip--
TERMUX_PKG_SHA256=49acabc0c4859e4f0527cf40c0b06f88240c5dd70e662a63bf3eb853043917f9
TERMUX_PKG_AUTO_UPDATE=true
TERMUX_PKG_DEPENDS="libandroid-spawn"

termux_step_make() {
	$CC $CPPFLAGS $CFLAGS "$TERMUX_PKG_SRCDIR"/await.c -o await $LDFLAGS -landroid-spawn
}
--snip--
```

With the flag in place the build now finishes perfectly

At the time of writing this the pr is currently under review, hopefully by the time you read this this [pr](https://github.com/termux/termux-packages/pull/32028) will be merged.

Thank you and have a nice day

## See also

- [Github pr](https://github.com/termux/termux-packages/pull/32028) - the pull request
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
