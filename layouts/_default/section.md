# {{ .Site.Title }}

[![](https://img.shields.io/badge/blog-summinglearn-green)](https://summinglearn.com)
[![push public file to coding](https://github.com/summingyu/blog-hugo-page/actions/workflows/hg-pages.yml/badge.svg)](https://github.com/summingyu/blog-hugo-page/actions/workflows/hg-pages.yml)

{{ T "archiveCounter" (len .Data.Pages) }}

{{- $posts := .Data.Pages.ByDate.Reverse }}
{{- range $index, $post := $posts }}
  {{- $thisYear := $post.Date.Format "2006" }}
  {{- $lastPost := $index | add -1 | index $posts }}
  {{- $postPath := replace $post.File.Path " " "%20" }}
  {{- if or (eq $index 0) ( ne ($lastPost.Date.Format "2006") $thisYear ) }}
## {{ $thisYear }}
  {{- end }}
- {{ $post.Date.Format "01-02" }} [{{ .Title }}](content/{{ $postPath }})
{{- end }}