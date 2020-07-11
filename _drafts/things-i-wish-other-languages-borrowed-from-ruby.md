---
layout: post
title: "Things I Wish Other Languages Would Borrow From Ruby"
date: 2020-07-11
categories: ["ruby"]
---

This is a rant post. It frustrates me when new languages don't learn from the good parts of what has come before.

Ruby has been my favorite language for many years now. It certainly has its downsides, but it also has a lot of

## Package Management

Ruby has the "gem" command to install/uninstall rubygems, and "bundler" to manage project-specific dependencies. I have never had a single complaint about either, especially bundler.

* Installs to central folder by default, so no wasted HD space. Projects can re-use each others' gems.
* Always respects lock file unless you type `bundle update`.
* Semantics of `bundle install` and `bundle update` are obvious and do what you would expect. They ONLY ever modify the lock file.
* Does not allow a project to use multiple versions of the same dependency.
* Has easy `bundle install --deployment` to vendor gems for deployment.

### Tools that also get this right

* Maven

### Tools that don't

* npm (big surprise)
* yarn (inherited the sins of npm, now has nothing to offer since npm has a lock file)

I honestly do not know the semantics of most of the npm commands. They seem to alter package.json and package-lock.json willy nilly, or not at all (if you don't specify `-S`). Obviously "node_modules" is just awful, hundreds of MB of duplicated vendored dependencies for every project and pathological transitive dependencies. At leat [pnpm][pnpm] fixes this.

Also what should be a dependency vs a devDependency? This is unclear at best on most projects.

### Tools I'm not qualified to comment on

* Python pip

I think pip must be decent, but I'll be damned if I can find a single comprehensive overview of python package management anywhere so I remain totally confused by the common procedure of:

```bash
sudo easy_install pip
pip install ...
env/activate
```


[bundler]: https://bundler.io/
[pnpm]:
