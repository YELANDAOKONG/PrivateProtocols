# 公共组织目录事件消息协议

---

# 定义

**索引位置：** 事件消息文件和索引文件在 Web 服务器上的根目录（应当允许被HTTP/HTTPS访问），如：

```text
https://example.com/events/
```

**事件消息：** 由标题、图标、链接、描述、内容等信息构成的一段 JSON 或 XML 格式数据。

**事件消息ID：** 用于标识事件消息的唯一ID，一般为自然数。

---

# URI

此版本协议支持使用 URI 链接表示一个特定的“索引位置”；

URI 链接格式规范如下：

```text
协议方案名://用户名:密码@服务器地址:服务器端口/带层次的文件路径/?查询字符串#片段标识符
```

其中，协议方案名为 **poiseip** 或 **poiseips**，根据HTTP/HTTPS区分，即基于 HTTP 使用 poiseip ，基于 HTTPS 使用 poiseips ；

**URI 规范以 RFC3986 为准。**

### URI 示例：

（以下链接均表示在 http://example.com/events/ 下存的索引位置）

```text
# 基于 HTTP
poiseip://example.com/events/

# 基于 HTTPS
poiseips://example.com/events/

# 指定端口
poiseip://example.com:80/events/
poiseips://example.com:443/events/

# 携带身份验证信息
poiseips://user:password@example.com/events/
poiseips://user:password@example.com:443/events/

# 指定具体事件（使用片段标识符）
poiseips://example.com/events/#1000
poiseips://user:password@example.com/events/?version=11000-0#1000
```

## 片段标识符

片段标识符用于在 URI 中直接标识某个具体事件，URI 中的片段标识符字段直接作为事件ID解析。

---

# 安全

此协议为组织协议的附属，因此使用组织协议提供的加密管理以确保安全。

### 索引加密认证

使用此方式需要加密除 index.dat 外所有文件，并将加密相关元数据存储到 index.dat 文件中。

> **注意：协议无法完全加密事件消息数据目录，攻击者仍可通过遍历的方式列出索引位置上的事件消息（ID），并推测它们的发布时间。**

出于扩展性考虑，本协议不限制加密算法，您可以使用任何加密算法加密您的数据，包括任何非对称加密算法。

被加密的文件遵循以下格式：

```json
{
    "type": "text", // @NULLABLE 标识文件的类型 (为NULL表示不设置/未加密)
    "data": "<BASE64...>"   // Base64 编码的原文件数据
}
```

加密元数据存储在 index.dat->security->encryption 中，本协议不对加密算法做规定；
这意味着您可以在元数据中直接包含解密密钥（相当于不加密），也可以包含非对称加密的公钥、证书等，也可以什么都不包含。

---

# 规范

> 下面列出的文件均存储在“索引位置”中。

## index.dat

该文件存储此“事件索引服务”的“名称”、“简介”，“协议版本”等数据。

以下为一个完整的标准示例：

```json
{
    "security": {                           // @NULLABLE 安全相关选项（可选，如无该字段标识不使用安全及加密选项）
        "encryption": {                     // @NULLABLE 加密相关选项（可选）
            "enabled": false,               // @NULLABLE 是否启用加密 (可为NULL，为NULL默认False)
            "algorithm": null,              // 加密算法名称
            "specification": null,          // 密钥规格信息
            "parameters": [],               // 密钥参数信息
            "keys": {
                "public": {             // @NULLABLE 公钥（可选）
                    "format": null,     // @NULLABLE 密钥格式【可为Null】
                    "data": null,       // @NULLABLE 密钥内容【可为Null】
                    "extension": {}
                },
                "private": {            // @NULLABLE 私钥（可选）
                    "format": null,     // @NULLABLE 密钥格式【可为Null】
                    "data": null,       // @NULLABLE 密钥内容【可为Null】
                    "extension": {}
                },
                "protected": {          // @NULLABLE 对称加密密码（可选）
                    "format": null,     // @NULLABLE 密钥格式【可为Null】
                    "data": null,       // @NULLABLE 密钥内容【可为Null】
                    "extension": {}
                },
                "chain": {              // @NULLABLE 证书链（可选）
                    "format": null,     // @NULLABLE 证书格式【可为Null】
                    "data": null,       // @NULLABLE 证书内容【可为Null】
                    "extension": {}
                },
                "extension": {}             // 扩展字段（根据使用场景自行增加，本协议不做规定）
            },
            "extension": {}                 // 扩展字段（根据使用场景自行增加，本协议不做规定）
        },
        "extension": {}
    },
    "extension": {}                         // 扩展字段（根据使用场景自行增加，本协议不做规定）
}
```

> **上述示例中 security 及 security->encryption 均为可选字段，为了降低复杂性，建议仅在需要时添加。**

## counter.dat

该文件存储上一个“事件消息”的ID和对应的发布时间。

