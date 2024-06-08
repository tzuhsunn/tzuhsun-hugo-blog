---
title: 'Hugo Set Up with PaperMod'
date: 2024-06-08T22:22:14+08:00
draft: false
description: "Bluid a blog with Hugo, PaperMod & Firebase"
tags: ["Hugo", "Firebase"]
categories: ["Hugo Note"]
summary: "Bluid a blog with Hugo, PaperMod & Firebase"
---
## Installation
[PaperMod Installation Guide](https://adityatelange.github.io/hugo-PaperMod/posts/papermod/papermod-installation/)  
安裝 Hugo
```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew install hugo
hugo new site <site-name> --format yaml
cd <site-name>
```
選擇主題並下載，我選擇PaperMod  
[Related YT tutorial](https://www.youtube.com/playlist?list=PLeiDFxcsdhUrzkK5Jg9IZyiTsIMvXxKZP)
```bash
git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
# If you want to update the theme
cd themes/PaperMod
git pull

# Start Hugo server
hugo server
```
## Setup `hugo.yaml`
參考 PaperMod Installation Guide, 直接修改 `hugo.yaml`，我還有很多features沒有研究透徹...  
TODO: Google analytics features  
[Feature 設定](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#search-page)
```yaml
baseURL: "" # 部署用的域名
title: TzuHsun's Blog # 網站 title
languageCode: en-us
paginate: 10 # 分頁數
theme: PaperMod # 設定主題
frontmatter:
  lastmod: [":fileModTime", "lastmod"] # 新增lastmod變數用來顯示最後修改時間

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

#googleAnalytics: UA-123-45

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production # to enable google analytics, opengraph, twitter-cards and schema.
  title: TzuHsun's Blog
  description: "TzuHsun's Blog description"
  keywords: [Blog, Portfolio, PaperMod]
  author: Me
  # author: ["Me", "You"] # multiple authors
  #images: ["<link or path of image for opengraph, twitter-cards>"]
  DateFormat: "January 2, 2006" # 設定時間顯示格式
  defaultTheme: auto # dark, light
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true # 顯示複製code的按鈕
  ShowWordCount: true
  ShowRssButtonInSectionTermList: true
  UseHugoToc: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: false
  hidemeta: false
  hideSummary: false
  showtoc: false
  tocopen: false

  assets:
    # disableHLJS: true # to disable highlight.js
    # disableFingerprinting: true
    favicon: "<link / abs url>"
    favicon16x16: "<link / abs url>"
    favicon32x32: "<link / abs url>"
    apple_touch_icon: "<link / abs url>"
    safari_pinned_tab: "<link / abs url>"

  label:
    text: "TzuHsun's  Blog"


  # profile-mode
  profileMode:
    enabled: false # needs to be explicitly set
    title: ExampleSite
    subtitle: "This is subtitle"
    imageUrl: "<img location>"
    imageWidth: 120
    imageHeight: 120
    imageTitle: my image
    buttons:
      - name: Posts
        url: posts
      - name: Tags
        url: tags

  # home-info mode
  homeInfoParams: # 設定主頁資訊
    Title: "Hi there \U0001F638\U0001F44B"
    Content: A Blog about My Daily Life and Tech Notes

  socialIcons: # 設定社群相關連結
    - name: github
      url: "https://github.com/tzuhsunn"
    - name: linkedin
      url: "https://www.linkedin.com/in/tzuhsuntseng/"
    - name: email
      url: "mailto:hsungrass@gmail.com"
    - name: medium
      url: "https://medium.com/@hsunslife"

  analytics:
    google:
      SiteVerificationTag: "XYZabc"
    bing:
      SiteVerificationTag: "XYZabc"
    yandex:
      SiteVerificationTag: "XYZabc"

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page
    linkFullImage: true # link cover image to full image

  editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link

  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content"]
menu: # 設定右上角頁面選項
  main:
    - identifier: categories
      name: Categories
      url: /categories/
      weight: 10
    - identifier: tags
      name: Tags
      url: /tags/
      weight: 20
    - identifier: archives # 要加上content/archive.md
      name: Archives
      url: /archives/
      weight: 30
    - identifier: search # 要加上content/search.md
      name: Search
      url: /search/
      weight: 40
# Read: https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#using-hugos-syntax-highlighter-chroma
pygmentsUseClasses: true
markup:
  highlight:
    noClasses: false
    anchorLineNos: true
    codeFences: true
    guessSyntax: true
    lineNos: true
    # style: monokai

outputs: # 如果要search的話要有以下設定
  home:
    - HTML
    - RSS
    - JSON # necessary for search

```

## Add search, archives
1. Add `content/search`
```md
---
title: "Search" # in any language you want
layout: "search" # necessary for search
description: "Search something interesting!"
summary: "search"
placeholder: "Search <--"
---
```
2. Add `content/archives`
```md
---
title: 'Archives'
layout: 'archives'
url: "/archives/"
summary: archives
---
```
## Add new Post
Type command in terminal
```bash
hugo new posts/first_post.md
# Edit first_post.md
```

## Show last modified time in post
想要達成的效果：
{{< figure  src="/hugo-posts/hugo-setup/post-with-last-update-time.png" title="文章顯示最後更新時間" width="70%" align="center">}}
[Reference](https://www.jacksonlucky.net/posts/use-lastmod-with-papermod/)  
如果要 override 原本主題的 layouts，複製 themes/PaperMod/layouts 中想改的檔案到 layouts 底下  
  
1. Copy the theme’s `themes/PaperMod/layouts/partials/post_meta.html` to `layouts/partials/post_meta.html`
2. 修改`layouts/partials/post_meta.html`，最後會同時顯示建立日期跟修改日期
```html
{{ $date := .Date.Format "02.01.2006" }} # Add this
{{ $lastmod := .Lastmod.Format "02.01.2006" }} # Add this

{{- $scratch := newScratch }}

{{- if not .Date.IsZero -}}
{{- $scratch.Add "meta" (slice (printf "<span title='%s'>%s</span>" (.Date) (.Date | time.Format (default "2006-01-02" site.Params.DateFormat)))) }}
{{- end }}

# Add this part
{{- if ne $lastmod $date -}}
{{- $scratch.Add "meta" (slice (printf "<span title='%s'>(updated %s)</span>" (.Lastmod) (.Lastmod | time.Format (default "January 2, 2006" site.Params.DateFormat)))) }}
{{- end }}

# ... other contents

# Add this
{{- with ($scratch.Get "meta") }}
{{- delimit . "&nbsp;·&nbsp;" -}}
{{- end -}}
```

## Add table of contents 顯示側邊目錄
[Reference](https://www.sulvblog.cn/posts/blog/hugo_toc_side/)  
1. 修改 `hugo.yaml`
```yaml
  showtoc: true
  tocopen: true
```
2. Copy and edit `layouts/partials/toc.html`
~~~html
{{- $headers := findRE "<h[1-6].*?>(.|\n])+?</h[1-6]>" .Content -}}
{{- $has_headers := ge (len $headers) 1 -}}
{{- if $has_headers -}}
<aside id="toc-container" class="toc-container wide">
    <div class="toc">
        <details {{if (.Param "TocOpen") }} open{{ end }}>
            <summary accesskey="c" title="(Alt + C)">
                <span class="details">{{- i18n "toc" | default "Table of Contents" }}</span>
            </summary>

            <div class="inner">
                {{- $largest := 6 -}}
                {{- range $headers -}}
                {{- $headerLevel := index (findRE "[1-6]" . 1) 0 -}}
                {{- $headerLevel := len (seq $headerLevel) -}}
                {{- if lt $headerLevel $largest -}}
                {{- $largest = $headerLevel -}}
                {{- end -}}
                {{- end -}}

                {{- $firstHeaderLevel := len (seq (index (findRE "[1-6]" (index $headers 0) 1) 0)) -}}

                {{- $.Scratch.Set "bareul" slice -}}
                <ul>
                    {{- range seq (sub $firstHeaderLevel $largest) -}}
                    <ul>
                        {{- $.Scratch.Add "bareul" (sub (add $largest .) 1) -}}
                        {{- end -}}
                        {{- range $i, $header := $headers -}}
                        {{- $headerLevel := index (findRE "[1-6]" . 1) 0 -}}
                        {{- $headerLevel := len (seq $headerLevel) -}}

                        {{/* get id="xyz" */}}
                        {{- $id := index (findRE "(id=\"(.*?)\")" $header 9) 0 }}

                        {{- /* strip id="" to leave xyz, no way to get regex capturing groups in hugo */ -}}
                        {{- $cleanedID := replace (replace $id "id=\"" "") "\"" "" }}
                        {{- $header := replaceRE "<h[1-6].*?>((.|\n])+?)</h[1-6]>" "$1" $header -}}

                        {{- if ne $i 0 -}}
                        {{- $prevHeaderLevel := index (findRE "[1-6]" (index $headers (sub $i 1)) 1) 0 -}}
                        {{- $prevHeaderLevel := len (seq $prevHeaderLevel) -}}
                        {{- if gt $headerLevel $prevHeaderLevel -}}
                        {{- range seq $prevHeaderLevel (sub $headerLevel 1) -}}
                        <ul>
                            {{/* the first should not be recorded */}}
                            {{- if ne $prevHeaderLevel . -}}
                            {{- $.Scratch.Add "bareul" . -}}
                            {{- end -}}
                            {{- end -}}
                            {{- else -}}
                            </li>
                            {{- if lt $headerLevel $prevHeaderLevel -}}
                            {{- range seq (sub $prevHeaderLevel 1) -1 $headerLevel -}}
                            {{- if in ($.Scratch.Get "bareul") . -}}
                        </ul>
                        {{/* manually do pop item */}}
                        {{- $tmp := $.Scratch.Get "bareul" -}}
                        {{- $.Scratch.Delete "bareul" -}}
                        {{- $.Scratch.Set "bareul" slice}}
                        {{- range seq (sub (len $tmp) 1) -}}
                        {{- $.Scratch.Add "bareul" (index $tmp (sub . 1)) -}}
                        {{- end -}}
                        {{- else -}}
                    </ul>
                    </li>
                    {{- end -}}
                    {{- end -}}
                    {{- end -}}
                    {{- end }}
                    <li>
                        <a href="#{{- $cleanedID -}}" aria-label="{{- $header | plainify -}}">{{- $header | safeHTML -}}</a>
                        {{- else }}
                    <li>
                        <a href="#{{- $cleanedID -}}" aria-label="{{- $header | plainify -}}">{{- $header | safeHTML -}}</a>
                        {{- end -}}
                        {{- end -}}
                        <!-- {{- $firstHeaderLevel := len (seq (index (findRE "[1-6]" (index $headers 0) 1) 0)) -}} -->
                        {{- $firstHeaderLevel := $largest }}
                        {{- $lastHeaderLevel := len (seq (index (findRE "[1-6]" (index $headers (sub (len $headers) 1)) 1) 0)) }}
                    </li>
                    {{- range seq (sub $lastHeaderLevel $firstHeaderLevel) -}}
                    {{- if in ($.Scratch.Get "bareul") (add . $firstHeaderLevel) }}
                </ul>
                {{- else }}
                </ul>
                </li>
                {{- end -}}
                {{- end }}
                </ul>
            </div>
        </details>
    </div>
</aside>
<script>
    let activeElement;
    let elements;
    window.addEventListener('DOMContentLoaded', function (event) {
        checkTocPosition();

        elements = document.querySelectorAll('h1[id],h2[id],h3[id],h4[id],h5[id],h6[id]');
         // Make the first header active
         activeElement = elements[0];
         const id = encodeURI(activeElement.getAttribute('id')).toLowerCase();
         document.querySelector(`.inner ul li a[href="#${id}"]`).classList.add('active');
     }, false);

    window.addEventListener('resize', function(event) {
        checkTocPosition();
    }, false);

    window.addEventListener('scroll', () => {
        // Check if there is an object in the top half of the screen or keep the last item active
        activeElement = Array.from(elements).find((element) => {
            if ((getOffsetTop(element) - window.pageYOffset) > 0 && 
                (getOffsetTop(element) - window.pageYOffset) < window.innerHeight/2) {
                return element;
            }
        }) || activeElement

        elements.forEach(element => {
             const id = encodeURI(element.getAttribute('id')).toLowerCase();
             if (element === activeElement){
                 document.querySelector(`.inner ul li a[href="#${id}"]`).classList.add('active');
             } else {
                 document.querySelector(`.inner ul li a[href="#${id}"]`).classList.remove('active');
             }
         })
     }, false);

    const main = parseInt(getComputedStyle(document.body).getPropertyValue('--article-width'), 10);
    const toc = parseInt(getComputedStyle(document.body).getPropertyValue('--toc-width'), 10);
    const gap = parseInt(getComputedStyle(document.body).getPropertyValue('--gap'), 10);

    function checkTocPosition() {
        const width = document.body.scrollWidth;

        if (width - main - (toc * 2) - (gap * 4) > 0) {
            document.getElementById("toc-container").classList.add("wide");
        } else {
            document.getElementById("toc-container").classList.remove("wide");
        }
    }

    function getOffsetTop(element) {
        if (!element.getClientRects().length) {
            return 0;
        }
        let rect = element.getBoundingClientRect();
        let win = element.ownerDocument.defaultView;
        return rect.top + win.pageYOffset;   
    }
</script>
{{- end }}
~~~
3. Copy and Edit `assets/css/extended/blank.css`
```css
:root {
    --nav-width: 1380px;
    --article-width: 650px;
    --toc-width: 300px;
}

.toc {
    margin: 0 2px 40px 2px;
    border: 1px solid var(--border);
    background: var(--entry);
    border-radius: var(--radius);
    padding: 0.4em;
}

.toc-container.wide {
    position: absolute;
    height: 100%;
    border-right: 1px solid var(--border);
    left: calc((var(--toc-width) + var(--gap)) * -1);
    top: calc(var(--gap) * 2);
    width: var(--toc-width);
}

.wide .toc {
    position: sticky;
    top: var(--gap);
    border: unset;
    background: unset;
    border-radius: unset;
    width: 100%;
    margin: 0 2px 40px 2px;
}

.toc details summary {
    cursor: zoom-in;
    margin-inline-start: 20px;
    padding: 12px 0;
}

.toc details[open] summary {
    font-weight: 500;
}

.toc-container.wide .toc .inner {
    margin: 0;
}

.active {
    font-size: 110%;
    font-weight: 600;
}

.toc ul {
    list-style-type: circle;
}

.toc .inner {
    margin: 0 0 0 20px;
    padding: 0px 15px 15px 20px;
    font-size: 16px;

    max-height: 83vh;
    overflow-y: auto;
}

.toc .inner::-webkit-scrollbar-thumb {  
    background: var(--border);
    border: 7px solid var(--theme);
    border-radius: var(--radius);
}

.toc li ul {
    margin-inline-start: calc(var(--gap) * 0.5);
    list-style-type: none;
}

.toc li {
    list-style: none;
    font-size: 0.95rem;
    padding-bottom: 5px;
}

.toc li a:hover {
    color: var(--secondary);
}
```
## Deploy to Firebase
1. Build Hugo static file
```bash
hugo
```
