---
title: "🗄️ Backend"
layout: archive
permalink: categories/backend
author_profile: true
sidebar_main: true
---

{% assign posts = site.categories.backend %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}