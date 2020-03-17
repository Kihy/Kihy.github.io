---
layout: page
title: Blog
description: Random things written by me
nav-menu: true
image: assets/images/pub.jpg
author: null
show_tile: true
---

<div id="main" class="alt">

<!-- One -->


	<div class="inner">
		<header class="major">
			<h1>Latest Posts</h1>
		</header>

    <ul class="alt">
      {% for post in site.posts %}
        <li>
          <div class="row">
          <div class="4u 12u$(small)">
          <b><a href="{{ post.url }}">{{ post.title }}</a></b>
          </div>
          <div class="8u 12u$(small)">
          {{post.description}}
          </div>
          </div>
          <hr>
        </li>
      {% endfor %}
    </ul>


</div>
