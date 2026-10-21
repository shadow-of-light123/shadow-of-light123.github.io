---
title: 'JSON Schema：一份合格的 Schema 长啥样'
date: 2026-10-21
draft: false
categories: ['前端']
tags: ['JSON', '数据校验']
---

JSON Schema 是一种用来描述 JSON 数据结构的规范，广泛用在接口参数校验、表单校验、配置文件约束等场景。但「写了一个 schema」和「写了一份合格的 schema」是两回事——前者只要求是个 JSON，后者要能真正准确、无歧义地表达数据约束。

## 什么是一份合格的 JSON Schema

一份「合格」的 JSON Schema，至少要满足三层：

1. **是合法 JSON**：不能有注释、尾逗号、单引号等。
2. **符合某个 JSON Schema 草案的 meta-schema**：也就是 schema 本身是合法的 schema。
3. **能清晰表达并验证目标数据结构**：约束不矛盾，能准确描述业务数据。

一句话总结：

> 合格的 JSON Schema = 合法 JSON + 正确声明草案 + 通过 meta-schema 校验 + 约束不矛盾 + 能准确表达业务数据结构。

## 一份合格的 JSON Schema 示例

以一个「系统用户」的 schema 为例（基于 Draft 2020-12）：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/schemas/user.schema.json",
  "title": "User",
  "description": "系统用户",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "minimum": 1
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 50
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "role": {
      "type": "string",
      "enum": ["admin", "user"]
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" },
      "uniqueItems": true,
      "default": []
    },
    "profile": {
      "$ref": "#/$defs/profile"
    }
  },
  "required": ["id", "name", "email", "role"],
  "additionalProperties": false,
  "$defs": {
    "profile": {
      "type": "object",
      "properties": {
        "age": {
          "type": "integer",
          "minimum": 0,
          "maximum": 150
        },
        "city": {
          "type": "string"
        }
      },
      "additionalProperties": false
    }
  },
  "examples": [
    {
      "id": 1,
      "name": "Alice",
      "email": "alice@example.com",
      "role": "user",
      "tags": ["vip"],
      "profile": {
        "age": 30,
        "city": "上海"
      }
    }
  ]
}
```

这份 schema 合格的点：

- `$schema` 声明了使用的草案版本（Draft 2020-12）。
- `$id` 给了唯一标识，便于引用和发布。
- `type` 明确根结构是 object。
- `properties` 描述字段结构，`required` 描述必填字段。
- 用了 `minimum`、`minLength`、`enum`、`items` 等约束。
- 用 `$defs` 和 `$ref` 复用子结构。
- 用 `title`、`description`、`examples` 提供文档信息。
- 用 `additionalProperties: false` 控制未知字段，是否启用要看业务是否允许扩展。

## 满足 Schema 的数据示例

下面两份 JSON 数据都满足上面的 `User` schema。

数据一，普通用户：

```json
{
  "id": 1,
  "name": "Alice",
  "email": "alice@example.com",
  "role": "user",
  "tags": ["vip", "new"],
  "profile": {
    "age": 30,
    "city": "上海"
  }
}
```

数据二，管理员用户：

```json
{
  "id": 2,
  "name": "Bob",
  "email": "bob@example.com",
  "role": "admin",
  "tags": ["ops"],
  "profile": {
    "age": 42,
    "city": "北京"
  }
}
```

## Schema 与数据对照表

| 字段           | Schema 约束                   | 数据一 Alice                | 数据二 Bob                |
| -------------- | ----------------------------- | --------------------------- | ------------------------- |
| `id`           | integer，最小值 1             | `1`，符合                   | `2`，符合                 |
| `name`         | string，长度 1~50             | `"Alice"`，符合             | `"Bob"`，符合             |
| `email`        | string，email 格式            | `"alice@example.com"`，符合 | `"bob@example.com"`，符合 |
| `role`         | enum：`admin` / `user`        | `"user"`，符合              | `"admin"`，符合           |
| `tags`         | 字符串数组，元素唯一          | `["vip", "new"]`，符合      | `["ops"]`，符合           |
| `profile`      | 引用 `$defs/profile`          | 对象，符合                  | 对象，符合                |
| `profile.age`  | integer，0~150                | `30`，符合                  | `42`，符合                |
| `profile.city` | string                        | `"上海"`，符合              | `"北京"`，符合            |
| 必填字段       | `id`、`name`、`email`、`role` | 全部存在，符合              | 全部存在，符合            |
| 额外字段       | 根对象和 `profile` 都不允许   | 无额外字段，符合            | 无额外字段，符合          |

注意：`format: "email"` 在不同校验器里可能是注解而非强制校验，但上面给的邮箱本身是合法格式。

## 合格 Checklist

1. **本身是合法 JSON**：不能有注释、尾逗号、单引号等。
2. **声明 `$schema`**：不要混用 Draft-07 和 2020-12 的写法。
3. **能通过对应 meta-schema 校验**：也就是「schema 本身是合法的 schema」。
4. **结构语义正确**：
   - `type` 是字符串或数组；
   - `required` 是字符串数组；
   - `properties` 是对象；
   - `items` 是 schema；
   - `enum` 是数组；
   - 不要出现 `minimum > maximum` 这种自相矛盾约束。
5. **约束符合业务**：比如 `email` 用 `format: "email"`，但要注意很多验证器把 `format` 当注解，不一定强制校验。
6. **版本兼容**：Draft-07 常用 `definitions`，2020-12 推荐 `$defs`；`$ref` 行为也有差异，别混着抄。
7. **可维护**：复杂 schema 用 `$defs` / `$ref` 拆分；给 `title`、`description`、`examples`；必要时加 `$comment`。

## 常见「不合格」写法

反例一：

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "strng" }
  },
  "required": {
    "name": true
  }
}
```

问题：`strng` 拼错、`required` 应该是数组而不是对象、缺少 `$schema`。

反例二：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "definitions": {
    "x": { "type": "string" }
  }
}
```

问题：在 2020-12 里更推荐 `$defs`，`definitions` 虽常被兼容，但不应作为新写法的首选。

## 如何验证

用 Ajv（Node.js）：

```bash
ajv compile -s user.schema.json
```

或用 Python：

```python
import json
from jsonschema import Draft202012Validator

schema = json.load(open("user.schema.json"))
Draft202012Validator.check_schema(schema)
```

## 总结

**合格的 JSON Schema = 合法 JSON + 正确声明草案 + 通过 meta-schema 校验 + 约束不矛盾 + 能准确表达业务数据结构。** 写完之后，拿一两个真实数据跑一遍校验器，是确认「合格」最直接的方式。