> 应用程序可在其本地储存一份 Counter 文件，以便判断服务器有哪些新发布且未被获取的事件消息。

**客户端可通过轮询该文件快速判断服务器上是否存在新的事件消息，而不必轮询 Index 和 Data 文件。**

*此文件用于客户端轮询。*

以下为一个完整的标准示例：

```json
{
    "status": true,                        // 用于判断服务器是否存在事件消息 (为TRUE表示存在，为FALSE表示服务器没有事件消息)
    "latest": 1000,                        // 最新的事件消息ID
    "time": 1300000000000,                 // 13位时间戳，标识最新事件消息的发布时间
    "extension": {}                        // 扩展字段（根据使用场景自行增加，本协议不做规定）
}
```

## data.dat

该文件存储每一个事件消息的ID和其部分的元数据。

> 客户端可通过读取该文件快速筛选“类型”、“标签”、“分类”，而不必逐个获取完整的事件消息文件。

以下为一个完整的标准示例：

```json
{
    "index": [                             // 此处为 JSON 数组，所有的事件消息应当按照特定顺序在此列出
        {
            "id": 1000,                    // 事件消息ID（以下元数据应当和事件消息文件中的值一致）
            "hide": false,                 // 是否隐藏，如此值为 true ，客户端应当忽略该消息（无论标签分类是否匹配）；默认为 false
            "classes": {
                "type": [                  // 类型：用于区分指定消息（如“用户可读”，“程序可读”等），本协议不做规定；此处仅作示例；允许一条事件消息存在多个类型，可为空列表
                    "normal"               // （仅示例，无规定）
                ],
                "categories": [            // 分类：用于详细区分指定消息，本协议不做规定；此处仅作示例；允许一条事件消息存在多个分类，可为空列表
                    "system"               // （仅示例，无规定）
                ],
                "tags": [                  // 标签：用于详细区分指定消息，本协议不做规定；此处仅作示例；允许一条事件消息存在多个标签，可为空列表
                    "test"                 // （仅示例，无规定）
                ]
            },
            "time": 1300000000000,         // 事件消息发布时间
            "extension": {}                // 扩展字段（根据使用场景自行增加，本协议不做规定）
        }
    ],
    "extension": {}                        // 扩展字段（根据使用场景自行增加，本协议不做规定）
}
```

## data/<事件消息ID>.dat

data 目录下的所有 JSON 文件，每个 JSON 文件标识一条事件消息，其文件名由其ID构成；如：

```text
data/1000.dat
data/1001.dat
data/1002.dat
data/1003.dat
data/1004.dat
data/1005.dat
......
```

以下为一个完整的标准示例：

```json
{
    "id": 1000,                            // 事件消息ID
    "hide": false,                         // 是否隐藏，如此值为 true ，客户端应当忽略该消息（无论标签分类是否匹配）；默认为 false
    "classes": {
        "type": [                          // 类型：用于区分指定消息（如“用户可读”，“程序可读”等），本协议不做规定；此处仅作示例；允许一条事件消息存在多个类型，可为空列表
            "normal"                       // （仅示例，无规定）
        ],
        "categories": [                    // 分类：用于详细区分指定消息，本协议不做规定；此处仅作示例；允许一条事件消息存在多个分类，可为空列表
            "system"                       // （仅示例，无规定）
        ],
        "tags": [                          // 标签：用于详细区分指定消息，本协议不做规定；此处仅作示例；允许一条事件消息存在多个标签，可为空列表
            "test"                         // （仅示例，无规定）
        ]
    },
    "time": 1300000000000,                 // 事件消息发布时间
    "data": {                              // 此处存放事件消息的具体数据
        "title": "System Test",            // 事件消息的标题（不可为NULL）
        "icon": null,                      // 事件消息的图标（可为NULL，表示没有图标）
        "type": "text",                    // 事件消息的类型（不可为NULL），用于表示正文内容的类型；可为任意类型，本协议不做详细规定，可根据使用场景自主区分（如“text”、“md”、“html”等）
        "uri": null,                       // 事件消息的引用链接（可为NULL，表示没有引用链接）；可为任意格式/协议，本协议不做详细规定，可根据使用场景自主区分（如将本协议用于“发布公告”时，这里可以是公告的链接等）
        "describe": "Hello, World! ",                                       // 事件消息的描述简介（不可为NULL）
        "content": [                                                        // 以换行符切割的事件消息正文数组（可为空）；按照一定的顺序排列，客户端在读取时应当自动在字符串之间添加换行符以合并为正文
            "Hello World, Hello World! "                                    // （仅示例，无规定）
        ]
    },
    "files": [                                        // @NULLABLE 此字段标识事件消息附加的资源文件列表（如不增加资源文件，可不添加此字段或将本字段内URI值设为Null）
        {
            "uri": "https://exanmple.com/file.mp3",      // @NULLABLE 资源文件URI地址（可为Null，不增加此字段或值为Null表示无资源文件）
            "type": "audio/mpeg",                        // @NULLABLE 资源类型（可为Null，不增加此字段或值为Null表示未知的资源类型，由客户端在需要时自行判断）
            "length": -1                                 // @NULLABLE 资源大小（不增加此字段或值为-1表示未知的资源大小，由客户端在需要时自行判断）
        }
    ],
    "extension": {}                        // 扩展字段（根据使用场景自行增加，本协议不做规定）
}
```

