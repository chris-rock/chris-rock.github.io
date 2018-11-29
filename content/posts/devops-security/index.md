---
title: DevOps and Security
author: chris
date: 2015-06-19
template: article.jade
---

To ensure the security of your IT services, different disciplines need to come together. Development, operations and security departments need to work hand in hand in order to ship a secure product. Every department has its core competencies and it is a challenge to create a common view on the security implementation for a product.

## The challenge

The separation of knowledge between departments has its advantages but comes with some disadvantages. Quite often, the separation leads to an environment where different departments have different goals:

 * dev teams are under pressure to ship the product
 * ops teams need to ensure everything runs smoothly
 * security teams have to ensure that the new product is secure

Since extensive technical know-how is required to ship a product (and even more is required to ship a successful product), every party needs to understand and breath the whole product delivery process and every aspect of the implementation need to be well known. On top of that, each department has to deal with different artifacts:

 * source code (developers)
 * packages & runtime (ops)
 * regularities, pen-tests (security)

In a static world, this would not be an issue, but since the velocity of technology increases, the shipment of each artifacts need to synchronized in order to ship a product in time. The question is, how departments are able deliver a product together by focusing on their core competency?

## Find the same language

About two years ago, we faced the challenge mentioned above and it got us thinking? From our perspective, the current delivery process was broken. Would there be a better approach? Could we use new technologies to speed up the development and time-to-market, as well as help departments to work more effectively together?

We identified a few patterns:

- Security teams need a way to describe their rules without the need of extensive programming skills
- Developers need an automated way to deal with security configuration
- Operations need a deterministic and manageable software

In a small team of four ([Dominik](http://arlimus.github.io/), Patrick, [Edmund](http://ehaselwanter.com/) and I) worked in an interdisciplinary team with know-how in each area for product delivery and security. We started prototyping, developing, deploying. Eventually, we created a project called [DevSec Hardening Framework](http://dev-sec.io/) with Deutsche Telekom. What is special about the Hardening Framework?

 * Security configuration is documented as code which is great for developers, because they are able to focus on product implementation
 * Instead of abstract regularities, we used Serverspec to define tests. This allowed use the tests as a verification mechanism
 * All security requirements are automated via a configuration management tool
 * The implementation is completely deterministic which is essential for operations
 * Security is implemented by design

In practice, we developed the security test rules first (like):

 * https://github.com/dev-sec/tests-os-hardening

From now on, security teams started developing tests instead of static documents, because those can be re-used across the whole company. E.g. those can be integrated in CI environments or used on local test machines.

Next, development and operations needed an implementation that automatically remedies failed tests. We implemented the security configuration in Chef, Puppet and Ansible:

 * https://github.com/dev-sec/chef-os-hardening
 * https://github.com/dev-sec/puppet-os-hardening
 * https://github.com/dev-sec/ansible-os-hardening

All implementations of the os-hardening module use the same tests. By writing all security requirements as tests rules, we are able to automate tests and increase the velocity of the product delivery. Think of a manufacturing line that has security built-in.

## Take-Away

It is essential to address the need of each department to ship a successful product. DevOps tools help companies to align development and operations teams. By defining the common set as regulatory rules, all departments find a way to collaborate more efficiently together and are able integrate security tests into the normal product development workflow.

If you have any questions contact me via [Twitter @chri_hartmann](https://twitter.com/chri_hartmann) or [Github](https://github.com/chris-rock)

[Vulcano Security](http://vulcanosec.com/) provides support for security automation, compliance scanning and patch management.
