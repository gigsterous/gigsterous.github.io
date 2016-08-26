# Gigsterous Blog

[![Build Status](https://travis-ci.org/gigsterous/gigsterous.github.io.svg?branch=master)](https://travis-ci.org/gigsterous/gigsterous.github.io)

This is a repository for our engineering blog [hosted on GitHub](http://gigsterous.github.io).

## Posting + Continuous Integration

The publishing side is completely automated. When a push is made, not only is the page fully built, but every post is also checked by [html-proofer](https://github.com/gjtorikian/html-proofer).

To publish a new post:

1. Create a new branch and prefix it with **pages-**. This is for Travis, who builds master and every branche prefixed with **pages-**.
2. Write your post/article. [Here](https://jekyllrb.com/docs/posts/) is how to write posts and [here](https://jekyllrb.com/docs/drafts/) is how to use drafts.
3. Submit a PR.
