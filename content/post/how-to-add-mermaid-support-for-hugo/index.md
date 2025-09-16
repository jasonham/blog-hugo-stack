---
title: hugo中添加mermaid支持
description: mermaid介绍，如何添加mermaid支持，常用的图表
date: 2024-07-01T01:04:18+08:00
lastmod: 2024-07-01T01:04:18+08:00
slug: how-to-add-mermaid-support-for-hugo

tags:
  - mermaid
categories:
  - hugo
---

## [mermaid](https://mermaid.js.org) 介绍

mermaid 是一个开源的 JavaScript 库，用于创建图表和流程图。
基于 JavaScript 的图表和图表工具，可以在 Markdown 文档中使用文本定义，用以动态创建和修改图表。

---

## 添加 mermaid 支持

### layouts/partials/head/custom.html

```html
<script src="https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({
    startOnLoad: true,
    theme: "default", // 可选：'dark', 'forest', 'neutral'
    securityLevel: "loose", // 如果图表包含交互或链接
  });
</script>
```

### layouts/shortcodes/mermaid.html

```html
<div class="mermaid">{{- .Inner -}}</div>
```

### md 文件中使用 shortcode 标签包裹

```markdown
{{</* mermaid */>}}

# mermaid 指令码

{{</* /mermaid */>}}
```

---

## mermaid 支持多种类型的图表，以下是常用的图表指令：

### 1. 流程图 (Flowchart)

> - 早期版本中使用 `graph LR` 指令
> - graph LR 实际上是 flowchart（流程图）的一种语法形式，它属于 mermaid 的流程图功能的一部分
> - 在较新版本的 mermaid 中，推荐使用 flowchart 作为关键字来定义流程图，例如 flowchart LR，而 graph 是旧版本的语法
> - graph LR 和 flowchart LR 功能基本相同，都是定义从左到右方向的流程图，但 flowchart 是更现代化和标准化的写法

```

flowchart LR
A[开始] --> B[步骤 1]
B --> C[步骤 2]
C --> D[(结束)]

```

{{<mermaid>}}
flowchart LR
A[开始] --> B[步骤 1]
B --> C[步骤 2]
C --> D[(结束)]
{{</mermaid>}}

### 2. 序列图 (Sequence Diagram)

```

sequenceDiagram
participant A as 用户
participant B as 系统
A->>B: 登录请求
B-->>A: 登录响应

```

{{<mermaid>}}
sequenceDiagram
participant A as 用户
participant B as 系统
A->>B: 登录请求
B-->>A: 登录响应
{{</mermaid>}}

### 3. 甘特图 (Gantt Diagram)

```

gantt
title 项目计划
dateFormat YYYY-MM-DD
section 第一阶段
任务 1 :done, des1, 2023-01-01, 2023-01-10

```

{{<mermaid>}}
gantt
title 项目计划
dateFormat YYYY-MM-DD
section 第一阶段
任务 1 :done, des1, 2023-01-01, 2023-01-10
{{</mermaid>}}

### 4. 类图 (Class Diagram)

```

classDiagram
Animal <|-- Dog
Animal: +name
Animal: +eat()
Dog: +bark()

```

{{<mermaid>}}
classDiagram
Animal <|-- Dog
Animal: +name
Animal: +eat()
Dog: +bark()
{{</mermaid>}}

### 5. 状态图 (State Diagram)

```

stateDiagram
[*] --> Still
Still --> Moving
Moving --> Still

```

{{<mermaid>}}
stateDiagram
[*] --> Still
Still --> Moving
Moving --> Still
{{</mermaid>}}

### 6. 饼图 (Pie Chart)

```

pie
title 市场份额
"苹果" : 40
"橙子" : 30
"香蕉" : 30

```

{{<mermaid>}}
pie
title 市场份额
"苹果" : 40
"橙子" : 30
"香蕉" : 30
{{</mermaid>}}

### 7. 实体关系图 (ER Diagram)

```

erDiagram
CUSTOMER ||--o{ ORDER : places
ORDER ||--|{ LINE-ITEM : contains

```

{{<mermaid>}}
erDiagram
CUSTOMER ||--o{ ORDER : places
ORDER ||--|{ LINE-ITEM : contains
{{</mermaid>}}

## 这些是 `mermaid` 中最常用的图表类型指令，每种图表都有其特定的语法和用途。
