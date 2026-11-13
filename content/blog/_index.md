---
title: "文章"
type: blog
layout: wide
description: "技术笔记、工作经验、故障复盘与学习记录"
---

{{< blog-list.inline >}}
{{- $readMore := (T "readMore") | default "Read more →" -}}
{{- $pages := partial "utils/sort-pages" (dict "page" .Page "by" site.Params.blog.list.sortBy "order" site.Params.blog.list.sortOrder) -}}
{{- $blogURL := .Page.RelPermalink -}}
{{- $categoryNames := dict
  "essays" "随笔"
  "incident-review" "故障复盘"
  "operations-guide" "运维手册"
  "tech-sharing" "技术分享"
  "design-research" "设计调研"
  "learning-notes" "学习记录"
  "other" "其他"
-}}
{{- $categoryOrder := slice "essays" "incident-review" "operations-guide" "tech-sharing" "design-research" "learning-notes" "other" -}}
{{- $categoryCounts := dict -}}
{{- $tagCounts := dict -}}
{{- $tagNames := slice -}}
{{- range $pages -}}
  {{- $category := index .Params.categories 0 | default "" -}}
  {{- if $category -}}
    {{- $categoryCounts = merge $categoryCounts (dict $category (add (index $categoryCounts $category | default 0) 1)) -}}
  {{- end -}}
  {{- range .Params.tags -}}
    {{- $tag := . -}}
    {{- $tagCounts = merge $tagCounts (dict $tag (add (index $tagCounts $tag | default 0) 1)) -}}
    {{- $tagNames = $tagNames | append $tag -}}
  {{- end -}}
{{- end -}}

<div style="display: flex; flex-wrap: wrap; gap: 2rem; align-items: flex-start;">
  <aside id="blog-filter" style="flex: 0 0 16rem; max-width: 100%;" aria-label="文章筛选">
    <div class="hx:mb-8 hx:rounded-lg hx:border hx:border-gray-200 hx:p-4 hx:dark:border-neutral-800">
      <div class="hx:flex hx:items-center hx:justify-between">
        <span class="hx:font-semibold">筛选文章</span>
      </div>

      <fieldset class="hx:mt-4">
        <legend class="hx:mb-2 hx:text-sm hx:font-semibold" style="width: 100%;">
          <span style="display: flex; align-items: center; justify-content: space-between; width: 100%;">
            <span>分类（可多选）</span>
            <button type="button" class="hx:text-sm hx:underline hx:underline-offset-4" style="font-weight: normal;" data-blog-category-reset hidden>清除分类</button>
          </span>
        </legend>
        <div>
          {{- range $category := $categoryOrder }}
            <label class="hx:flex hx:items-center hx:justify-between hx:py-1 hx:text-sm hx:cursor-pointer">
              <span>
                <input type="checkbox" class="hx:mr-2" data-blog-category value="{{ $category }}">
                {{ index $categoryNames $category }}
              </span>
              <span class="hx:opacity-50">{{ index $categoryCounts $category | default 0 }}</span>
            </label>
          {{- end }}
        </div>
      </fieldset>

      <fieldset class="hx:mt-4">
        <legend class="hx:mb-2 hx:text-sm hx:font-semibold" style="width: 100%;">
          <span style="display: flex; align-items: center; justify-content: space-between; width: 100%;">
            <span>标签（可多选）</span>
            <button type="button" class="hx:text-sm hx:underline hx:underline-offset-4" style="font-weight: normal;" data-blog-tag-reset hidden>清除标签</button>
          </span>
        </legend>
        <p class="hx:mb-2 hx:text-sm hx:opacity-70" data-blog-tags-hint hidden>当前分类下暂无标签。</p>
        <div class="hextra-scrollbar" style="max-height: 20rem; overflow-y: auto;">
          {{- range $tag := sort (uniq $tagNames) }}
            <label class="hx:flex hx:items-center hx:justify-between hx:py-1 hx:text-sm hx:cursor-pointer" data-blog-tag-option data-tag="{{ $tag }}">
              <span>
                <input type="checkbox" class="hx:mr-2" data-blog-tag value="{{ $tag }}">
                {{ $tag }}
              </span>
              <span class="hx:opacity-50" data-blog-tag-count>{{ index $tagCounts $tag }}</span>
            </label>
          {{- end }}
        </div>
      </fieldset>

      <p class="hx:mt-4 hx:mb-0 hx:text-sm hx:opacity-70" data-blog-status aria-live="polite">共 {{ len $pages }} 篇文章</p>
    </div>
  </aside>

  <div style="flex: 1 1 24rem; min-width: 0;">
    {{- range $pages -}}
      {{- $category := index .Params.categories 0 | default "" -}}
      <div class="hx:mb-10" data-blog-article data-category="{{ $category }}" data-tags="{{ delimit .Params.tags "|" }}">
        <h3>
          <a style="color: inherit; text-decoration: none;" class="hx:block hx:font-semibold hx:mt-8 hx:text-2xl" href="{{ .RelPermalink }}">{{ .Title }}</a>
        </h3>
        <div class="hx:text-sm hx:leading-7">
          {{- if $category }}
            <a class="hx:inline-block hx:whitespace-nowrap hx:mr-2 hx:text-gray-500 hx:hover:text-gray-900 hx:dark:text-gray-400 hx:dark:hover:text-gray-100" href="{{ $blogURL }}?category={{ $category | urlquery }}">{{ index $categoryNames $category | default $category }}</a>
          {{- end }}
          {{- if site.Params.blog.list.displayTags }}
            {{- range $tag := .Params.tags }}
              <a class="hx:inline-block hx:whitespace-nowrap hx:mr-2 hx:text-gray-500 hx:hover:text-gray-900 hx:dark:text-gray-400 hx:dark:hover:text-gray-100" href="{{ $blogURL }}?tag={{ $tag | urlquery }}">#{{ $tag }}</a>
            {{- end }}
          {{- end }}
        </div>
        <p class="hx:opacity-80 hx:mt-4 hx:leading-7">{{ partial "utils/page-description" . }}</p>
        <p class="hx:opacity-80 hx:mt-1 hx:leading-7">
          <a class="hx:text-[color:hsl(var(--primary-hue),100%,50%)] hx:underline hx:underline-offset-2 hx:decoration-from-font" href="{{ .RelPermalink }}" aria-label="{{ printf "%s %s" $readMore .Title }}">{{ $readMore }}</a>
        </p>
        <p class="hx:opacity-50 hx:text-sm hx:mt-4 hx:leading-7">{{ partial "utils/format-date" .Date }}</p>
      </div>
    {{- else -}}
      <p class="hx:opacity-70">暂无文章。</p>
    {{- end -}}
    <p class="hx:opacity-70" data-blog-empty hidden>没有符合当前分类和标签的文章。</p>
  </div>
