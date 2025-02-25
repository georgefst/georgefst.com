# Cross compiling Haskell (without Nix)

This is a guide to building GHC as a [cross compiler](https://en.wikipedia.org/wiki/Cross_compiler), i.e. creating a `ghc` executable which runs in one environment and outputs code for another. The most common motivation for this is a desire to run Haskell code on a machine which is not itself powerful enough to run GHC effectively.

We'll be using the following configuration:

- An Intel-powered Arch Linux system is used both as the _build_ environment (where the compiler is built) and the _host_ environment (where the compiler runs).
- The _target_ environment (where the code produced by the compiler will run) is a fourth-generation Raspberry Pi running Raspberry Pi OS.

Much of this guide will generalise to other setups, at least for GNU/Linux.
it's likely that the only differences will be in how you get hold of an appropriate GCC toolchain [link to below].
and what prefix you pass in place of `aarch64-none-linux-gnu`
and so long as running `uname -m` outputs `x86_64` on the build/host system (which will be the case for the vast majority of non-Mac laptops and desktops [^5]) and `aarch64` for the target (most Pi-esque cheap computers), you can probably use the same ARM toolchain used here, you'll just have to find some way to unpack it manually if you're not on Arch

We'll be using GHC 9.10.1, though there haven't been _major_ changes to any of this since 9.2, which introduced the aarch64 backend
[what about Hadrian?]
[see Historical Notes section]
and GCC \_

## Why not just use Nix?

I love the _idea_ of Nix, and have used it extensively on some large projects, but it often makes for a frustrating experience. The major pain points being awful documentation and error messages. I won't go in to more detail here because it's [all](https://blog.wesleyac.com/posts/the-curse-of-nixos) [been](https://remy.grunblatt.org/nix-and-nixos-my-pain-points.html) [said](https://ianthehenry.com/posts/how-to-learn-nix/introduction) [before](https://serokell.io/blog/haskell-in-production-mercury).

we really need proper hyperlink highlighting here

I'm also wary of the "just use Nix" answer that's often given to all sorts of questions on Haskell forums. It can send new users down a major rabbit hole, and even give a false impression that mastering Nix is _necessary_ in order to work effectively with Haskell. We often don't seem to realise how simple the non-Nix approach can be, particularly for problems which have become simpler over time due to upstream tooling improvements. In that vein, I'm somewhat inspired by [this post on static linking](https://hasufell.github.io/posts/2024-04-21-static-linking.html).

One place where Nix is currently the only serious option (?) is TemplateHaskell
I don't know if anyone using ghc iserv without Nix
https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/compiler/external-interpreter
haskell.nix sets it up for you with qemu
Do both haskell Nix solutions solve this equally?
aha, cross compilation was haskell.nix's original raison detre
https://www.reddit.com/r/haskell/comments/h0e5ui/comment/ftnn2de/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button

well, actually, I guess using qemu for the whole thing is also an option
but much slower
https://www.reddit.com/r/haskell/comments/h0e5ui/comment/fto9eq9/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button

and of course I use qemu rather than Haskell.nix's own cross-compilation for my nixos-config repo

although things should in principle be much simpler as we often just want stuff like `makeLenses` which is platform-independent and should be able to run on host
there's some serious work slowly happening to untangle the relevant parts of the compiler
https://github.com/ghc-proposals/ghc-proposals/pull/682
on the other hand, it's surprising how few common libraries rely on TH these days
I've build some projects with pretty huge dependency graphs where it hasn't been a problem


## Steps

This is a CPU-intensive process, so you may wish to close down most windows in order to speed it up[^1]

took 17m on Billy with quick
a mid-range 2021 laptop with an i7 and 16GB of RAM
what about Crow?

## Install an appropriate cross-compiling GCC

this is, as mentioned, the part that you'll have to do quite differently if you're not on Arch

The easiest way to do this on Arch is to use your favourite AUR helper
e.g. run `yay aarch64-none-linux-gnu-gcc` and select an appropriate package
we'll use [`gcc-12.3`](https://aur.archlinux.org/packages/aarch64-none-linux-gnu-gcc-12.3-bin)
for `glibc` reasons
how do we know how to match?

Note that you should always be careful when it comes to trusting AUR packages
basically unmoderated (?)
source is fortunately small and clearly basically just downloads a tarball from a reputable website
and unpacks stuff
yes you should read the source in principle, but all the Hackage packages you're depending on are a similar threat
(?)
[^3]

There is an official package at `extra/aarch64-linux-gnu-gcc` but I've never used it since, Arch being what it is, it's always too up-to-date to be useful unless targeting a system which also uses the very latest `glibc`. If, for example, you're running Arch on your target device, then this _should_ work.

## GHCup cross-compilation

link back to https://discourse.haskell.org/t/wasm-and-singletons-base/10972/4?u=george.fst
(not sure why I said that?)

tools: https://gitlab.haskell.org/ghc/ghc/-/wikis/building/preparation/linux#arch
since we're using a source distribution, won't need alex or happy
well, that was a lie...
this would appear to be a lie: https://gitlab.haskell.org/ghc/ghc/-/wikis/building/getting-the-sources#via-source-distributions
unless they're just needed for cross for some reason
can't build native to test because I hit latex errors - work out how to disable doc build?
punt this as GHC issue
it's likely everything else needed is already installed
maybe `sudo pacman -S autoconf automake`
just make sure you have a normal recent GHC and Cabal installed via GHCup

grab your preferred source distribution, as described [here](https://gitlab.haskell.org/ghc/ghc/-/wikis/building/getting-the-sources)
e.g.
we're using GHC 9.10.1 as it's the latest reasonably stable version
I should try 9.6 and/or 9.8 on Crow to see whether they hit separate errors
I think they did and it would be a good excuse to ignore them and stay cutting edge
maybe even 9.4 (GHCup recommended) and 9.2 (first with native aarch64 backend (and Hadrian? or was that 9.4?))

```sh
curl https://downloads.haskell.org/~ghc/9.10.1/ghc-9.10.1-src.tar.xz -o ghc-9.10.1-src.tar.xz
tar xf ghc-9.10.1-src.tar.xz
cd ghc-9.10.1
./configure --target=aarch64-none-linux-gnu
GHC=ghc-9.6 ./hadrian/build -j$(nproc) --flavour=quick+native_bignum binary-dist
```

the configure step shouldn't take more than a minute / a few seconds (13 on Crow)
is this even worth saying?

actual build took 20m19s on Crow with nproc, although that was accidentally using 9.6 to build
pretty sure Billy took 17 something


is `-j4` faster than nproc on Crow with all its cores? should try it

(specifying `GHC=9.6.5` is necessary in order to avoid errors very similar to
https://gitlab.haskell.org/ghc/ghc/-/issues/24364
but only when building 9.10.1 release, not HEAD
not sure why...
)

It _should_ be possible to just run the following:
`ghcup compile ghc --hadrian -v 9.10.1 -b 9.10.1 -x aarch64-none-linux-gnu -j $(nproc) --flavour=quick+native_bignum --hadrian-ghc=9.6`
But GHCup's support for (cross compilation with?) Hadrian is currently broken
https://gitlab.haskell.org/ghc/ghc/-/issues/23983

NB I have the bindist on Crow's Desktop for now, with a hacky `PATH` modification in `aarch64-none-linux-gnu-cabal`
undocumented!
i.e. `cp -r _build/bindist/ghc-9.10.1-aarch64-none-linux-gnu ~/Desktop/aarch64-ghc-prefix/`
should suggest putting somewhere on PATH I guess?
and now Billy too, as of 2025/01/20!

### alternatively, if needing a particular commit

```sh
git clone --recursive https://gitlab.haskell.org/ghc/ghc.git
cd ghc
git checkout ghc-9.10.1-release
```

various intermediate steps between clone and configure
autoconf, init submodules...
or is this just what boot script does?
is this where we again hit issue where Arch Happy is too new for GHC 9.10 (`cabal install happy-1.20.1.1`)?
didn't happen when building from source tarball

## cabal-install wrapper

simple

inspired by moritz

...

## third-party libs - work through evdev/zlib example

there's no particular reason that these steps couldn't also be turned in to AUR packages

- also publish AUR packages for cross-compiling libevdev, zlib etc. ?
  - is Debian a step above here? https://github.com/Spotifyd/spotifyd/pull/122/files
    - EDIT also https://gitlab.haskell.org/ghc/ghc/-/wikis/building/cross-compiling-on-debian
  - depend on my `gcc` package?
    - or do we not need to be tied to a version
      - actually, I don't think that's an issue as these would be built from source
        - we would need to make sure they're rebuilt when compiler is upgraded
          - in the way that `ghcid` should be (but isn't) when libraries it depends on are upgraded
    - it does seem a bit weird to have them all tied to my GCC instead of the package in `community`
      - but obviously _I_ can't use that because glibc is too bleeding edge for RPi and sometimes even NixOS
        - unless there is some way to compile against an older glibc?
  - alternatively, Haskell `zlib` package has a `bundled-c-zlib` flag
    - investigate
      - seems to have been designed for Windows but may well work with cross-compilation
      - maybe it isn't actually a great approach: https://github.com/haskell/zlib/issues/22
      - added in 2020 (and renamed): https://github.com/haskell/zlib/pull/31
    - recommend that and provide similar in evdev?
    - actually, I've seen this in a few other places now, including Rust `openssl` lib

```sh
cd ~/Desktop
# LIBEVDEV_VERSION=1.13.1
LIBEVDEV_VERSION=1.13.3
curl -OL https://www.freedesktop.org/software/libevdev/libevdev-$LIBEVDEV_VERSION.tar.xz
tar xf libevdev-$LIBEVDEV_VERSION.tar.xz
cd libevdev-$LIBEVDEV_VERSION
./configure --host=aarch64-none-linux-gnu --prefix=/usr/aarch64-none-linux-gnu/
make
sudo make install
```

```sh
cd ~/Desktop
# ZLIB_VERSION=1.2.13
ZLIB_VERSION=1.3.1
curl -OL https://www.zlib.net/zlib-$ZLIB_VERSION.tar.gz
tar xzf zlib-$ZLIB_VERSION.tar.gz
cd zlib-$ZLIB_VERSION
CHOST=aarch64-none-linux-gnu ./configure --prefix=/usr/aarch64-none-linux-gnu/
make
sudo make install
```

what's this for? oh, stuff like Brick I guess

```sh
cd ~/Desktop
CURSES_VERSION=6.4
curl -OL https://invisible-island.net/archives/ncurses/ncurses-$CURSES_VERSION.tar.gz
tar xzf ncurses-$CURSES_VERSION.tar.gz
cd ncurses-$CURSES_VERSION
# without `--enable-overwrite`, headers end up in a prefixed directory where Haskell `terminfo` library can't find them
# I'm not quite sure why `--disable-stripping` was needed (18/10/2023), and maybe we can set `STRIP` env var instead
./configure --host=aarch64-none-linux-gnu --prefix=/usr/aarch64-none-linux-gnu --enable-overwrite --disable-stripping
make
sudo make install
```

any other, e.g. openssl?

## test

```sh
cd ~/code/hello-hs
aarch64-none-linux-gnu-cabal build
scp $(aarch64-none-linux-gnu-cabal list-bin .) pi:/tmp
ssh pi /tmp/hello-hs
```

do a trivial non-cabal hello world
then a cabal script utilising zlib
and maybe one of evdev-examples

## historical notes

I've been following these developments closely for 6 years
have been posting bits of it for at least 3 years: https://www.reddit.com/r/haskell/comments/ryw1go/comment/hruafla
publishing this before it hopefully becomes obsolete!

docker
winter - Rust
aarch64 GHC backend
https://gitlab.haskell.org/ghc/ghc/-/issues/17973#note_378621
aarch64 for RPiOS
this has all been getting easier
angerman work
RPi going 64-bit OS by default about two years ago?
Hadrian over Make

this works so well that perhaps GHC should provide bindists?

future - runtime retargetability
would make a lot of this obsolete
explain exactly what?
https://www.reddit.com/r/haskell/comments/zk1qs6/comment/j09kcjb/?utm_source=share&utm_medium=web2x&context=3
https://gitlab.haskell.org/ghc/ghc/-/issues/11470
https://gitlab.haskell.org/ghc/ghc/-/issues/23682

as for the need for this guide, there's some good stuff on the GHC wiki but it tends to be surrounded be a sea of obscure or outdated information
in particular, the cross pages don't mention Hadrian, and vice versa
besides, I think there's value in a guide focussed on a particular common scenarion and largely on the happy path
footnote: and I don't personally feel qualified to fix that
plus doing it in full generality is quite a task and probably best left to a GHC dev who actually works on this stuff
https://gitlab.haskell.org/ghc/ghc/-/wikis/cross-compilation/
https://gitlab.haskell.org/ghc/ghc/-/wikis/building/cross-compiling

## misc

go to Haskell Discourse to leave a comment

George Thomas 30/01/2025

1st February 2025

# footnotes

proper highlighting
why doesn't browser back button do what I'd hope for (can punt if I'm calling website WIP when launching this post)

one advantage of not using GHCup is that build products are cached such that if something trivial goes wrong then we don't start from scratch
I suspect that it wouldn't be difficult to patch GHCup to improve this
I suffered with this a lot a while ago, though it's maybe less relevant when following this guide anyway
EDIT: actually would have hit it several times when trying minor variations of these commands

there's a common sentiment (misconception?) that Haskell on Arch is a disaster due to dynamic linking
it's actually fine as long as you use GHCup for development, and make sure it's directory is first on `PATH`
this is another thing that I've also posted in various places over the years
it could form the basis of another blog post
but honestly, there's not much more to say on the matter than that

- alternative - "sysroot" approach
  - there are plenty of guides online about this, but it's not the approach I've taken - essentially:
    - install any aarch64 libraries you need via the package manager on the Pi e.g. Apt
    - copy files from the Pi to create a "sysroot" folder
  - https://downloads.raspberrypi.org/raspios_lite_arm64/root.tar.xz
  - I prefer just building myself

[^1]: compared to when I started doing this in 2021 on the same machine, OOMs seem to have stopped happening - not sure if that's down to Hadrian, GHC itself, OS improvements or anything else
[^3]: in my case, at time of writing, the `PKGBUILD` file contains only trivial differences from [a previous version which I maintained myself](https://aur.archlinux.org/packages/aarch64-none-linux-gnu-gcc-10.3-bin) so that's more than enough for me personally
[^5]: and if not, then you're host is probably also `aarch64` in which case you don't need this guide at all (well, actually, unless the target isn't)
