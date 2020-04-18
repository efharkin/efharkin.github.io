---
title: Curriculum Vitae
layout: page
---

## Posters and Talks

<ul class="cv-list">
    {% for pres in site.data.cv.presentations %}
    <li>
        <h3>{{pres.title}}</h3>
        <div style="display: flex;">
            <p style="flex: 7;">{{pres.location}}</p>
            <div class="time-container" style="flex: 3;" align="right">
                <time datetime="{{pres.date | date: '%Y-%m'}}">{{ pres.date }}</time>
            </div>
        </div>
        <p>{{ pres.authors | markdownify }}</p>
        <p>{{ pres.description | markdownify }}</p>
    </li>
    {% endfor %}
</ul>

## Outreach

## Awards

<ul class="cv-list">
    {% for award in site.data.cv.awards %}
    <li>
        <h3>{{award.title}}</h3>
        <div style="display: flex;">
            <p style="flex: 6;">{{ award.source }}</p>
            <div class="time-container" style="flex: 4;" align="right">
                <time datetime="{{award.start_date | date: '%Y-%m'}}">{{ award.start_date }}</time>
                &ndash;
                <time datetime="{{award.end_date | date: '%Y-%m'}}">{{ award.end_date }}</time>
            </div>
        </div>
        <p>{{ award.description | markdownify }}</p>
    </li>
    {% endfor %}
</ul>

## Education

<ul class="cv-list">
    {% for degree in site.data.cv.education %}
    <li>
        <h3>{{degree.title}}</h3>
        <div style="display: flex;">
            <p style="flex: 6;">{{ degree.source }}</p>
            <div class="time-container" style="flex: 4;" align="right">
                <time datetime="{{degree.start_date | date: '%Y-%m'}}">{{ degree.start_date }}</time>
                &ndash;
                <time datetime="{{degree.end_date | date: '%Y-%m'}}">{{ degree.end_date }}</time>
            </div>
        </div>
        <p>{{ degree.description | markdownify }}</p>
    </li>
    {% endfor %}
</ul>
