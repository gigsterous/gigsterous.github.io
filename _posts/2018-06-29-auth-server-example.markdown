---
layout: post
title: Spring Boot example auth server
date: 2018-06-29 15:00:00 +0200
author: Martin
description: Open source auth server example written in Spring Boot.
categories: engineering
tags:
  - Spring-Boot
  - Java
  - Security
---

In the past, we discussed various techniques for writing a Spring Boot application, including security, REST api etc. We proudly announce the release of the most complex Spring Boot authorization example so far. Feel free to check out our [GitHub repository](https://github.com/gigsterous/auth-server).

Here is a list of features supported by the auth server:
- Username and password Authentication
- OAuth2 Access + Refresh Token Provision
- Registration with e-mail confirmation
- Basic account management including password change, forgotten password, e-mail change and account deletion
- Multilingual support
- Logout including token invalidation
- Easy SonarQube, Jacoco and Checkstyle integration for code-quality monitoring
- Basic unit and integration test coverage with example tests

This example is written in Spring Boot 2 and we will try to maintain it to keep up with the latest changes. In case you wish to contribute or ask a question, we will be happy to discuss it with you on GitHub!