</div>

<script>
  (() => {
    const filter = document.getElementById("blog-filter");
    if (!filter) return;

    const categoryInputs = Array.from(filter.querySelectorAll("[data-blog-category]"));
    const tagInputs = Array.from(filter.querySelectorAll("[data-blog-tag]"));
    const tagOptions = Array.from(filter.querySelectorAll("[data-blog-tag-option]"));
    const categoryReset = filter.querySelector("[data-blog-category-reset]");
    const tagReset = filter.querySelector("[data-blog-tag-reset]");
    const status = filter.querySelector("[data-blog-status]");
    const tagsHint = filter.querySelector("[data-blog-tags-hint]");
    const articles = Array.from(document.querySelectorAll("[data-blog-article]"));
    const empty = document.querySelector("[data-blog-empty]");

    const selectedValues = (inputs) => new Set(
      inputs.filter((input) => input.checked).map((input) => input.value)
    );

    const updateTagOptions = (selectedCategories) => {
      const counts = new Map();
      for (const article of articles) {
        if (selectedCategories.size && !selectedCategories.has(article.dataset.category)) continue;
        const tags = article.dataset.tags ? article.dataset.tags.split("|") : [];
        for (const tag of new Set(tags)) {
          counts.set(tag, (counts.get(tag) || 0) + 1);
        }
      }

      for (const option of tagOptions) {
        const tag = option.dataset.tag;
        const count = counts.get(tag) || 0;
        const input = option.querySelector("[data-blog-tag]");
        if (!count) input.checked = false;
        option.hidden = !count;
        option.style.display = count ? "flex" : "none";
        option.querySelector("[data-blog-tag-count]").textContent = count;
      }
      tagsHint.hidden = counts.size !== 0;
    };

    const applyFilters = (updateURL = true) => {
      const selectedCategories = selectedValues(categoryInputs);
      updateTagOptions(selectedCategories);
      const selectedTags = selectedValues(tagInputs);
      let visibleCount = 0;

      for (const article of articles) {
        const tags = article.dataset.tags ? article.dataset.tags.split("|") : [];
        const categoryMatches = !selectedCategories.size || selectedCategories.has(article.dataset.category);
        const tagMatches = !selectedTags.size || tags.some((tag) => selectedTags.has(tag));
        const visible = categoryMatches && tagMatches;
        article.hidden = !visible;
        if (visible) visibleCount += 1;
      }

      const filtering = Boolean(selectedCategories.size || selectedTags.size);
      status.textContent = filtering ? `筛选结果：${visibleCount} 篇文章` : `共 ${visibleCount} 篇文章`;
      categoryReset.hidden = !selectedCategories.size;
      tagReset.hidden = !selectedTags.size;
      empty.hidden = visibleCount !== 0;

      if (updateURL) {
        const url = new URL(window.location.href);
        url.searchParams.delete("category");
        url.searchParams.delete("tag");
        for (const input of categoryInputs) {
          if (input.checked) url.searchParams.append("category", input.value);
        }
        for (const input of tagInputs) {
          if (input.checked) url.searchParams.append("tag", input.value);
        }
        window.history.replaceState(null, "", url);
      }
    };

    for (const input of [...categoryInputs, ...tagInputs]) {
      input.addEventListener("change", () => applyFilters());
    }
    categoryReset.addEventListener("click", () => {
      for (const input of categoryInputs) input.checked = false;
      applyFilters();
    });
    tagReset.addEventListener("click", () => {
      for (const input of tagInputs) input.checked = false;
      applyFilters();
    });

    const params = new URLSearchParams(window.location.search);
    const initialCategories = new Set(params.getAll("category"));
    const initialTags = new Set(params.getAll("tag"));
    for (const input of categoryInputs) input.checked = initialCategories.has(input.value);
    for (const input of tagInputs) input.checked = initialTags.has(input.value);
    applyFilters(false);
  })();
</script>
{{< /blog-list.inline >}}
