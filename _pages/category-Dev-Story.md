---
title: "📒 개발 일지"
layout: archive
permalink: categories/dev-story
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.dev-story %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}