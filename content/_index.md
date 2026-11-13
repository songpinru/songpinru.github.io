---
title: "Pinru's Home"
layout: hextra-home
toc: false
---

{{< hextra/hero-badge link="blog" >}}后端开发 · 数据平台 · 基础设施{{< /hextra/hero-badge >}}

{{< hextra/hero-headline >}}
  记录问题，沉淀思考。
{{< /hextra/hero-headline >}}

{{< hextra/hero-subtitle >}}
  分享 Kafka、Flink、Hadoop 与分布式系统的实践经验。
  <br class="hx:max-sm:hidden" />
  写下故障排查的证据链，也记录方案设计、源码阅读和长期维护中的细节。
{{< /hextra/hero-subtitle >}}

<div class="hx:mt-6 hx:mb-6 hx:flex hx:flex-wrap hx:gap-3">
  {{< hextra/hero-button text="阅读全部文章" link="blog" >}}
  {{< hextra/hero-button text="查看故障复盘" link="blog?category=incident-review" >}}
</div>

<div class="hx:mb-12"></div>

{{< hextra/hero-section heading="h3" >}}按分类浏览{{< /hextra/hero-section >}}

{{< hextra/feature-grid cols="3" >}}
  {{< hextra/feature-card title="故障复盘" subtitle="故障过程、证据链、根因分析与整改" link="blog?category=incident-review" >}}
  {{< hextra/feature-card title="运维手册" subtitle="部署、升级、迁移与日常维护" link="blog?category=operations-guide" >}}
  {{< hextra/feature-card title="技术分享" subtitle="原理分析、实践经验与机制拆解" link="blog?category=tech-sharing" >}}
  {{< hextra/feature-card title="设计调研" subtitle="方案设计、技术选型与验证记录" link="blog?category=design-research" >}}
  {{< hextra/feature-card title="学习记录" subtitle="知识归纳、源码阅读与学习实践" link="blog?category=learning-notes" >}}
  {{< hextra/feature-card title="随笔" subtitle="零散思考、观点与个人记录" link="blog?category=essays" >}}
{{< /hextra/feature-grid >}}

<div class="hx:mt-12"></div>

{{< hextra/hero-section heading="h3" >}}最近更新{{< /hextra/hero-section >}}

{{< recent-posts.inline >}}
{{- $categoryNames := dict
  "essays" "随笔"
  "incident-review" "故障复盘"
  "operations-guide" "运维手册"
  "tech-sharing" "技术分享"
  "design-research" "设计调研"
  "learning-notes" "学习记录"
  "other" "其他"
-}}
{{- $pages := first 5 (where site.RegularPages "Section" "blog").ByDate.Reverse -}}
<div>
  {{- range $pages -}}
    {{- $category := index .Params.categories 0 | default "" -}}
    <article class="hx:py-4 hx:border-b hx:border-gray-200 hx:dark:border-neutral-800">
      <div class="hx:flex hx:items-center hx:justify-between hx:text-sm hx:opacity-60">
        <span>{{ index $categoryNames $category | default $category }}</span>
        <time datetime="{{ .Date.Format "2006-01-02" }}">{{ partial "utils/format-date" .Date }}</time>
      </div>
      <h3 class="hx:mt-2 hx:mb-2 hx:text-xl hx:font-semibold">
        <a style="color: inherit; text-decoration: none;" href="{{ .RelPermalink }}">{{ .Title }}</a>
      </h3>
      <p class="hx:mt-2 hx:mb-0 hx:leading-7 hx:opacity-70">{{ partial "utils/page-description" . }}</p>
    </article>
  {{- end -}}
</div>
<p class="hx:mt-6 hx:text-sm">
  <a class="hx:underline hx:underline-offset-4" href="{{ "blog" | relURL }}">查看全部文章 →</a>
</p>
{{< /recent-posts.inline >}}
