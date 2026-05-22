---
layout: post
title: "Dead config and live redirects"
date: 2026-05-22
type: phase-update
entry_type: note
subtype: log
projects: [claudony]
tags: [quarkus, github, housekeeping]
---

Short session today — a cleanup branch and one unexpected discovery.

The actual work for #126 was already sitting in the working tree as unstaged changes. A previous session had removed the `quarkus.arc.selected-alternatives` block from both `application.properties` files but never committed. Twenty-two bean class names, no longer needed since qhorus#172 switched from `@Alternative` plus explicit selection to `@IfBuildProperty`/`@UnlessBuildProperty`. At build time only one stack is included — no ambiguity, no need for the config. We ran the tests, got 507 passing, committed, closed.

The interesting moment came during the push. `origin` still pointed to the old `mdproctor/claudony` URL — GitHub transferred the repo to `casehubio/claudony` at some point and we had not updated the remote. I expected either a failure or a rejected push. Instead it worked. The output flagged "This repository moved. Please use the new location" as an informational banner, then delivered the commit to `casehubio/claudony` anyway. Exit code zero.

I had not seen git push follow a redirect like that before. GitHub keeps the old URL alive and the HTTP transport just follows it. The commit was confirmed on `upstream/main` after a fetch — it landed in the right place. Worth noting: the banner is easy to miss if you are not reading push output carefully. You would think your code went one place when it actually went another.

Submitted that to the garden as an undocumented behaviour. Cleaned up a stale note in CLAUDE.md while I was at it — the line saying `selected-alternatives` was required. It was not, as of this commit.
