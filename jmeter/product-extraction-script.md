# JMeter Product Extraction Prerequisite Script

This document explains the product extraction prerequisite script embedded in `Workshop-spring-1.jmx`.

## Purpose

The setup thread group `Prerequisites - Extract Products` crawls the public OpenCart storefront UI and generates two files in the JMeter folder:

```text
products.json
products.csv
```

The generated product mapping has this shape:

```json
[
  { "productId": 40, "path": "20",    "categoryId": 20, "subcategoryId": null },
  { "productId": 41, "path": "20_27", "categoryId": 20, "subcategoryId": 27 }
]
```

The CSV format is:

```csv
productId,path,categoryId,subcategoryId
40,20,20,
41,20_27,20,27
```

## Activation

The script runs only when the Test Plan variable is set to:

```text
extract_products=true
```

By default it should normally stay:

```text
extract_products=false
```

This avoids crawling products on every test execution.

## Why BeanShell is used

The script uses BeanShell:

```xml
<stringProp name="scriptLanguage">bsh</stringProp>
```

Reason: JMeter 5.6.3 bundles Groovy 3.0.20, which may fail under Java 25 with:

```text
Unsupported class file major version 69
```

BeanShell avoids the Groovy ASM bytecode parsing issue.

## HTTP GET helper

The script uses:

```java
URL url = new URL(urlStr);
BufferedReader reader = new BufferedReader(new InputStreamReader(url.openStream(), "UTF-8"));
```

It intentionally avoids:

```java
HttpURLConnection
openConnection()
setRequestMethod()
setRequestProperty()
setReadTimeout()
```

Reason: on Java 25, BeanShell may reflect into the internal JDK class:

```text
sun.net.www.protocol.http.HttpURLConnection
```

and Java modules block access to it.

## Source 1 — Home page category menu

Request:

```text
GET /index.php?route=common/home&language=en-gb
```

Template:

```text
upload/catalog/view/template/common/menu.html
```

Root HTML component parsed:

```html
<nav id="menu" class="navbar navbar-expand-lg bg-primary">
  <div id="navbar-menu" class="collapse navbar-collapse">
    <ul class="nav navbar-nav">
      ...category and subcategory links...
    </ul>
  </div>
</nav>
```

OpenCart renders category links like:

```html
<a href="...&amp;path=20">Desktops</a>
<a href="...&amp;path=20_27">Mac</a>
```

Regex used:

```java
Pattern pathPat = Pattern.compile("path=([0-9_]+)");
```

Captured values:

| Captured `path` | Meaning |
|---|---|
| `20` | Top-level category |
| `20_27` | Subcategory `27` under category `20` |

The script stores paths in a `LinkedHashSet` to deduplicate while preserving menu order.

## Source 2 — Category page products

For every extracted category path, the script requests:

```text
GET /index.php?route=product/category&language=en-gb&path={path}
```

Template:

```text
upload/catalog/view/template/product/category.html
```

Root area:

```html
<div id="content">
  <h1>{heading}</h1>
  ...optional subcategory refinement links...
  <product-list sort="" order=""></product-list>
</div>
```

The category page can expose product IDs in two possible forms.

### Pattern A — product-list web component

Template:

```text
upload/catalog/view/template/product/product_list.html
```

Example HTML:

```html
<div id="product-list" class="row row-cols-1 row-cols-sm-2 row-cols-md-2 row-cols-lg-4">
  <div class="col mb-3"><product-thumb product="40"></product-thumb></div>
  <div class="col mb-3"><product-thumb product="41"></product-thumb></div>
</div>
```

### Pattern B — product links / hidden product field

Template:

```text
upload/catalog/view/template/product/thumb.html
```

Example HTML:

```html
<a href="...&amp;product_id=40"><img ...></a>
<input type="hidden" name="product_id" value="40"/>
```

The extraction script uses one combined regex:

```java
Pattern prodPat = Pattern.compile("(?:product-thumb product=\"|product_id=)([0-9]+)");
```

It catches both:

```text
product-thumb product="40"
product_id=40
```

The script stores product IDs in a `LinkedHashSet` because the same product ID can appear multiple times on one page.

## Mapping category and subcategory

For each path:

```java
String[] parts = path.split("_");
int categoryId = Integer.parseInt(parts[0]);
String subcatStr = parts.length > 1 ? parts[parts.length - 1] : "";
Integer subcategoryId = subcatStr.isEmpty() ? null : Integer.parseInt(subcatStr);
```

Examples:

| path | categoryId | subcategoryId |
|---|---:|---:|
| `20` | `20` | `null` |
| `20_27` | `20` | `27` |

## Deduplication

The script deduplicates by this key:

```java
String key = pid + "_" + path;
```

That means:

- same product repeated multiple times on the same category page is stored once
- same product in different categories can appear multiple times with different `path` values

Example allowed output:

```json
[
  { "productId": 40, "path": "20", "categoryId": 20, "subcategoryId": null },
  { "productId": 40, "path": "20_27", "categoryId": 20, "subcategoryId": 27 }
]
```

## Output files

### `products.json`

Human-readable product mapping:

```json
[
  {
    "productId": 40,
    "path": "20",
    "categoryId": 20,
    "subcategoryId": null
  }
]
```

### `products.csv`

Best format for JMeter `CSV Data Set Config`:

```csv
productId,path,categoryId,subcategoryId
40,20,20,
41,20_27,20,27
```

The existing CSV config is prepared with:

```text
filename=products.csv
ignoreFirstLine=true
variableNames=productId,path,categoryId,subcategoryId
```

## JMeter properties exposed

The script also sets:

```java
props.put("products_json", jsonText);
props.put("product_count", String.valueOf(results.size()));
vars.put("product_count", String.valueOf(results.size()));
```

| Name | Scope | Purpose |
|---|---|---|
| `products_json` | JMeter property | Full JSON content |
| `product_count` | JMeter property | Total product/category associations |
| `product_count` | JMeter variable | Same count in current setup thread |

## How to use the CSV later

Enable:

```text
CSV Data Set Config - Products
```

Then in HTTP samplers use:

### Browse category

```text
/index.php?route=product/category&path=${path}
```

### Open product

```text
/index.php?route=product/product&product_id=${productId}
```

The recommended navigation values are:

| Variable | Use |
|---|---|
| `${path}` | Navigate to the exact category/subcategory page |
| `${productId}` | Open product details |
| `${categoryId}` | Parent category reference |
| `${subcategoryId}` | Optional subcategory reference |

## Limitations

This is UI-based extraction, not database/API extraction. Therefore:

- only storefront-visible categories are extracted
- hidden/disabled products may not appear
- products not linked from menu-visible categories may not appear
- theme/template changes may require regex updates
- pagination may need additional handling if categories contain more products than the default page size

## Recommended data model

Use this row model throughout JMeter:

```csv
productId,path,categoryId,subcategoryId
```

Reason: `path` is OpenCart's native category navigation key, so it is more reliable than manually reconstructing category URLs from IDs.

