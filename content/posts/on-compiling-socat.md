+++
title = "On Compiling Socat for Windows 10"
author = ["Jimmy Lee"]
lastmod = 2021-02-12T01:02:10+08:00
tags = ["tools", "win10"]
categories = ["blog"]
draft = false
weight = 2003
+++

## Cannot find socat on this thing {#cannot-find-socat-on-this-thing}

Socat is so cool.  It is a tool comparable to netcat but has some extra features that make it worth it to learn how to use. It is installed in Kali Linux by default. It is also only a pacman -S socat away in Arch Linux, but not readily available in Windows.

Actually, with quick search, I did find several binaries to download but I didn't feel right downloading random .exe file from the internet.

-   <https://sourceforge.net/projects/unix-utils/files/socat/1.7.3.2/>
-   <https://cybercircuits.co.nz/web/blog/socat-1-7-3-2-for-windows>
-   <https://github.com/StudioEtrange/socat-windows>

First two links are actually maintained by the same person and third link was commited 6 years ago (socat has been updated since then).  Not that I didn't trust the blobs (healthy distrust?), I just wanted to compile it myself.


## Going to the Source {#going-to-the-source}

-   <http://www.dest-unreach.org/socat/>

README mentioned that it is possible to compile socat with Cygwin. I've never used cygwin before (I've only used linux and Mac ever since I started programming) but decided this will be the way.

These two videos helped me get started on Cygwin:

-   [How to Install Cygwin on Windows 10](https://www.youtube.com/watch?v=QonIPpKodCw)
-   [How to Install Cygwin Packages](https://www.youtube.com/watch?v=VyIY8cjn9xY)

Installed required Cygwin packages (gcc-{fortran,objc,objc++} may not be necessary):

-   [X] gcc-core - 10.2.0-1
-   [ ] gcc-fortran - 10.2.0-1
-   [X] gcc-g++ - 10.2.0-1
-   [ ] gcc-objc - 10.2.0-1
-   [ ] gcc-objc++ - 10.2.0-1
-   [X] libkrb5-devel - 1.15.2-2
-   [X] libkrb5\_3 - 1.15.2-2
-   [X] libreadline-devel - 7.0.3-3
-   [X] libssl-devel - 1.1.1f-1
-   [X] libwrap-devel - 7.6-26
-   [X] make - 4.3-1
-   [X] tcp\_wrappers - 7.6-26

Then in Cygwin64 Terminal:

```shell
tar -xvzf socat-1.7.4.1.tar.gz
cd socat-1.7.4.1
./configure
make
make install
```

Finally, socat can be found at \`/usr/local/bin/socat.exe\`.

But wait... what if you want to bring it along to another machine?


## Bringing it with you {#bringing-it-with-you}

Copy over \`socat.exe\` to another machine to test.
It will complain about these files not being found:

```text
cygreadline7.dll
cygcrypto-1.1.dll
cygwin1.dll
cygssl-1.1.dll
cygncursesw-10.dll
cygwrap-0.dll
cygz.dll
```

Easy fix is to copy these .dlls and socat.exe into a folder.
Zipped up, it becomes 3.08MB and easy to move around. Tada~


## Hope this was helpful {#hope-this-was-helpful}

I hope this walkthrough was helpful for anybody looking to have socat on Windows 10.  Going through this process not only gave me socat.exe that I can use (going through OSCP/PWK materials) but I've learned how I can bring in some of linux tools I take for granted while using Windows.

Even after having read all of this, if you just want to find the file to use - <https://github.com/honuonhval/socat-win10> - I've created a repository.