## classics.json

> **此文件目前保留不使用。**

该文件独立于事件存储/索引系统，用于提供对 RSS 订阅协议的兼容。

以下为一个完整的标准示例：

```json
{
    "channels": [
        {
            "title": "Example Channel",                                 // 同RSS协议：title
            "link": "https://exanmple.com/",                            // 同RSS协议：link
            "description": "This is an example channel.",               // 同RSS协议：description
            "optional": {                                               // （RSS协议中的可选字段）
                "language": "en-us",                                    // 同RSS协议：language
                "copyright": "Copyright (c) 2024 Example Channel",      // 同RSS协议：copyright
                "editor": "channel@example.com",                        // 同RSS协议：managingEditor
                "master": "channel@example.com",                        // 同RSS协议：webMaster
                "date": {
                    "publish": 1300000000000,                           // 同RSS协议：pubDate
                    "update": 1300000000000                             // 同RSS协议：lastBuildDate
                },
                "categories": [                                         // 同RSS协议：category
                    "Example Category"
                ],
                "generator": "Event Indexes Server",                    // 同RSS协议：generator
                "document": "https://exanmple.com/document/",           // 同RSS协议：docs
                "cloud": {                                              // 同RSS协议：cloud（XML属性转为子元素）
                    "domain": "example.com",
                    "port": 80,
                    "path": "/event/indexes/",
                    "procedure": "test",
                    "protocol": "soap"
                },
                "ttl": 60,                                              // 同RSS协议：ttl
                "image": {                                              // 同RSS协议：image（XML属性转为子元素）
                    "url": "https://exanmple.com/image.png",
                    "title": "Example Channel",
                    "link": "https://exanmple.com/",
                    "width": 640,
                    "height": 480
                },
                "rating": "safe",                                        // 同RSS协议：rating
                "input": {                                               // 同RSS协议：textInput（XML属性转为子元素）
                    "name": "test",
                    "title": "Test",
                    "description": "This is a test.",
                    "link": "https://exanmple.com/test/"
                },
                "skip": {                                                // 非RSS协议字段（本协议扩展）
                    "years": 0,                                          // 非RSS协议字段（本协议扩展）
                    "months": 0,                                         // 非RSS协议字段（本协议扩展）
                    "days": 0,                                           // 同RSS协议：skipDays
                    "hours": 0,                                          // 同RSS协议：skipHours
                    "minutes": 0,                                        // 非RSS协议字段（本协议扩展）
                    "seconds": 0                                         // 非RSS协议字段（本协议扩展）
                }
            },
            "items": [
                {
                    "title": "Example Item",                                        // 同RSS协议：title
                    "link": "https://exanmple.com/",                                // 同RSS协议：link
                    "description": "This is an example item.",                      // 同RSS协议：description
                    "optional": {                                                   // （RSS协议中的可选字段）
                        "author": "item@example.com",                               // 同RSS协议：author
                        "categories": [],                                           // 同RSS协议：category
                        "comments": "https://exanmple.com/comments/",               // 同RSS协议：comments
                        "enclosure": {                                              // 同RSS协议：enclosure（XML属性转为子元素）
                            "url": "https://exanmple.com/enclosure.mp3",
                            "length": 1024,
                            "type": "audio/mpeg"
                        },
                        "guid": {                                                   // 同RSS协议：guid（XML属性转为子元素）
                            "value": "https://exanmple.com/",
                            "isPermaLink": true
                        },
                        "date": {
                            "publish": 1300000000000,                               // 同RSS协议：pubDate
                            "update": 1300000000000                                 // 非RSS协议字段（本协议扩展）
                        },
                        "source": "https://exanmple.com/source/",                   // 同RSS协议：source
                        "content": {                                                // 非RSS协议字段（本协议扩展）
                            "format": "html",                                       // 正文格式类型
                            "value": [                                              // 以换行符切割的事件消息正文数组（可为空）；按照一定的顺序排列，客户端在读取时应当自动在字符串之间添加换行符以合并为正文
                                "<p>This is an example item.</p>"                   // ...
                            ]
                        },
                        "extension": {}                                             // 扩展字段（根据使用场景自行增加，本协议不做规定）
                    }
                }
            ],
            "extension": {}                                                         // 扩展字段（根据使用场景自行增加，本协议不做规定）
        }
    ]
}
```

## classics/*.json

> **此目录目前保留不使用。**


