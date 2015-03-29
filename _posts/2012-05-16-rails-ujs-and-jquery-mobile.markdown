---
layout: post
title: "Rails UJS and jQuery Mobile"
date: 2012-08-04 14:02
comments: true
categories: rails, jquery, javascript
---

I ran into a gotcha when working with jQuery Mobile and Rails UJS helpers. As you probably know, adding `remote: true` to your `form_for` arguments triggers Rails to add some UJS helpers that submit that form via AJAX rather than a normal post.

However, jQuery Mobile looks for any form element and does the same. As a result, I was getting double posts each time I clicked submit on what should have been a very simple form.

Adding a data attribute of `data-ajax='false'` prevents jQuery Mobile from trying to muck with the form at all and lets Rails do its thing.

