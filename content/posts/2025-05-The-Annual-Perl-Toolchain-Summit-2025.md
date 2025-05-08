---
title: The Annual Perl Toolchain Summit 2025
date: 2025-05-08 15:45:58+02:00
tags: [software engineering, perl, toolchain, conference, yaml]
draft: false
---

## Introduction

The [Perl Toolchain Summit](https://perltoolchainsummit.org/pts2025/) is an annual
event where around 30-35 people are gathering to work on toolchain related
tasks. The topics are the [Perl Authors Upload Server](https://pause.perl.org/),
[MetaCPAN](https://metacpan.org/), CPAN clients, Testing and Coverage modules,
Smoketesting Perl, CPAN Security, Cyber Resilience Act, Module Dependencies,
and many more.

This year it took place in Leipzig, Germany from May 1-4.

It's a mix of discussions, pair programming and presentations. Like in the last
years, all attendees were in the same hotel where the conference was happening,
so during those four days a useful discussion can start any time. It's crucial
to be in the same room(s) for many attendees.

This year, my employer [SUSE](https://www.suse.com/) was a Silver sponsor.
Thanks!

Since I have been organizing this event in 2015 in Berlin with [Andreas
König](https://metacpan.org/author/ANDK), I have attended multiple times.

This time I helped organizing a little bit. The main local organizer,
[Daniel Boehmer](https://metacpan.org/author/DBOEHMER), had no experience with this conference, so I was helping
with my experience, suggesting what kind of venue we need, creating TODO lists,
some correspondence with the hotel.

The French Perl Mongers, mainly [Philippe
Bruhat](https://metacpan.org/author/BOOK) and [Laurent
Boivin](https://metacpan.org/author/ELBEHO), were organizing the financial
stuff and the event itself, regarding inviting people, topics, scheduling.

Below I write about what I worked on at the PTS, but you can also check out the
[blog posts from other
attendees](https://perltoolchainsummit.org/pts2025/wiki?node=Blogs) and the
[Results](https://perltoolchainsummit.org/pts2025/wiki?node=Results) wiki page.

At the event itself I had enough time for some coding. Daniel had everything
under control and provided us with everything we need.

### YAML::XS

My main project was bringing YAML 1.2 compatibility to
[YAML::XS](https://metacpan.org/pod/YAML::XS).

I started this already at [SUSE Hackweek
24](https://hackweek.opensuse.org/24/projects/create-object-oriented-api-for-perls-yaml-xs-module).

Another issue was that YAML::XS has been using package variables for
configuration, which can be changed from every place at any time in a program.

So I copied the whole functional interface to a new Object Oriented
interface. The functional interface will work as before.

After the hackweek there were some leftover issues, which I worked on
at the PTS.

One was fixing a segmentation fault, which was caused by a race condition
between an exception handler and the DESTROY method.

The other one was matching special values like booleans, null and numbers.
I had been using a call to Perl and a regular expression, but this is
way too slow, so I ported this to checking character by character in C.
This was taking the most time, but now the new interface is almost
as fast as the functional interface.

After getting the approval from [Ingy](https://metacpan.org/author/INGY) I
released [version
0.904.0](https://metacpan.org/release/TINITA/YAML-LibYAML-v0.904.0).

I will write another post going into more details about YAML::XS.

### YAML::Syck

I helped [Todd Rinaldo](https://metacpan.org/author/TODDR) with a little issue
in [YAML::Syck](https://metacpan.org/dist/YAML-Syck).

### Other

I attended the talk by [Paul Evans](https://metacpan.org/author/PEVANS) about
what's new in Perl 5.42.

I learned that with
[Sublike::Extended](https://metacpan.org/pod/Sublike::Extended) it is possible
already for older perl versions to write subroutines with named arguments.

[Chad Granum](https://metacpan.org/author/EXODIST) gave us a little informal
talk about a new [ORM](https://metacpan.org/dist/DBIx-QuickORM) he is working
on, which can be an alternative to
[DBIx::Class](https://metacpan.org/dist/DBIx-Class).

## Sponsors

Thanks to the following organizations, companies and people to support
this financially:

### Monetary Sponsors

[Booking.com](https://www.booking.com/), [WebPros](https://www.webpros.com/),
[CosmoShop](https://www.cosmoshop.de/), [Datensegler](https://datensegler.at/),
[OpenCage](https://opencagedata.com), [SUSE](https://www.suse.com/),
[Simplelists Ltd](https://www.simplelists.com/), [Ctrl O
Ltd](https://www.ctrlo.com/), [Findus
Internet-OPAC](https://www.findus-internet-opac.de/), [plusW
GmbH](https://www.plusw.de/)

### In-kind sponsors

[Grant Street Group](https://www.grantstreet.com/),
[Fastmail](https://www.fastmail.com/), [shift2](https://en.shift2.nl/),
[Oleeo](https://www.oleeo.com/), [Ferenc Erki](https://ferki.it/)

### Community Sponsors

[The Perl and Raku Foundation](https://www.perlfoundation.org/), [Japan Perl
Association](https://japan.perlassociation.org/), Harald Joerg, Alexandros
Karelas ([PerlModules.net](https://www.perlmodules.net/)),  Matthew Persico,
Michele Beltrame ([Sigmafin](https://www.blendgroup.it/)), Rob Hall, Joel Roth,
Richard Leach, Jonathan Kean, Richard Loveland, Bojan Ramsa
