---
title: The Annual Perl Toolchain Summit
date: 2024-05-07T17:13:50+02:00
tags: [software engineering, perl, toolchain, conference, yaml]
draft: false
---

## Introduction

The [Perl Toolchain Summit](https://perltoolchainsummit.org/pts2024/) is an annual
event where around 30-35 people are gathering to work on toolchain related
tasks. The topics are the [Perl Authors Upload Server](https://pause.perl.org/),
[MetaCPAN](https://metacpan.org/), CPAN clients, Testing and Coverage modules,
Smoketesting Perl, CPAN Security, Cyber Resilience Act, Module Dependencies,
and many more.

This year it took place in Lisbon, Portugal from April 25-28.

It's a mix of discussions, pair programming and presentations. All attendees
are usually in the same hotel where the conference is happening, so during
those four days a useful discussion can start any time. It's crucial to
be in the same room(s) for many attendees.

This time, my employer [SUSE](https://www.suse.com/) sponsored my travel and
accomodation to go there. Thanks!

This was my sixth Toolchain Summit, including my first [in
2015](http://perltoolchainsummit.org/qa2015/) in Berlin which I organized
together with Andreas König.

Below I write about what I worked on at the PTS, but you can also check out the
[blog posts from other
attendees](https://perltoolchainsummit.org/pts2024/wiki?node=Blogs).

### YAML

My topic has been mosty YAML for all the Summits. While it's not strictly
really toolchain related, YAML modules in Perl are important, and many other
modules depend on one of them.

This time I looked into some issues reported by
[OSS-Fuzz](https://oss-fuzz.com/) for
[libyaml](https://github.com/yaml/libyaml).
[YAML::XS](https://metacpan.org/dist/YAML-LibYAML), a binding to libyaml, is a
widely used module.

The reported issues were security relevant regarding loading and dumping
untrusted YAML documents. There had also been reported some issues a few weeks
before the Summit.

I already was able to find out before the summit that one of the issues was not
an issue with libyaml itself, but with the fuzzing code. I made a
[pull request to oss-fuzz](https://github.com/google/oss-fuzz/pull/11818) that
got merged pretty soon. This PR also allowed to fuzztest much more input,
because the code had been skipping a lot of test cases before because of the
bug I fixed.

Then we got two more issues reported shortly before the summit, and I had to
look into some libyaml C code I had never looked at before. So I learned a bit
of C here :)

It turns out, those two issues where also resulting from mistakes in the fuzzing
code, so I created two more pull requests
[oss-fuzz#11840](https://github.com/google/oss-fuzz/pull/11840) and
[oss-fuzz#11848](https://github.com/google/oss-fuzz/pull/11848). Those were
also merged quickly.

So the fuzztesting improved. And just a few days after the Summit we got
another report for libyaml, and this time a valid one, showing a parsing bug in
libyaml, and a potential stack overflow in the emitter.

I managed to fix the parsing bug
[yesterday](https://github.com/yaml/libyaml/pull/295).  The stack overflow in
the emitter is still unclear, but with the fixed parser it won't happen
anymore. The default nesting limit will prevent this specific case now.

I will do a libyaml release hopefully in the next weeks, and also update
YAML::XS.

### Perl Module Versions

As part of the Cyber Resilience Act / SBOM discussion, I gave a short talk on
perl module versions.

We want to introduce a language independent way of specifying module names,
for example with [Package URLs](https://github.com/package-url/purl-spec).

Related to that is a way to specify what version(s) of a module an
application or another module depends on. And comparing version numbers
against each other works a bit different in perl than in most other languages.

So I gave a [short talk](https://www.youtube.com/watch?v=BrB9_VZxIVw) about it,
and I also recently wrote [a blog
post](../2024-04-how-suse-cpan-modules-are-updated/) related on that topic.

Versions in perl are simply decimal numbers, so `0.9` would actually be greater
than `0.89`. But you can also use versions with more than one dot, e.g.
`3.14.15`, which are not decimal.

The normalized versions of `0.89` and `0.9` would be `0.890.0` and `0.900.0`.

And that format is what most other languages and package managers use and
understand. So it might be a good idea to use those normalized versions when
using a language independent package url, so it won't be necessary to use
different algorithms for checking version requirements of an application.

### Perl Module Metadata

As I'm responsible for updating new CPAN releases in
[devel:languages:perl](https://build.opensuse.org/project/show/devel:languages:perl)
for openSUSE, I maintain [cpanspec](https://github.com/openSUSE/cpanspec),
a script to extract all kinds of metadata from a module tarball.

It runs `Makefile.PL` if necessary to get dynamic metadata from the
`MYMETA.(json|yml)` files, otherwise from `META.(json|yml)`.

Ideally all information should come from the META files, however in reality
that's often not the case, so it also looks into other files and the pod
documentation to find out about the license, description etc.

Additionally to that, sometimes there is simply missing information, for example
a missing dependency. For that we have a `cpanspec.yml` file for each module,
where we specify such "errata". Also it's necessary for specifying extra
dependencies, for example on C libraries, that cannot be specified in a
Perl module currently.

cpanspec merges the data from reading the tarball information plus the
`cpanspec.yml` file and automatically generates the `.spec` file for rpm.

Now at the PTS we discussed with a handful of people that it would be nice
if not every vendor would have to maintain their own script and errata,
but that we rather share some of this.

As a first step I collected all `cpanspec.yml` and `.spec` data we have
into a [cpan-meta repository](https://github.com/perlpunk/cpan-meta).

As a next step, I would like to factor out some of the cpanspec
functions into a module that can be released to CPAN.

#### Running cpanspec

Currently, the script to update CPAN moduled is run daily on one of our
servers at SUSE.

It also updates **all** CPAN modules in the specific [CPAN-A](https://build.opensuse.org/project/show/devel:languages:perl:CPAN-A) etc. subprojects. This part still needs to
run on one of our servers for now, because of technical reasons (it needs to
keep a big status file of all modules).

But I now started to move the update of
[devel:languages:perl](https://build.opensuse.org/project/show/devel:languages:perl)
into a GitHub action. This makes it more transparent, and every CPAN author
would be able to look into the log and see why there could be a problem with a
specific module.

The work is in [this
branch](https://github.com/perlpunk/cpanspec/tree/update-perl-gh) so far.  I
also started a container image
`registry.opensuse.org/home/tinita/cpanspec/container/containers/dlp-autoupdate:latest`
that has all dependencies for cpanspec and the automated update installed. The
Dockerfile comes from the [Open Build
Service](https://build.opensuse.org/package/show/home:tinita:cpanspec:container/autoupdate)
and is also maintained in my branch currently.


## Sponsors

Thanks to the tireless organizing team for making this happen again!

And thanks to the following organizations, companies and people to support
this financially:

Monetary sponsors:
[Booking.com](http://www.booking.com/),
[The Perl and Raku Foundation](https://www.perlfoundation.org/),
[Deriv](https://deriv.com/?utm_source=perltoolchainsummitdotorg&utm_medium=pr&utm_campaign=sponsorship&utm_content=homepage_logo),
[cPanel, Inc](https://cpanel.net/)
[Japan Perl Association](https://japan.perlassociation.org/),
[Perl-Services](https://www.perl-services.de/),
[Simplelists Ltd](https://www.simplelists.com/),
[Ctrl O Ltd](https://www.ctrlo.com/),
[Findus Internet-OPAC](https://www.findus-internet-opac.de/),
Harald Joerg,
Steven Schubiger.

In kind sponsors:
[Fastmail](https://www.fastmail.com/),
[Grant Street Group](https://www.grantstreet.com/),
[Deft](https://deft.com/),
[Procura](https://www.procura.nl/),
[Healex GmbH](https://healex.systems/),
[SUSE](https://www.suse.com/),
[Zoopla](https://www.zoopla.co.uk/).
