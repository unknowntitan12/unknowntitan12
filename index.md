---
layout: default
title: ""
---

<div class="home-hero">
  <div class="hero-eyebrow">ICT1012 · Operating Systems</div>
  <h1>unknown<span>titan</span><em>12</em></h1>
  <p>xv6 lab solutions, study notes, and prediction questions — every week, every lab.</p>
</div>

<section class="home-section">
  <div class="section-label">// Lab Solutions</div>
  <div class="home-grid">
    {% assign lab1 = site.os_notes | where: "lab", "lab1" | sort: "order" %}
    {% for note in lab1 %}
    <a href="{{ note.url | relative_url }}" class="home-card lab1">
      <div class="card-eyebrow">Lab 1 &middot; Week {{ note.week }}</div>
      <div class="card-title">{{ note.short_title | default: note.title }}</div>
      <div class="card-date">{{ note.date | date: "%d %b %Y" }}</div>
    </a>
    {% endfor %}

    {% assign lab2 = site.os_notes | where: "lab", "lab2" | sort: "order" %}
    {% for note in lab2 %}
    <a href="{{ note.url | relative_url }}" class="home-card lab2">
      <div class="card-eyebrow">Lab 2 &middot; Week {{ note.week }}</div>
      <div class="card-title">{{ note.short_title | default: note.title }}</div>
      <div class="card-date">{{ note.date | date: "%d %b %Y" }}</div>
    </a>
    {% endfor %}

    {% assign lab3 = site.os_notes | where: "lab", "lab3" | sort: "order" %}
    {% for note in lab3 %}
    <a href="{{ note.url | relative_url }}" class="home-card lab3">
      <div class="card-eyebrow">Lab 3 &middot; Week {{ note.week }}</div>
      <div class="card-title">{{ note.short_title | default: note.title }}</div>
      <div class="card-date">{{ note.date | date: "%d %b %Y" }}</div>
    </a>
    {% endfor %}
  </div>
</section>

<section class="home-section">
  <div class="section-label">// Predictions</div>
  <div class="home-grid">
    {% assign preds = site.predictions | sort: "title" %}
    {% for pred in preds %}
    <a href="{{ pred.url | relative_url }}" class="home-card pred">
      <div class="card-eyebrow">★ Prediction</div>
      <div class="card-title">{{ pred.title }}</div>
      <div class="card-date">{{ pred.date | date: "%d %b %Y" }}</div>
    </a>
    {% endfor %}
    {% if preds.size == 0 %}
    <div class="home-card" style="opacity:0.35;cursor:default;">
      <div class="card-eyebrow">Predictions</div>
      <div class="card-title">None yet</div>
    </div>
    {% endif %}
  </div>
</section>
