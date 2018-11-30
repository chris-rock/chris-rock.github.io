---
title: "Continuous Compliance"
author: Christoph Hartmann
date: 2017-10-19
tags:
  - inspec
  - security
  - compliance
conference: DevSecCon London 2017
link: https://www.devseccon.com/london-2017/session/devsec-continuous-compliance/
thumbnail: thumbnail.jpg
---

<script async class="speakerdeck-embed" data-id="e5da854e56af49c292ff7c1d81286858" data-ratio="1.77777777777778" src="//speakerdeck.com/assets/embed.js"></script>

Best-practices for server hardening and patching have been in place for decades. Nevertheless, it is still very cumbersome to enforce those rules continuously and many servers are still unsecured in 2016. DevOps tools like Chef, Puppet or Ansible help to enforce secure configuration, but they cannot fully assess a state of a machine e.g. you cannot easily verify if something is not installed. InSpec is here to help. It is an open source tool for infrastructure, security and compliance testing. InSpec’s DSL is a human and machine-readable assessment language that is extendable and customizable. Since testing can be fully automated with InSpec, companies are enabled to assess and enforce secure configuration across their IT fleet. Integration with CI/CD systems allows continuous testing in high-velocity organizations. This talk will give an introduction to InSpec and demonstrate how patch and security level can be assessed in CI/CD and production environments.