B2-RC-Depends
=============

This is a sample project to demonstrate an issue with
[B2](https://www.bfgroup.xyz/b2/) and dependency scanning of `.rc` files, which
was reported in bfgroup/b2#420.

The problem is that in some cases, header files that are included in such `.rc`
files are not picked up. In particular, when the header is in a different
directory, and referenced by an absolute path.

This problem came up in our use case, because in our proprietary project we
generate a `revision.h` header into a stage directory containing the current
revision number, and this file is referenced by its absolute path by a `Jamfile`.

The same `revision.h` file is included in both a `.cpp` file and a `.rc` file,
but if the `revision.h` file is touched, _only_ the `.cpp` file gets rebuilt!

Laout of the sample project
---------------------------

The top-level contains:

* `main.cpp`, which includes `revision.h` and prints the value of `REVISION` on
  stdout.
* `version.rc`, which has a Windows-style `VERSIONINFO` block, and also
  includes `revision.h`, to get at the `REVISION` value.
* `Jamfile`, which references `main.cpp`, a `revision//revision` project, and a
  `version.rc` requirement for Windows.

The `revision` subdirectory contains:

* `revision.h`, which defines `REVISION` to 42.
* `Jamfile`, which declares `revision` as an `alias`, with `usage-requirements`
  set to `<include>` the _absolute_ path of the `revision` subdirectory.

Building the sample project
---------------------------

This project can be built anywhere B2 runs, but to specifically see the
issue with `.rc` files, it must be built on Windows.

If it is built from scratch, there is nothing really special (some
output elided):

```
msvc.compile.rc bin\msvc-14.3\debug\threading-multi\version_res.obj

    call "bin\standalone\msvc\msvc-14.3\msvc-setup.bat"  >nul
 rc /nologo -l 0x409   -I"D:\rc-depends\revision" -fo "bin\msvc-14.3\debug\threading-multi\version_res.obj" "version.rc"

compile-c-c++ bin\msvc-14.3\debug\threading-multi\main.obj

    call "bin\standalone\msvc\msvc-14.3\msvc-setup.bat"  >nul
 cl /Zm800 -nologo "main.cpp" -c -Fo"bin\msvc-14.3\debug\threading-multi\main.obj"     -TP /wd4675 /EHs /GR /Zc:throwingNew /Z7 /Od /Ob0 /W3 /MDd /Zc:forScope /Zc:wchar_t /Zc:inline /favor:blend "-ID:\rc-depends\revision"

main.cpp
msvc.link bin\msvc-14.3\debug\threading-multi\rc-depends.exe

        call "bin\standalone\msvc\msvc-14.3\msvc-setup.bat"  >nul
 link /NOLOGO /INCREMENTAL:NO "bin\msvc-14.3\debug\threading-multi\main.obj" "bin\msvc-14.3\debug\threading-multi\version_res.obj"      /DEBUG /MANIFEST:EMBED /MACHINE:X64 /subsystem:console /out:"bin\msvc-14.3\debug\threading-multi\rc-depends.exe"
```

As expected, it compiles both the `.rc` file and the `.cpp` file, then
links them together.

Now if you touch the `revision\revision.h` file, for example by changing
`REVISION` to `43`, _only_ the `.cpp` file gets rebuilt:

```
compile-c-c++ bin\msvc-14.3\debug\threading-multi\main.obj

    call "bin\standalone\msvc\msvc-14.3\msvc-setup.bat"  >nul
 cl /Zm800 -nologo "main.cpp" -c -Fo"bin\msvc-14.3\debug\threading-multi\main.obj"     -TP /wd4675 /EHs /GR /Zc:throwingNew /Z7 /Od /Ob0 /W3 /MDd /Zc:forScope /Zc:wchar_t /Zc:inline /favor:blend "-ID:\rc-depends\revision"

main.cpp
msvc.link bin\msvc-14.3\debug\threading-multi\rc-depends.exe

        call "bin\standalone\msvc\msvc-14.3\msvc-setup.bat"  >nul
 link /NOLOGO /INCREMENTAL:NO "bin\msvc-14.3\debug\threading-multi\main.obj" "bin\msvc-14.3\debug\threading-multi\version_res.obj"      /DEBUG /MANIFEST:EMBED /MACHINE:X64 /subsystem:console /out:"bin\msvc-14.3\debug\threading-multi\rc-depends.exe"
```

If you look at `b2 -d+3` output, it shows that the `res-scanner` thinks
`revision.h` is "missing", while that does not apply to the `c-scanner`:

```
...
make    --         <p.-object(c-scanner)@220>main.cpp
make    --         <p.-object(c-scanner)@220>main.cpp
bind    --         <p.-object(c-scanner)@220>main.cpp: main.cpp
time    --         <p.-object(c-scanner)@220>main.cpp: 2024-11-18 20:19:39.990406100 +0000
make    --          <p.-object(c-scanner)@220>main.cpp
make    --          <p.-object(c-scanner)@220>main.cpp
time    --          <p.-object(c-scanner)@220>main.cpp: unbound
make    --           <object(c-scanner)@220>iostream
make    --           <object(c-scanner)@220>iostream
bind    --           <object(c-scanner)@220>iostream: iostream
time    --           <object(c-scanner)@220>iostream: missing
made    stable       <object(c-scanner)@220>iostream
make    --           <object(c-scanner)@220#.>revision.h
make    --           <object(c-scanner)@220#.>revision.h
bind    --           <object(c-scanner)@220#.>revision.h: D:\rc-depends\revision\revision.h
time    --           <object(c-scanner)@220#.>revision.h: 2024-11-18 20:51:09.093000000 +0000
made*   newer        <object(c-scanner)@220#.>revision.h
made    stable     <p.-object(c-scanner)@220>main.cpp
...
make    --         <p.-object(res-scanner)@227>version.rc
make    --         <p.-object(res-scanner)@227>version.rc
bind    --         <p.-object(res-scanner)@227>version.rc: version.rc
time    --         <p.-object(res-scanner)@227>version.rc: 2024-10-24 13:57:53.714396200 +0000
make    --          <p.-object(res-scanner)@227>version.rc
make    --          <p.-object(res-scanner)@227>version.rc
time    --          <p.-object(res-scanner)@227>version.rc: unbound
make    --           <object(res-scanner)@227#.>revision.h
make    --           <object(res-scanner)@227#.>revision.h
bind    --           <object(res-scanner)@227#.>revision.h: revision.h
time    --           <object(res-scanner)@227#.>revision.h: missing
made    stable       <object(res-scanner)@227#.>revision.h
made    stable     <p.-object(res-scanner)@227>version.rc
```

Similar for `b2 -d+13` output:

```
fate change  <object(c-scanner)@220>iostream set to missing
fate change <object(c-scanner)@220>iostream to STABLE from missing, no actions, no dependencies and do not care
fate change  <object(c-scanner)@220#.>revision.h set to newer (by timestamp)
fate change  <p.-object(c-scanner)@220>main.cpp from newer to stable
fate change  <pbin\msvc-14.3\debug\threading-multi>main.obj set to old (by timestamp)
fate change  <object(res-scanner)@227#.>revision.h set to missing
fate change <object(res-scanner)@227#.>revision.h to STABLE from missing, no actions, no dependencies and do not care
fate change  <pbin\msvc-14.3\debug\threading-multi>rc-depends.exe from old to update
fate change  <pbin\msvc-14.3\debug\threading-multi>rc-depends.pdb from old to update
```

E.g., the `c-scanner` finds that `revision.h` was updated, while the
`res-scanner` finds `revision.h` missing, and then ignores it.
