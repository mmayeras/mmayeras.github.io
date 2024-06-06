---
title: "hugo loveit admonition cheatsheet"
date: 2023-03-24
tags:
- hugo
categories:
- misc
draft: true
---


# Hugo LoveIt admonition


<!--more-->

> `admonition`(警告/訓詞) shortcode 提供 **12** 種樣式讓你可以在文章中放入各種提示
>
> Markdown 或 HTML 也都可以寫入 `admonition`

## Note
{{< admonition >}}
  Some note tips...
{{< /admonition >}}
```markdown
  {{</* admonition note "Note" */>}}
    Some note tips...
  {{</* /admonition */>}}
```

## Abstract
{{< admonition abstract >}}
  Some abstract tips...
{{< /admonition >}}
```markdown
  {{</* admonition "Abstract" */>}}
    Some abstract tips...
  {{</* /admonition */>}}
```

## Info
{{< admonition info >}}
  Some info tips...
{{< /admonition >}}
```markdown
  {{</* admonition info "Info" */>}}
    Some info tips...
  {{</* /admonition */>}}
```

## Tip
{{< admonition tip >}}
  Some tip tips...
{{< /admonition >}}
```markdown
  {{</* admonition tip "Tip" */>}}
    Some tip tips...
  {{</* /admonition */>}}
```

## Success
{{< admonition success >}}
  Some success tips...
{{< /admonition >}}
```markdown
  {{</* admonition success "Success" */>}}
    Some success tips...
  {{</* /admonition */>}}
```

## Question
{{< admonition question >}}
  Some question tips...
{{< /admonition >}}
```markdown
  {{</* admonition question "Question" */>}}
    Some question tips...
  {{</* /admonition */>}}
```

## Warning
{{< admonition warning >}}
  Some warning tips...
{{< /admonition >}}
```markdown
  {{</* admonition warning "Warning" */>}}
    Some warning tips...
  {{</* /admonition */>}}
```

## Failure
{{< admonition failure >}}
  Some failure tips...
{{< /admonition >}}
```markdown
  {{</* admonition failure "Failure" */>}}
    Some failure tips...
  {{</* /admonition */>}}
```

## Danger
{{< admonition danger >}}
  Some danger tips...
{{< /admonition >}}
```markdown
  {{</* admonition danger "Danger" */>}}
    Some danger tips...
  {{</* /admonition */>}}
```

## Bug
{{< admonition bug >}}
  Some bug tips...
{{< /admonition >}}
```markdown
  {{</* admonition bug "Bug" */>}}
    Some bug tips...
  {{</* /admonition */>}}
```

## Example
{{< admonition example >}}
  Some example tips...
{{< /admonition >}}
```markdown
  {{</* admonition * "Example" */>}}
    Some example tips...
  {{</* /admonition */>}}
```

## Quote
{{< admonition quote >}}
  Some quote tips...
{{< /admonition >}}
```markdown
  {{</* admonition quote "Quote" */>}}
    Some quote tips...
  {{</* /admonition */>}}
```

## admonition 有三個參數可以設定，你可以選擇以下兩種寫法

```markdown
{{</* admonition type=tip title="This is a tip" open=false */>}}

{{</* /admonition */>}}
```
OR
```markdown
{{</* admonition tip "This is a tip" false */>}}

{{</* /admonition */>}}
```