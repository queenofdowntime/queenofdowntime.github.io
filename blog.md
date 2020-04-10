---
layout: page
title: blog
permalink: /blog
menu: nav
---

<div class="home">
  <ul class="post-list">
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
    <li>
      <span class="post-meta">{{ "2020-04-10" | date: date_format }}</span>
        <a class="post-link" href="/blog/remote-pair-programming">
          {{ "How to pair-program remotely and not lose the will to live (or: there has never been a better time to learn Vim)" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2019-10-14" | date: date_format }}</span>
        <a class="post-link" href="/blog/inotify-release-deadlock-envoy">
          {{ "Envoy Proxy Deadlocked My Cloud" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2019-09-11" | date: date_format }}</span>
        <a class="post-link" href="/blog/hiking-in-norway">
          {{ "7 Days Hiking Alone from Voss to Dale, Norway" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2019-03-29" | date: date_format }}</span>
        <a class="post-link" href="https://www.cloudfoundry.org/blog/an-overlayfs-journey-with-the-garden-team/">
          {{ "Why is OverlayFS slow now?" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2018-09-08" | date: date_format }}</span>
        <a class="post-link" href="/blog/the-route-to-rootless-containers">
          {{ "The Route to Rootless Containers" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2018-05-09" | date: date_format }}</span>
        <a class="post-link" href="/blog/a-day-in-the-life-of-a-cf-engineer">
          {{ "A day in the life of a Cloud Foundry engineer" | escape }}
        </a>
    </li>
    <li>
      <span class="post-meta">{{ "2017-09-08" | date: date_format }}</span>
        <a class="post-link" href="/blog/container-rootfilesystems-in-prod">
          {{ "Container Root Filesystems in Production" | escape }}
        </a>
    </li>
  </ul>
</div>
