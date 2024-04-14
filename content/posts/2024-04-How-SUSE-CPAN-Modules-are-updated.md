---
title: How Perl CPAN Modules Are Updated in SUSE
date: 2024-04-14T21:18:03+02:00
tags: [software engineering, open build service, perl]
draft: false
---

## Introduction

Packages for SUSE distributions are built in the [Open Build
Service (OBS)](https://build.opensuse.org/).

In my previous posts I showed how [a perl module is built in OBS](../2024-04-building-a-perl-module-in-obs/) and how [you can use it as a CI](../2024-04-obs-as-a-ci-for-perl-modules/).

Now I want to show how we at SUSE maintain modules that are part of an official openSUSE release.

In the [`devel:languages:perl`](https://build.opensuse.org/project/show/devel:languages:perl)
repository we currently have over 3100 perl modules from CPAN.

Not all of them land in the official SUSE distribution repositories though.
[openSUSE:Factory](https://build.opensuse.org/project/show/openSUSE:Factory)
is the repository which tracks those packages. In there we currently have
almost 1400 CPAN modules.

There are too many CPAN uploads to all do them manually in OBS.

So we have an automated process for updating. One part is the script to generate spec files, the other one is to put the new modules in a OBS subproject.

## Automatically generating a .spec file for a CPAN module

[cpanspec](https://github.com/openSUSE/cpanspec) is the script which we use to generate the .spec file from a CPAN tarball.
The git log goes back to 2011, the Changes file to 2006.

It tries to find out as many information as it can from the included META files, and also looks into the documentation.

Some of this is easy, but there are some challenges. We need to find out:
* The distribution and module version(s)
* The module(s) it provides
* The module title, summary and description
* The license
* Which files are supposed to be documentation files
* Which file contains the changelog, for generating a changelog entry ourselves if possible
* Whether the build needs to be architecture specific
* The requirements (other CPAN modules)

Sometimes the module does not provide all necessary information.
A common case is that it's missing a license 😢 or missing dependencies.
Also any non-perl dependencies cannot be handled with CPAN META files currently.

For those cases we have a [`cpanspec.yml`](https://github.com/openSUSE/cpanspec/blob/master/cpanspec.yml) file which we can add to a module to configure manual overrides.
This is also very important when we want to apply patches that haven't been included upstream.
The file is very useful because any manual changes to the spec file would be
overwritten with the next release.

I collected all cpanspec.yml files into [one big file recently](https://github.com/perlpunk/cpan-meta/blob/main/cpanspec.yaml).

### Versions

One of the bigger challenges is the versioning.

At the time of this writing, most perl modules in SUSE have the literal version from CPAN in the spec.

Why is this problematic? CPAN modules use decimal versions. If you know basic maths, they are easy to understand.

One module release can be `3.19`, and the next would be `3.2`, which can also be written `3.20`, which is greater than `3.19`.

However, for rpm `3.19` is greater than `3.2`.
😵‍💫

When the module author is using the good practice to always use the same
number of digits after the dot, e.g. `3.20`, then rpm would be fine with
that.
But if they don't, in those cases we have to adjust the version manually.
For the next release, e.g. `3.21` it's going to be fine again usually.

We are currently in the process of converting the versions to the normalized form, which can be generated with the builtin version module:
`version->parse($cpan_version)->normal`. Here are some examples:

```
3.14    -> 3.140.0
3.140   -> 3.140.0
3.014   -> 3.14.0
3.001   -> 3.1.0
3.14159 -> 3.141.590
```

Note that the normalized version is divided into triplets. That means for
versions with more than four decimals the normalized version is lower.

Because of that, in the transition period we will have some inconsistencies. The problem is not so much the version of the module itself, but other modules requiring a specific minimum version of it.

There are not many distributions which are using the normalized format. For
example here is the [Repology page for
Mojolicious](https://repology.org/project/perl:mojolicious/versions).

Repology can't even cope with that format and says that `9.360.0` is
bigger than the CPAN version `9.36` and because of that it's marked as
"untrusted".

### Dependencies

Dependencies can be specified in the `META.json` or `META.yml` file, but also just in the Makefile.PL.

If you specify `dynamic: 0` in your metadata, then tools know that they
don't have to call `Makefile.PL` (or other build scripts) to find out
further information. In many cases that's true. The most common case where
you need to call `Makefile.PL` is dependencies that depend on the platform.

So if `dynamic` is set to 1, we call the build script and then read the generated `MYMETA.json`/`MYMETA.yml` after it.


## Updating modules in OBS automatically

We have a
[script](https://github.com/openSUSE/cpanspec/blob/master/bin/update-autoupdate.sh)
that runs every day and fetches new releases from CPAN.

It checks out the perl package from
[`devel:languages:perl`](https://build.opensuse.org/project/show/devel:languages:perl),
generates the spec with `cpanspec`, updates the Changes file and put it into the
[`devel:languages:perl:autoupdate`](https://build.opensuse.org/project/show/devel:languages:perl:autoupdate)
project.

From there we make so called "Submit requests" to `devel:languages:perl`, currently
manually. Here is an [example](https://build.opensuse.org/request/show/1160761).

## Building all CPAN modules in OBS

If you take a closer look at the [subprojects of
`devel:languages:perl`](https://build.opensuse.org/project/subprojects/devel:languages:perl),
you will find a subproject for each letter, for example
[CPAN-Y](https://build.opensuse.org/project/show/devel:languages:perl:CPAN-Y).

We indeed update all CPAN realeases. However, not all modules will be able to
build, because we only take into account `devel:languages:perl` as a repository
for using dependencies from. So a module that only has dependencies that are in
`devel:languages:perl` will be able to build, others will show as "unresolvable".

And if the module has external non-perl dependencies, it of course also cannot
build.



