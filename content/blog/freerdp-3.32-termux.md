---
title: Freerdp v3.32 in termux packages
description: "Showing the behind the scenes of upgrading packages in termux"
date: 2026-09-23
tags: ["github", "termux", "open source"]
---

## Introduction

I was planning to contribute something to opensource when i noticed this issue on [github](https://github.com/termux/termux-packages/issues/31875).

This was an automated issue made by the termuxbot2 when the autoupgrade script failed to update a package (usually due to not being able to apply existing patches). The simple fix is to manually add the patches to the source code and then regenerate the patch file.

## Downloading the code

First we need to visit the freerdp's source code (usually on github or gitlab) which can be found via looking in the `build.sh` file in the `TERMUX_PKG_HOMEPAGE` variable. In our case the link point to a [website](https://www.freerdp.com/) which has the link to the [github](https://github.com/FreeRDP/FreeRDP) directly in the header.

Under the [releases](https://github.com/FreeRDP/FreeRDP/releases) we can see that the latest release version is `3.32`.

We need to change the `TERMUX_PKG_VERSION` and `TERMUX_PKG_SHA256` to the latest version and update the checksum.

```
diff --git a/x11-packages/freerdp/build.sh b/x11-packages/freerdp/build.sh
index 9281eeedaa..8c4f911282 100644
--- a/x11-packages/freerdp/build.sh
+++ b/x11-packages/freerdp/build.sh
@@ -2,9 +2,9 @@ TERMUX_PKG_HOMEPAGE=https://www.freerdp.com/
 TERMUX_PKG_DESCRIPTION="A free remote desktop protocol library and clients"
 TERMUX_PKG_LICENSE="Apache-2.0"
 TERMUX_PKG_MAINTAINER="@termux"
-TERMUX_PKG_VERSION="3.31.1"
+TERMUX_PKG_VERSION="3.32.0"
 TERMUX_PKG_SRCURL=https://github.com/FreeRDP/FreeRDP/archive/refs/tags/$TERMUX_PKG_VERSION.tar.gz
-TERMUX_PKG_SHA256=254de9fe176758e9787347469fb310523782f03c61130508b51b266e374eb6c1
+TERMUX_PKG_SHA256=0453420dc3d9c3c03952e4e3f52e5b213ce0eae37346cd9a08bbd30c30a23c21
 TERMUX_PKG_AUTO_UPDATE=true
 TERMUX_PKG_DEPENDS="libandroid-shmem, libcairo, libicu, libjpeg-turbo, libusb, libwayland, libx11, libxcursor, libxdamage, libxext, libxfixes, libxi, libxinerama, libxkbcommon, libxkbfile, libxrandr, libxrender, libxv, openssl, pulseaudio, zlib"
 TERMUX_PKG_BUILD_DEPENDS="libwayland-cross-scanner, libwayland-protocols"
```

>NOTE: We need to run `(source build.sh 2>/dev/null; curl -LO "$TERMUX_PKG_SRCURL")` after updating the `TERMUX_PKG_VERSION` variable to get the source code locally

>NOTE: In order the get the hash we need to use `sha256sum freerdp.tar.gz`

>NOTE: In order the open the zip file use `tar -xvf freerdp.tar.gz`

>NOTE: Terms like `freerdp.tar.gz`,`freerdp` and `freerdp.mod` are fictional names, the actual names will vary.

Now we have the source code, we can use `cp -a freerdp freerdp.mod`.
We will use this `freerdp.mod` to apply the patch changes and then use diff to regenerate the patch file.

## Applying the patch

Looking at the end of the log file pasted by termuxbot2 in the [issue](https://github.com/termux/termux-packages/issues/31875) we can see which patch is failing.

![issue](/images/freerdp/issue.png)

We can see that the `0004-fix-hardcoded-paths.patch` patch is failing, now we open the file and apply the changes to `freerdp.mod`

After we apply the patches manually then we can regenerate the `0004-fix-hardcoded-paths.patch` via diff by
```
diff -uNr freerdp freerdp.mod > 0004-fix-hardcoded-paths.patch
```

We can confirm if it was successful via `git diff` not recommended as viewing the patch of a patch file is very hard but it should look something like this.

![max](/images/freerdp/max.png)

## Building locally

Now we try to build the package locally to make sure it is working via docker.

>NOTE: Since i use ubuntu i use the `run-docker.sh` yours may vary. [Learn more](https://github.com/termux/termux-packages/wiki/Build-environment)

Now we enter the docker container via `./script/run-docker.sh` which should give you a shell then run

```
./build-package -I -f freerdp
```

>NOTE: The `-I` and `-f` flags are not necessary but it is good practice to use them. [Learn more](https://github.com/termux/termux-packages/wiki/Building-packages)

Remember things will not work out of the box in the first attempt, so read the logs and never stop trying and hopefully you should be able to learn something from this short blog, feel free to send emails regarding your doubts.

Thank you and have a nice day

## See also

- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
