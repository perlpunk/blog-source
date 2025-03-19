---
title: Generating rpm spec files for Perl Modules
date: 2025-03-20T21:57:40+01:00
tags: [software engineering, perl, rpm, yaml]
draft: false
---

## Introduction

To build a Perl module on openSUSE or other distributions, you need a file
with build instructions. For rpm that's usually a file called
`package-name.spec`.

See also [General packaging
guidelines](https://en.opensuse.org/openSUSE:Packaging_guidelines) and [the
Perl packaging guidelines](https://en.opensuse.org/openSUSE:Packaging_Perl).

In [Open Build Service devel:languages:perl](https://build.opensuse.org/project/show/devel:languages:perl) we currently have over 3100 perl packages.
That would not have been possible without some automation.

In this post I will describe how
[cpanspec](https://github.com/openSUSE/cpanspec) can create spec files
automatically, and how it can even update spec files for a new release without
overriding manual changes from before.

I will also mention some differences to other languages and specifically
[py2pack](https://github.com/openSUSE/py2pack).

Another similar script is [gem2rpm](https://github.com/fedora-ruby/gem2rpm) for
ruby.

## Anatomy of a Perl Module 🦴

A perl module on [CPAN](https://www.cpan.org/) is actually called a
distribution. It can contain more than one module, in this case referring to
perl package names and files.

I will describe the common structure here, but there can be exceptions to
basically everything. As an example I will show
[YAML::PP on CPAN](https://metacpan.org/dist/YAML-PP) / [perl-YAML-PP in
OBS](https://build.opensuse.org/package/show/devel:languages:perl/perl-YAML-PP).

The CPAN server does not enforce any rules for the uploaded tarballs. But
the build tools which are creating those distributions ensure some minimum
common structure for the large majority of perl distributions out there.

That is not the case for some other languages, which makes automating packaging
harder for them.

### Source 📝

The source is uploaded as a tarball (`tgz`, `tar.gz`, but can also be `bz2` or
`zip`).

The tarball for `YAML::PP` is called `YAML-PP-v0.39.0.tar.gz`.

Every distribution consists of one or more modules, and there is usually
one main module the distribution gets its name and version from.

The tarball standard name is `Main-Module-version.tar.gz`, and that's the case
for over 98% of the distributions, I'd say.

In python, the tarballs sometimes use underscores, sometimes hyphens, which
already requires manual adjustments.

### Files 📁

The most common layout is something like this:

    lib/Main/Module.pm
    lib/Main/Module/Other.pm
    t/some-test.t
    META.yml
    META.json
    Makefile.PL
    Changes
    LICENSE
    MANIFEST

### Documentation 📖

Perl modules are documented with [Pod - Plain Old
Documentation](https://en.wikipedia.org/wiki/Plain_Old_Documentation).

The most common and recommended layout of the documentation for modules,
especially for the main module, is:

* `NAME`: Main::Module - Abstract
* `SYNOPSIS`: Short code examples
* `DESCRIPTION`: Short description
* Other sections going into more details
* `LICENSE`

There seems to be no such standard for python modules, for example, so
`py2pack` just takes the whole README file, which always has to be shortened
for the spec file manually.

### Meta Data

Most modules except ancient ones have a `META.yml` and/or `META.json` file with
useful meta data, like the license, abstract, distribution name, author,
dependencies, provided package names, homepage, bugtracker.

### License ⚖️

The License can be in several places. There's the LICENSE file, there is the
LICENSE section in the Pod, and it should also be in the META files.

If it's in the META files, it's usually a short identifier we can use (more or
less) directly.
Otherwise cpanspec uses heuristics to read the License text and generate a
short identifier from it automatically.

See also [SPDX Licence List](https://spdx.org/licenses/).

For example `py2pack` can not do this at this time, so it needs to be adjusted
manually.

## Running cpanspec on a new module

If you don't use `cpan` or `cpanm` on your machine, you have to download the
package index first:

    % curl -O http://www.cpan.org/modules/02packages.details.txt.gz

Then download the tarball. It is also possible to specify the module name,
but for demonstration purposes we use the tarball directly.

    % curl -O https://cpan.metacpan.org/authors/id/T/TI/TINITA/YAML-PP-v0.38.0.tar.gz

Then generate the spec file:

    % cpanspec --pkgdetails 02packages.details.txt.gz YAML-PP-v0.38.0.tar.gz
    ...
    cat perl-YAML-PP.spec
    #
    # spec file for package perl-YAML-PP
    #
    # Copyright (c) 2025 SUSE LLC
    #
    ...

For the first generation, it will also create a `perl-YAML-PP.changes` for you.
You can skip that with `--skip-changes`.

To get more information about the metadata that `cpanspec` found, use the `-v` switch.

### The generated spec file

Let's go through the file step by step:

    %define cpan_name YAML-PP
    Name:           perl-YAML-PP
    Version:        0.38.0
    Release:        0

The license in the META file says `perl_5`, so `cpanspec` uses this SPDX
identifier automatically:

    License:        Artistic-1.0 OR GPL-1.0-or-later

Also from META:

    Summary:        YAML 1.2 Processor

Generic url and source for all modules:

    URL:            https://metacpan.org/release/%{cpan_name}
    Source0:        https://cpan.metacpan.org/authors/id/T/TI/TINITA/%{cpan_name}-%{version}.tar.gz

`cpanspec` uses some heuristics to decide if it's a noarch build (no .c files
etc.):

    BuildArch:      noarch

Common requirements:

    BuildRequires:  perl
    BuildRequires:  perl-macros

Requirements from module meta data:

    BuildRequires:  perl(Module::Load)
    BuildRequires:  perl(Test::More) >= 0.98
    BuildRequires:  perl(Test::Warn)
    Requires:       perl(Module::Load)

Common macro:

    %{perl_requires}

From module Pod:

    %description
    YAML::PP is a modular YAML processor.

    It aims to support 'YAML 1.2' and 'YAML 1.1'. See https://yaml.org/. Some
    (rare) syntax elements are not yet supported and documented below.

    ...

Common preparation:

    %prep
    %autosetup  -n %{cpan_name}-%{version} -p1

The build section can vary depending on the used build script, detected
automatically by `cpanspec`:

    %build
    perl Makefile.PL INSTALLDIRS=vendor
    %make_build

    %check
    make test

Common:

    %install
    %perl_make_install
    %perl_process_packlist
    %perl_gen_filelist

`cpanspec` can detect documentation based on some heuristics:

    %files -f %{name}.files
    %doc Changes CONTRIBUTING.md examples Makefile.dev README.md
    %license LICENSE

    %changelog

Everything is correct! We're done 😃

### Conclusion

In most cases it is as easy as that. But thanks to the large number of modules,
there are still many cases where we have to adjust the abstract, license,
requirements, description etc.

## Updating a spec file for a new version

If we didn't have to make manual changes in the original generation, it is as
easy as this:

    % curl -O https://cpan.metacpan.org/authors/id/T/TI/TINITA/YAML-PP-v0.39.0.tar.gz
    % cpanspec --force --pkgdetails 02packages.details.txt.gz YAML-PP-v0.39.0.tar.gz

The `--force` switch is needed to automatically overwrite the existing spec
file.

Let's look at the diff:

```diff
diff --git a/perl-YAML-PP.changes b/perl-YAML-PP.changes
index 4fa718b..cc8cbac 100644
--- a/perl-YAML-PP.changes
+++ b/perl-YAML-PP.changes
@@ -1,3 +1,9 @@
+-------------------------------------------------------------------
+Thu Mar 20 14:28:45 UTC 2025 - Tina Müller <tina.mueller@suse.com>
+
+- updated to 0.39.0
+   see /usr/share/doc/packages/perl-YAML-PP/Changes
+
 -------------------------------------------------------------------
 Wed Mar 19 14:46:52 UTC 2025 - Tina Müller <tina.mueller@suse.com>
 
diff --git a/perl-YAML-PP.spec b/perl-YAML-PP.spec
index a5a54ed..acced96 100644
--- a/perl-YAML-PP.spec
+++ b/perl-YAML-PP.spec
@@ -18,7 +18,7 @@
 
 %define cpan_name YAML-PP
 Name:           perl-YAML-PP
-Version:        0.38.0
+Version:        0.39.0
 Release:        0
 License:        Artistic-1.0 OR GPL-1.0-or-later
 Summary:        YAML 1.2 Processor
```

That's easy, and if there were any changes like requiring a new module
or a newer version of it, it would automatically be generated.

## Doing manual changes

Let's assume we have to adjust some fields for `YAML::PP`, because they are
wrong.

We could manually adjust the spec file. But then our changes would be
overwritten with the next version.

Instead we use a configuration file named `cpanspec.yml`.

### Editing `cpanspec.yml`

```yaml
---
patches:
  some-bug-in-0.38.patch: -p1

summary: Improved abstract here

# Leftover file in tarball
skip_doc: Makefile.old

license: Actual-license

# A module might have a non-perl dependency
preamble:
  BuildRequires: libfoo
```

We can now regenerate the spec, and it will automatically read `cpanspec.yml`,
overwrite the fields and add it to the sources:

    % cpanspec --force --skip-changes --pkgdetails 02packages.details.txt.gz YAML-PP-v0.38.0.tar.gz

```diff
diff --git a/perl-YAML-PP.spec b/perl-YAML-PP.spec
index a5a54ed..398ff1a 100644
--- a/perl-YAML-PP.spec
+++ b/perl-YAML-PP.spec
@@ -20,10 +20,13 @@
 Name:           perl-YAML-PP
 Version:        0.38.0
 Release:        0
-License:        Artistic-1.0 OR GPL-1.0-or-later
-Summary:        YAML 1.2 Processor
+#Upstream: Artistic-1.0 or GPL-1.0-or-later
+License:        Actual-license
+Summary:        Improved abstract here
 URL:            https://metacpan.org/release/%{cpan_name}
 Source0:        https://cpan.metacpan.org/authors/id/T/TI/TINITA/%{cpan_name}-v%{version}.tar.gz
+Source1:        cpanspec.yml
+Patch0:         some-bug-in-0.38.patch
 BuildArch:      noarch
 BuildRequires:  perl
 BuildRequires:  perl-macros
@@ -32,6 +35,9 @@ BuildRequires:  perl(Test::More) >= 0.98
 BuildRequires:  perl(Test::Warn)
 Requires:       perl(Module::Load)
 %{perl_requires}
+# MANUAL BEGIN
+BuildRequires:  libfoo
+# MANUAL END

 %description
 YAML::PP is a modular YAML processor.
```

There are many more fields you can adjust with this.
See this [example file](https://github.com/openSUSE/cpanspec/blob/master/cpanspec.yml)
with all supported fields.

This way we will never have to touch the spec file itself, and it will
survive any version update.

(There can be exceptions, when you have to make changes that `cpanspec.yml`
does not support, but it's rare.)

## Perl Module Versions

Perl module versions are a bit special. The traditional versions are simply
decimals, unlike semantic versions we know from elsewhere. In the example
above the module used the new version format with two dots and a `v` in front,
which behave like we expect from semantic versions.

In the [article about updating CPAN
modules](../2024-04-how-suse-cpan-modules-are-updated/) I explained it more
detailed.

`cpanspec` is currently getting a feature to "normalize" the old decimal
versions, but it's not yet released at the time of this writing.

