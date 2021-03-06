---
layout: page
title: Reproducible Builds

---


F-Droid supports [reproducible builds](https://reproducible-builds.org)
of apps, so that anyone can run the build process again and reproduce
the same APK as the original release.  This means that F-Droid can
verify that an app is 100% free software while still using the original
developer's APK signatures.  Ideally, all of the built APKs will have
the exact same hash, but that is a more difficult standard with less
payoff. Right now, F-Droid verifies reproducible builds using the APK
signature.

This concept is occasionally called "deterministic builds".  That is a
much stricter standard: that means that the whole process runs with
the same ordering each time.  The most important thing is that anyone
can run the process and end up with the exact same result.


### How it is implemented as of now

Publishing signed binaries from elsewhere (e.g. the upstream developer)
is now possible after verifying that they match ones built using a
recipe. Publishing only takes place if there is a proper match. (Which
seems very unlikely to be the case unless the exact same tool-chain is
used, so I would imagine that unless the person building and signing
the incoming binaries uses fdroidserver to build them, probably the
exact same buildserver id, they will not match. But at least we have
the functionality to support that.)

This procedures are implemented as part of `fdroid publish`. At the
publish step, the reproducibility check will follow this logic:

![Flow-chart for reproducibility check]({{ site.baseurl }}/assets/docs/reproducible-builds/publish.png)


#### Publish both (upstream-)developer signed and F-Droid signed APKs

Use this approach for shipping a version of an app, with both
(upstream-)developer signed and F-Droid signed APKs. This enables us
to ship updates for users who installed apps from other sources than
F-Droid (eg. Play Store) which are therefore signed by the
app-developers, while also shipping updates for apps which were built
and signed by F-Droid.

This requires to put (upstream-)developer signatures into fdroiddata.
We provide a command for easily extracting signatures from APKs:

```console
$ cd /path/to/fdroiddata
$ fdroid signatures F-Droid.apk
```

You may also supply HTTPS-URLs directly to `fdroid signatures` instead
of local files. The signature files are extracted to the according
metadata directory ready to be used with `fdroid publish`. A signature
consists of 3 files and the result of extracting one will resemble this
file listing:

```console
$ ls metadata/org.fdroid.fdroid/signatures/1000012/
CIARANG.RSA  CIARANG.SF  MANIFEST.MF
```

#### Exclusively publishing (upstream-)developer signed APKs

For using this older approach, everything in the metadata should be the
same as normal, with the addition of the `Binaries:` directive to
specify where to get the binaries from. In this case F-Droid will never
attempt to ship APKs signed by F-Droid. Should `fdroid publish` manage
to verify that a downloaded APK can be built reproducibly, the
downloaded APK will be published. Otherwise F-Droid will skip
publishing this version of the app.

Here is an example for a _Binaries_ directive:

```yaml
Binaries: https://foo.com/path/to/myapp-%v.apk
```

Also see: [Build Metadata Reference - Binaries](../Build_Metadata_Reference/#Binaries)


### Verification builds

Many people or organizations will only be interested in reproducing
builds to make sure that the f-droid.org builds match the original
source and nothing has been inserted in.  In that case, the resulting
APKs are not published for installation.  The
[Verification Server](../Verification_Server) automates this process.


### Reproducible Builds

An awful lot of builds already verify with no extra effort since Java
code is often compiled into the same bytecode by a wide range of Java
versions.  The Android SDK's _build-tools_ will create differences in
the resulting XML, PNG, etc. files, but this is usually not a problem
since the _build.gradle_ includes the exact version of _build-tools_
to use.

Anything built with the NDK will be much more sensitive.  For example,
even for builds that use the exact same version of the NDK
(e.g. _r13b_) but on different platforms (.e.g OSX version Ubuntu), the
resulting binaries will have differences.

Additionally, we'll have to look out for anything that includes
timestamping information, is sensitive to sort order, etc.

Google is also working towards reproducible builds of Android apps, so
using recent versions of the Android SDK helps.  One specific case is
starting with Gradle Android Plugin v2.2.2, timestamps in the APK
file's ZIP header are automatically zeroed out.


## _platform_ Revisions

The Android SDK tools
[were changed](https://issuetracker.google.com/issues/37132313) in
2014 to
[stick two](https://android.googlesource.com/platform/frameworks/base/+/ad2d07d%5E!/)
[data elements](https://android.googlesource.com/platform/frameworks/base/+/5283fab%5E!/)
in _AndroidManifest.xml_ as part of the build process:
`platformBuildVersionName` and `platformBuildVersionCode`.
`platformBuildVersionName` includes the "revision" of the _platforms_
package built against (e.g. _android-23_), however different
"revisions" of the same _platforms_ package cannot be installed in
parallel.  Plus the SDK tools do not support specifying the required
revision as part of the build process.  This often results in an
otherwise reproducible build where the only difference is the
`platformBuildVersionName` attribute.

The "_platform_" is part of the Android SDK that represents the
standard library that is installed on the phone. They have two parts
to their version: "version code", which is an integer that represents
the SDK release, and the "revision", which represents bugfix versions
to each platform. These versions can be seen in the included
_build.prop_ file. Each revision has a different number in
_ro.build.version.incremental_.  Gradle has no way to specify the
revision in _compileSdkVersion_ or _targetSdkVersion_. Only one
"_platform-23_" can be installed at a time, unlike _build-tools_, where
every release can be installed in parallel.

Here are two examples where I think all the differences came from just
different revisions of the platform:

* https://verification.f-droid.org/de.nico.asura_12.apk.diffoscope.html
* https://verification.f-droid.org/de.nico.ha_manager_25.apk.diffoscope.html


### PNG Crush/Crunch

A standard part of the Android build process is to run some kind of
PNG optimization tool, like `aapt singleCrunch` or `pngcrush`.  These
do not provide deterministic output, it is still an open question as
to why.  Since PNGs are normally committed to the source repo, a
workaround to this problem is to run the tool of your choice on the
PNG files, then commit those changes to the source repo (e.g. `git`).
Then, disable the default PNG optimization process by adding this to
_build.gradle_:

```
android {
    aaptOptions {
        cruncherEnabled = false
    }
}
```


### Build Server IDs

To describe the build environment used by F-Droid builds, APKs have two files inserted into them:

* _META-INF/fdroidserverid_ - git commit hash of [_fdroidserver_](https://gitlab.com/fdroid/fdroidserver) used for the build
* _META-INF/buildserverid_ - git commit hash of [_makebuildserver_](https://gitlab.com/fdroid/fdroidserver/blob/master/makebuildserver) used for the build

To ensure reproducibility, use the exact same revision of the
`./makebuildserver` and `fdroid build`. You can find the commit hash
of _fdroidserver_ by going to your git clone and running `git log
-n1`.  The build server instance is stamped with the git commit hash on
creation, and that ID is included in builds.


### ZIP entry info

The ZIP format was originally designed around the MSDOS FAT
filesystem.  UNIX file permissions were added as an extension.  APKs
only need the most basic ZIP format, without any of the extensions.
These extensions are often stripped out in the final release signing
process.  But the APK build process can add them.  For example:

```diff
--- a2dp.Vol_137.apk
+++ sigcp_a2dp.Vol_137.apk
@@ -1,50 +1,50 @@
--rw----     2.0 fat     8976 bX defN 79-Nov-30 00:00 AndroidManifest.xml
--rw----     2.0 fat  1958312 bX defN 79-Nov-30 00:00 classes.dex
--rw----     1.0 fat    78984 bx stor 79-Nov-30 00:00 resources.arsc
+-rw-rw-rw-  2.3 unx     8976 b- defN 80-000-00 00:00 AndroidManifest.xml
+-rw----     2.4 fat  1958312 b- defN 80-000-00 00:00 classes.dex
+-rw-rw-rw-  2.3 unx    78984 b- stor 80-000-00 00:00 resources.arsc
```


### Migration to reproducible builds

#### TODO

- NDK inserts changing _build-id_, probably via `ld`
- jar sort order for APKs
- `aapt` versions produce different results (XML and res/ subfolder names)


#### Sources

- <https://gitlab.com/fdroid/fdroidserver/commit/8568805866dadbdcc6c07449ca6b84b80d0ab03c>
- [Verification Server](../Verification_Server)
- <https://verification.f-droid.org>
- <https://reproducible-builds.org>
- <https://wiki.debian.org/ReproducibleBuilds>
- <https://gitian.org/>
- [Google Issue #70292819 platform-27\_r01.zip was overwritten with a new update](https://issuetracker.google.com/issues/70292819) (_Google login and Javascript required_)
- [Google Issue #37132313 platformBuildVersionName makes builds difficult to reproduce, creates unneeded diffs](https://issuetracker.google.com/issues/37132313) (_Google login and Javascript required_)
- [Google Issue #110237303 resources.arsc built with non-determism, prevents reproducible APK builds](https://issuetracker.google.com/issues/110237303) (_Google login and Javascript required_)
