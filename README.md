# Android Locale Parser

Parse simple text as an Android XML Resource files powered by Vue.js, Vite.js and WindiCSS

### Basic Usage

This will generated each folder corresponding to the locale.


```
locale: Content
locale2: Content2
locale3: Content3
```

Output:

`locale`

```xml
<string name="">Content</string>
```
---
`locale2`

```xml
<string name="">Content2</string>
```
---
`locale3`

```xml
<string name="">Content3</string>
```
---

### Headers

Keep our resource string tidy by specifying a header.

```
MainActivity

locale: Content
locale2: Content2
locale3: Content3
```

Output: 

`locale`

```xml
<!-- [MainActivity] -->
<string name="">Content</string>
```

`locale2`

```xml
<!-- [MainActivity] -->
<string name="">Content2</string>
```


`locale3`

```xml
<!-- [MainActivity] -->
<string name="">Content3</string>
```


### Resource Name

If you need to specify `name` for each item, you can use `#hashtag` 



### Headers

Keep our resource string tidy by specifying a header.

```
MainActivity #title_content_main

locale: Content
locale2: Content2
locale3: Content3
```

Output: 

`locale`

```xml
<!-- [MainActivity] -->
<string name="title_content_main">Content</string>
```

`locale2`

```xml
<!-- [MainActivity] -->
<string name="title_content_main">Content2</string>
```


`locale3`

```xml
<!-- [MainActivity] -->
<string name="title_content_main">Content3</string>
```

---

### Specify Output Files

In some cases, you might need to have different file name rather than `strings.xml`.

You can do that by using `@file` directive followed by the desired filename.

```
@file: my_file_name

en: English
km: ខ្មែរ
```

Result:

```
├── values
│   └── my_file_name.xml
└── values-km
    └── my_file_name.xml

2 directories, 2 files
```

