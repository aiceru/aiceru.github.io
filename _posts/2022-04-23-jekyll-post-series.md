---
layout: post
title: '[Jekyll] Collapsible Post Series Panel'
categories: [Blog]
tags: [jekyll, blog, collapsible, panel, series]
feature_image: 'https://images.unsplash.com/photo-1444012183556-ad267375edb4?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1770&q=80'
feature_license: 'Photo by <a href="https://unsplash.com/@davidstraight?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">David Straight</a> on <a href="https://unsplash.com/s/photos/foldable?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
use_math: false
published: true
last_modified_at: '2022-04-23 23:50:01'
---

<!-- more -->
# Jekyll blog 에서 'Post Series' 만들고 collapsible panel 로 post 목록 보여주기
포스팅을 하다 보면 간혹 너무 긴 글을 여러 편의 시리즈로 나누거나, 하나의 주제로 여러 편의 글을 시리즈로 작성하고 싶을 때가 있다. jekyll 에서 여러 편의 포스팅을 하나의 시리즈로 묶고, 시리즈의 모든 포스팅 첫 머리에 해당 시리즈의 포스팅 목록을 보여 주는 패널을 만들어 보자.

## Create template for 'series'
먼저 /include 디렉토리에, 시리즈 목록을 표시할 html template 파일을 작성한다.
```
{% raw %}
{% if page.series %}
  {% assign count = '0' %}
  {% assign idx = '0' %}
  {% for post in site.posts reversed %}
    {% if post.series == page.series %}
      {% capture count %}{{ count | plus: '1' }}{% endcapture %}
      {% if post.url == page.url %}
        {% capture idx %}{{count}}{% endcapture %}
      {% endif %}
    {% endif %}
  {% endfor %}
{% endif %}
<details>
  <summary>
    Post Series <sub>(click to expand)</sub>
    <h3>{{ page.series }}</h3>
  </summary>
  <div class="panel seriesNote">
    {% assign count = '0' %}
    {% for post in site.posts reversed %}
    {% if post.series == page.series %}
      {% capture count %}{{ count | plus: '1' }}{% endcapture %}
      👉 Part {{ count }} -
      {% if page.url == post.url %}
        {{ page.title }}
      {% else %}
        <a href="{{post.url}}">{{post.title}}</a>
      {% endif %}<br/>
    {% endif %}
    {% endfor %}
  </div>
</details>
<hr/>
{% endraw %}
```
{% include code-caption.html text="/_includes/series.html" %}

`<summary>` 태그에 'Post Series <sub>(click to expand)</sub>' 구문과 함께 시리즈 제목을 표시하고,
확장되는 패널(`<div class="panel seriesNote">` 태그) 에 같은 시리즈에 속한 포스트들의 목록을 표시하도록 했다. 조금만 코드를 뜯어본다면 원하는 대로 수정해서 사용할 수 있을 것이다.

## Add 'series' tag to Post
포스트의 front matter 에 series 태그를 추가하고, series tag 가 있는 포스트의 본문에는 `series.html` 을 include 해준다.
```markdown
---
...
series: "My series 1"
...
---

{% raw %}
{% include series.html %}
{% endraw %}
```

## Add css for Panel class
블로그의 css 또는 scss 파일에 panel class 를 위한 스타일을 추가해준다. 내 경우에는 _sass 디렉토리 아래 _panel.scss 파일을 새로 만들고 `@import` 구문을 이용하여 추가했다.

```css
.panel {
  padding: 15px;
  margin-bottom: 20px;
  background-color: #111111;
  border: 1px solid #3f3f3f;
  border-radius: 4px;
  -webkit-box-shadow: 0 1px 1px rgba(0,0,0,0.05);
  box-shadow: 0 1px 1px rgba(0,0,0,0.05)
}

.panel-heading {
  padding: 10px 15px;
  margin: -15px -15px 15px;
  background-color: #f5f5f5;
  border-bottom: 1px solid #ddd;
  border-top-right-radius: 3px;
  border-top-left-radius: 3px
}

.panel-title {
  margin-top: 0;
  margin-bottom: 0;
  font-size: 17.5px;
  font-weight: 500
}

.panel-footer {
  padding: 10px 15px;
  margin: 15px -15px -15px;
  background-color: #f5f5f5;
  border-top: 1px solid #ddd;
  border-bottom-right-radius: 3px;
  border-bottom-left-radius: 3px
}

.panel-primary {
  border-color: #428bca
}

.panel-primary .panel-heading {
  color: #fff;
  background-color: #428bca;
  border-color: #428bca
}

.panel-success {
  border-color: #d6e9c6
}

.panel-success .panel-heading {
  color: #468847;
  background-color: #dff0d8;
  border-color: #d6e9c6
}

.panel-warning {
  border-color: #fbeed5
}

.panel-warning .panel-heading {
  color: #c09853;
  background-color: #fcf8e3;
  border-color: #fbeed5
}

.panel-danger {
  border-color: #eed3d7
}

.panel-danger .panel-heading {
  color: #b94a48;
  background-color: #f2dede;
  border-color: #eed3d7
}

.panel-info {
  border-color: #bce8f1
}

.panel-info .panel-heading {
  color: #3a87ad;
  background-color: #d9edf7;
  border-color: #bce8f1
}
```
{% include code-caption.html text="/_sass/_panel.scss" %}

각자 블로그의 테마에 맞게 background, border color 등을 수정해 준다. 새로 추가한 series.html 파일에서 실제로는 `panel` class 만 이용하고 있으므로 전부 수정할 필요는 없고 맨 처음의 `.panel` 스타일링만 수정해 주면 된다. 나머지 스타일들은 혹시 모를 나중을 위해 남겨 두었다.

{% include figure.html image="/assets/img/posts/20220423/collapsed.png" caption="collapsed list" %}
{% include figure.html image="/assets/img/posts/20220423/expanded.png" caption="expanded list" %}

# References
- [Jekyll Part 13: Creating an Article Series](https://digitaldrummerj.me/blogging-on-github-part-13-creating-an-article-series/)