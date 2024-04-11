---
title: Using OBS as a CI for Perl Modules
date: 2024-04-11T23:11:55+02:00
tags: [software engineering, open build service, perl]
draft: false
---

## Introduction

In my previous [Building a Perl Module in
OBS](../2024-04-building-a-perl-module-in-obs/) post I showed how you can
build rpm packages in OBS from your perl module(s). Every commit on the main
branch will trigger a new build.

Now I will show how to automatically build the package from a GitHub Pull
Request. It's recommended to read the previous post first.

For that I need to add an OBS Workflow. I need a GitHub Token, an OBS Token
and a workflow YAML file.

## Create Github Access Token

Go to your [GitHub Settings](https://github.com/settings/profile) - Developer
Settings - [Personal Access Tokens](https://github.com/settings/tokens) and click on
"Generate new token (classic)".

Add a meaningful description like "Token for OBS workflow for
home:user:perlmodules/perl-YAML-PP".

Enable the scope "repo".

Click on "Generate token".

![OBS - Home Project](/create-obs-workflow/010-github-new-token.png)

You should see a message that the token was generated and a secret (marked
blue in the screenshot). Note that down.

![OBS - Home Project](/create-obs-workflow/020-github-created-token.png)

## Generate OBS Workflow token

Now go to [My tokens](https://build.opensuse.org/my/tokens) in OBS and create
a new token.

![OBS - Home Project](/create-obs-workflow/030-obs-tokens.png)

The token type is "workflow". Add a meaningful description, e.g. "Pull requests
for home:user:perlmodules/perl-YAML-PP".

Paste the GitHub Access Token.

Click on "Create".

![OBS - Home Project](/create-obs-workflow/040-obs-create-workflow-token.png)

You should now see a message that the token was successfully created.

Note down the secret (marked in red) and the trigger url (marked in green).

![OBS - Home Project](/create-obs-workflow/050-obs-created-workflow-token.png)

## Add GitHub Webhook

In your GitHub project, go to Seetings - Webhooks and add a new webhook.

Paste the trigger url and the secret from the last step.

![OBS - Home Project](/create-obs-workflow/060-github-create-webhook.png)

Check "Let me select individual events." and select "Push" and "Pull requests".

Click on "Add webhook".

![OBS - Home Project](/create-obs-workflow/061-github-create-webhook.png)

## Create Workflow configuration file

In your GitHub repository, add a new file at `.obs/workflows.yml`:
```
pr:
  steps:
  - branch_package:
      source_project: home:tinita:perlmodules
      source_package: perl-YAML-PP
      target_project: home:tinita:perlmodules:CI
  filters:
    event: pull_request
```

Commit and push to the main branch.

## Create a Pull request

Now everything is ready for getting GitHub OBS workflows checks.

Create a pull request. Maybe one that makes tests fail ;-)

![OBS - Home Project](/create-obs-workflow/070-github-create-pr.png)

Now go to the subprojects of your `home:user:perlmodules` project.

If everything went well, you should see a new subproject there, called
`home:tinita:perlmodules:CI:perlpunk:gpw2024-demo:PR-3` in my case.

![OBS - Home Project](/create-obs-workflow/080-obs-subprojects.png)

It should look almost like the original project.

![OBS - Home Project](/create-obs-workflow/090-obs-subproject-package.png)

And it should already start to build!

![OBS - Home Project](/create-obs-workflow/091-obs-subproject-package.png)

If you look at the Pull request now, you should see checks being added, in
my case five.

Note that it is a current limitation of OBS that those checks only appear
when they finish. A pending status is not supported.

![OBS - Home Project](/create-obs-workflow/100-github-pr.png)

## Troubleshooting

Things can go wrong in various places, and it's possible that it's not
your fault.

* Check the GitHub Webhook settings. You can list the triggers and see the
  payload
* Check the OBS Token settings. You can also look at the triggers there
* In some cases, OBS might simply not have been reachable. In that case
  it's necessary to delete the created subproject and retrigger the necessary
  events.


Have fun with the OBS Workflow!
