---
title: Pwning the Entire Nix Ecosystem
layout: post
description: Pwning Nixpkgs with a single pull request
tags: nixpkgs nix github-actions vulnerability
---

last year at nixcon, me and [my friend lexi](https://mastodon.catgirl.cloud/@49016) gave [a lightning talk](https://youtube.com/live/_7wqXN-7ebw?t=38450s) about how we found a vulnerability in nixpkgs that would have allowed us to pwn pretty much the entire nix ecosystem and inject malicious code into nixpkgs. it only took us about a day from starting our search to reporting it and getting it fixed. since i unfortunately was too sick to attend this years nixcon, i thought it might be a good time to write up what we found and how we did it.

## github actions: the easy target

github actions is a ci/cd system by github that can do pretty much anything in a repo. it's an easy target for attackers because if you have access to a workflow, you can just commit code without authorization and then you have a supply chain attack. plus, it's all written in yaml [🇳🇴](https://ruudvanasseldonk.com/2023/01/11/the-yaml-document-from-hell), which was NEVER meant to be executed !!



```yaml
name: learn-github-actions
on: [push]
jobs:
  check-bats-version:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm install -g bats
      - run: bats -v
```

this is a simple example of a github action. nothing fancy, just running some commands when code is pushed.

## the dangerous pull_request_target

actions run when a trigger activates them. there are a bunch of different triggers like pushes, commits, or pull requests. but there's a special one called `pull_request_target` that has a few critical differences from regular `pull_request`.

crucially, unlike `pull_request`, `pull_request_target` has read/write and secret access by default, even on pull requests from forks. this isn't vulnerable by itself, but things go south when you start trusting user input from those PRs.

github even warns about this in their docs:

> Warning: For workflows that are triggered by the `pull_request_target` event, the `GITHUB_TOKEN` is granted read/write repository permission unless the `permissions` key is specified and the workflow can access secrets, even when it is triggered from a fork.

so we started looking for workflows in nixpkgs that use `pull_request_target` and found 14 files. some of them were secure, like this labeler example:

```yaml
name: "Label PR"
on:
  pull_request_target:
jobs:
  labels:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/labeler@8558fd74291d67161a8a
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
```

this is safe because it just passes the token to a trusted action. but then we found some more interesting ones...

## the editorconfig vulnerability

the first vulnerable workflow we found was for checking editorconfig rules. here's a simplified version of what it was doing:

```yaml
steps:
  - name: Get list of changed files from PR
    run: gh api [...] | jq [ ... ] > "$HOME/changed_files"
  - uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf5d482be7e7871
    with:
      ref: refs/pull/${{ github.event.pull_request.number }}/merge
  - name: Checking EditorConfig
    run: cat "$HOME/changed_files" | xargs -r editorconfig-checker
```
the workflow would:
1. get a list of files changed in the PR
2. checkout the PR code
3. run editorconfig-checker on those files

the problem? it was using `xargs` to pass the filenames to editorconfig-checker. if you've read the [man page for xargs](https://man7.org/linux/man-pages/man1/xargs.1.html#:~:text=It%20is%20not%20possible%20for%20xargs%20to%20be%20used%20securely), you'll see this warning:

> It is not possible for xargs to be used securely

basically, we could create a file with a name that's actually a command line argument. for example, if we added a file called `--help` to our PR, when the workflow ran `cat "$HOME/changed_files" | xargs -r editorconfig-checker`, the filename would be passed as an argument to editorconfig-checker, causing it to print its help message instead of checking files.

![editor config run with help output](/assets/images/posts/editor-config-run.png)

this is a classic command injection vulnerability. we didn't take it further to try to execute arbitrary code since editorconfig-checker is written in go and we'd need to audit it more deeply, but it's most likely possible.

## the code owners vulnerability: local file inclusion

the second vulnerable workflow we found was even more serious. it was checking the CODEOWNERS file in PRs:

```yaml
steps:
  - uses: actions/checkout@eef61447b9ff4aafe5dcd4e0bbf
    with:
      ref: refs/pull/${{ github.event.number }}/merge
      path: pr
  - run: nix-build base/ci -A codeownersValidator
  - run: result/bin/codeowners-validator
    env:
      OWNERS_FILE: pr/ci/OWNERS
```

the workflow would:
1. checkout the PR code
2. build the codeowners validator
3. run the validator on the OWNERS file from the PR

the validator would echo the contents of the OWNERS file if there was an error. this meant we could put whatever we wanted in that file and it would get printed in the logs.

but it gets worse. since the workflow was checking out our PR code, we could replace the OWNERS file with a symbolic link to ANY file on the runner. like, say, the github actions credentials file:

```bash
$ rm OWNERS
$ ln -s /home/runner/runners/2.320.0/.credentials OWNERS
```

when the validator ran, it would try to read our symlinked file and helpfully print out an error message containing the first line:


![screenshot of error message including censored token](/assets/images/posts/codeowners-error.png)

and just like that, we had a github actions token with read/write access to nixpkgs. this would let us push directly to nixpkgs, bypassing all the normal review processes.

## the fix

after we found these vulnerabilities, we reported them to the nixpkgs maintainers, in this case infinisil, who immediately took action:

1. they disabled the vulnerable workflows in the repos action settings
2. they fixed the vulnerabilities by properly separating untrusted data from privileged operations
3. they renamed the fixed workflows after the security fixes, this is because of another pitfall with `pull_request_target` allowing you to target any branch the action is on, even if it's 5 or 10 years old as long as it hasn't been disabled.

the key lessons from this:
- avoid mixing untrusted data and secrets, or be very careful with them
- only allow the permissions you really need
- read the docs about permissions very carefully

if you think your org has vulnerable github actions, you can use the panic button too:
- go to your org at https://github.com/[org]
- go to the "Settings" tab
- go to "Actions" → "General" section
- under "Policies", switch "All repositories" to "Disable"

## conclusion

it only took us about a day to find, report, and help fix a vulnerability that could have compromised the entire nix ecosystem. this shows how important it is to be careful with github actions, especially when dealing with `pull_request_target`.

big thanks to intrigus and everyone at KITCTF (intrigus gave [a talk about exactly these issues](https://kitctf.de/learning/insecure-github-actions) that taught us how this works), and thanks to infinisil for fixing this on the same day we reported it.

if you want to learn more, check out these resources:
- [https://kitctf.de/talks/2023-10-26-insecure-github-actions/insecure-github-actions.pdf](https://kitctf.de/talks/2023-10-26-insecure-github-actions/insecure-github-actions.pdf])
- [https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/](https://securitylab.github.com/resources/github-actions-preventing-pwn-requests/)
- [https://github.com/NixOS/nixpkgs/pull/351446](https://github.com/NixOS/nixpkgs/pull/351446)

also, if you're curious, you can watch [our original lightning talk](https://youtube.com/live/_7wqXN-7ebw?t=38450s) from nixcon

anyway that's all. stay safe with your github actions. meow :3
