---
layout: post
title: "[ETC] Jekyll : Hydejack theme 의 submenu 커스텀"
description: >
    "#jekyll #hydejack #submenu"
category: 
    - devlog
    - etc
hide_last_modified: true
related_posts:
    - 2025-04-28-devlog.md
---

![Image](https://github.com/user-attachments/assets/da7c1391-7fa7-46be-86d4-e051f8da114a)

---


# Hydejack theme의 최대 단점 : submenu의 부재

> 이 단점을 해결하기 위한 submenu custom 방법은 여러가지가 있다.
> 나도 hydejack 버전을 올리기 전 까진 tag가 category의 submenu가 되어 트리구조로 보이도록 하는 방법을 써왔다.
> 이 방법도 좋은 방법이였지만, 태그 추가 / 포스팅 시 조금 번거로운 점 때문에, hydejack 버전 업그레이드 하는 김에 submenu 커스텀도 변경했다.

# category 밑에 category가 들어가도록 custom

- 기존에는 devlog라는 카테고리 밑에 ai, python, web, etc 라는 태그가 포함되는 구조였지만, 지금 적용할 방법은 devlog, ai, python, web, etc 모두 카테고리가 된다.

# 커스텀 시작 

> Directory가 없으면 생성하면 된다.

## 1. `assets/js/sidebar-folder.js` 파일 생성 

- sidebar-folder.js

```javascript
function spread(count){
    document.getElementById('folder-checkbox-' + count).checked = 
    !document.getElementById('folder-checkbox-' + count).checked
    document.getElementById('spread-icon-' + count).innerHTML = 
    document.getElementById('spread-icon-' + count).innerHTML == 'arrow_right' ?
    'arrow_drop_down' : 'arrow_right'
}
```

## 2. `_include/body/nav.html` 파일 생성

- nav.html

```html
<span class="sr-only">{{ site.data.strings.navigation | default:"Navigation" }}{{ site.data.strings.colon | default:":" }}</span>
<ul>
  {% if site.menu %}
    {% for node in site.menu %}
      {% assign url = node.url | default: node.href %}
      {% assign count = count | plus: 1 %}
      <li>
        <div class="menu-wrapper">
          {% if node.submenu %}
            <button class="spread-btn" onclick="javascript:spread({{count}})">
              <span id="spread-icon-{{count}}" class="material-icons">arrow_right</span>
            </button>
          {% endif %}
          <a
            {% if forloop.first %}id="_drawer--opened"{% endif %}
            href="{% include_cached smart-url url=url %}"
            class="sidebar-nav-item {% if node.external  %}external{% endif %}"
            {% if node.rel %}rel="{{ node.rel }}"{% endif %}
            >
            {{ node.name | default:node.title }}
          </a>
        </div>
        {% if node.submenu %}
          <div class="menu-wrapper">
            <input type="checkbox" id="folder-checkbox-{{count}}">
            <ul>
            {% for subnode in node.submenu %}
              <li>
                <a
                  class="sidebar-nav-item {% if node.external  %}external{% endif %}"
                  href="{% include_cached smart-url url=subnode.url %}"
                  >
                  {{ subnode.title }}
                </a>
              </li>
            {% endfor %}
            </ul>
          </div>
        {% endif %}
      </li>
    {% endfor %}
  {% else %}
    {% assign pages = site.pages | where: "menu", true %}
    {% assign documents = site.documents | where: "menu", true %}
    {% assign nodes = pages | concat: documents | sort: "order" %}

    {% for node in nodes %}
      {% unless node.redirect_to %}
        <li>
          <a
            {% if forloop.first %}id="_navigation"{% endif %}
            href="{{ node.url | relative_url }}"
            class="sidebar-nav-item"
            {% if node.rel %}rel="{{ node.rel }}"{% endif %}
            >
            {{ node.title }}
          </a>
        </li>
      {% else %}
        <li>
          <a href="{{ node.redirect_to }}" class="sidebar-nav-item external">{{ node.title }}</a>
        </li>
      {% endunless %}
    {% endfor %}
  {% endif %}
</ul>
```

## 3. `_sass/my-style.scss` 파일에 내용 추가
- my-style.scss
```scss
/* 추가 */
.sidebar-sticky {
  height: 100%;
  padding-top: 5%;
  position: absolute;
}
.sidebar-about {
  padding-bottom:10%;
}
.spread-btn{
  left: 7%;
  position: absolute;
  padding: 0;
  padding-top: 5px;
  border: none;
  background: none;
  color: white;
  cursor: pointer;
}

.spread-btn:hover{
  color: grey;
}

.menu-wrapper{
  display: flex;
  text-align: left;
  margin-left: 10%;
  margin-bottom: 0%;
  
  input[type=checkbox]{
    display: none;
  }
  
  input[type=checkbox] ~ ul{
    display: none;
    list-style: none;
  }
  
  input[type=checkbox]:checked ~ ul{
    display: block;
  }
}
```

## 4. `_includes/my-head.html` 파일에 내용 추가

- my-head.html

```html
<!-- 추가 -->
<link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet">
<script src="/assets/js/sidebar-folder.js"></script>
```

## 5. `_config.yml` 파일 수정
```yml
menu:
  - title:             Devlog
    url:               /devlog/
    #---- 추가 --------------------
    submenu: 
      - title:         AI
        url:           /ai/
      - title:         Python
        url:           /python/
      - title:         Web
        url:           /web/
      - title:         ETC
        url:           /etc/
    #-----------------------------
  - title:             Tennis
    url:               /tennis/
```

## 6. `_featured_categories` 에 category.md 추가

- ex) ai.md

```
---
layout: list
type: category
title: AI
slug: ai
sidebar: true
order: 1
description: >
   About AI.. 
---
```

## 7. 포스팅 시, 메타데이터에 서브메뉴까지 명시

- ex) 2025-04-29-posting.md

```
---
layout: post
title: "타이틀"
category: 
    - devlog
    # --- 추가 -----
    - ai
    # -------------
hide_last_modified: true
---
포스팅 내용
```

## 8. 수정 내용 배포 & 결과 확인

![Image](https://github.com/user-attachments/assets/4e9c811b-57b1-471f-b6e7-370c39c35cce)