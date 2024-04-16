---
layout: page
title: Resume
permalink: /resume/
---


{% include career-profile.html %}

{% unless site.data.data.sidebar.education %}
  {% include education.html %}
{% endunless %}

{% include experiences.html %}

{% include certifications.html %}
