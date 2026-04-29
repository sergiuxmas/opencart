# JMeter Plan — Product Navigation → Add to Cart → Checkout → Confirm Order

This document describes a planned JMeter scenario for reproducing user product purchase flows in OpenCart using `products.csv`.

Implementation is intentionally left for later. This is a component-by-component plan.

---

## 1. Scenario goal

Build a JMeter user flow that:

1. Reads a product row from `products.csv`.
2. Navigates to the product category.
3. Navigates to the subcategory if `subcategoryId` exists.
4. Opens/selects the product.
5. Completes quantity.
6. Adds product to cart.
7. Validates that the item was added.
8. Collects product details from the product page/cart.
9. Opens cart dropdown/popup.
10. Validates cart popup details against collected product details.
11. Opens full cart page.
12. Validates cart/order details.
13. Opens checkout.
14. Completes required personal and shipping/payment details.
15. Selects shipping and payment methods.
16. Confirms the order.
17. Validates checkout success.

---

## 2. Input file

File:

```text
jmeter/products.csv
```

Current format:

```csv
productId,path,categoryId,subcategoryId
42,20,20,
41,20_27,20,27
```

Column meaning:

| Column | Meaning | Example |
|---|---|---|
| `productId` | OpenCart product ID | `41` |
| `path` | Full OpenCart category path | `20_27` |
| `categoryId` | Top-level category ID | `20` |
| `subcategoryId` | Optional subcategory ID | `27` or empty |

Important:

- `path=20` means top-level category.
- `path=20_27` means subcategory `27` under parent category `20`.
- For actual category navigation, `${path}` is the most important value.

---

## 3. Random product row strategy

### Recommended JMeter-native strategy

Use:

```text
CSV Data Set Config - Products
```

This exposes each row as variables:

```text
${productId}
${path}
${categoryId}
${subcategoryId}
```

However, standard JMeter `CSV Data Set Config` reads rows sequentially, not randomly.

So for random-like behavior, use one of these approaches.

### Option A — preferred simple approach: pre-shuffle CSV before test

Before running JMeter, shuffle `products.csv` into a random order.

Then `CSV Data Set Config` reads sequentially, but the sequence is already randomized.

JMeter components:

| Component | Purpose |
|---|---|
| `CSV Data Set Config - Products` | Reads one product row per thread/iteration |
| `Thread Group` loop count | Controls how many product rows are consumed |

Pros:

- Simple.
- JMeter-native.
- No scripting inside the test flow.
- Works well for user flow reproduction.

Cons:

- Randomization happens before the test, not during each sampler.
- If multiple threads share the file, understand the selected `Sharing mode`.

### Option B — true random during runtime

If true random selection is required each iteration, use a `JSR223 PreProcessor` or `JSR223 Sampler` to read `products.csv` and pick a random row.

JMeter components:

| Component | Purpose |
|---|---|
| `JSR223 Sampler` or `JSR223 PreProcessor` | Load CSV and pick a random row |
| `vars.put(...)` | Set `productId`, `path`, `categoryId`, `subcategoryId` |

Pros:

- True random row per iteration.

Cons:

- Requires scripting.
- More maintenance.

### Recommendation for this project

Use **Option A** first:

```text
products.csv + CSV Data Set Config
```

If later you need true random per request, add scripting only for row selection.

---

## 4. Required JMeter components

### Test Plan level

Use existing or add:

| Component | Purpose |
|---|---|
| `HTTP Request Defaults` | Host `opencart.local`, port `8080`, protocol `http` |
| `HTTP Cookie Manager` | Preserve cart/session/checkout state |
| `HTTP Header Manager` | Browser-like headers, JSON/AJAX headers where needed |
| `CSV Data Set Config - Products` | Load product rows from `products.csv` |
| `User Defined Variables` | Checkout test data and defaults |
| `View Results Tree` | Debugging only |
| `View Results in Table` | Debugging/performance overview |

### Thread Group level

Create a new thread group, for example:

```text
User Thread Group - Product Purchase Flow
```

Inside it, add:

```text
Transaction Controller - Purchase Product
```

Recommended components inside the transaction:

| Component | Purpose |
|---|---|
| `HTTP Request` | Navigate pages / submit forms |
| `If Controller` | Only open subcategory when `${subcategoryId}` is not empty |
| `Response Assertion` | Validate HTML or JSON response content |
| `JSON Assertion` | Validate JSON from AJAX endpoints |
| `JSON Extractor` | Extract values from cart/checkout JSON |
| `Regular Expression Extractor` | Extract product name/model/price from HTML when CSS extractor is not enough |
| `CSS Selector Extractor` | Preferred for product page/cart page HTML extraction if available |
| `JSR223 Assertion` | Optional complex validation, e.g. compare totals/quantities |
| `Constant Timer` / `Uniform Random Timer` | Think time between user actions |
| `Debug Sampler` | Temporary troubleshooting only |

---

## 5. CSV Data Set Config setup

Use existing config:

```text
CSV Data Set Config - Products
```

Recommended values:

| Field | Value |
|---|---|
| Filename | `products.csv` |
| File encoding | `UTF-8` |
| Variable Names | `productId,path,categoryId,subcategoryId` |
| Ignore first line | `true` |
| Delimiter | `,` |
| Recycle on EOF | `true` for long-running tests, `false` for one pass |
| Stop thread on EOF | `false` if recycling, `true` if one pass |
| Sharing mode | `All threads` or `Current thread group`, depending on load model |

Variables available after CSV load:

```text
${productId}
${path}
${categoryId}
${subcategoryId}
```

---

## 6. Suggested user variables

Add these to `User Defined Variables` or a dedicated config element:

```text
language=en-gb
purchase_quantity=1
checkout_firstname=Performance
checkout_lastname=User
checkout_email_prefix=perfuser
checkout_telephone=0712345678
checkout_address_1=1 Test Street
checkout_city=London
checkout_postcode=SW1A1AA
checkout_country_id=222
checkout_zone_id=3563
checkout_customer_group_id=1
checkout_address_match=1
preferred_shipping_method=flat.flat
preferred_payment_method=cod.cod
```

Notes:

- `country_id` and `zone_id` depend on the test database configuration.
- Use valid country/zone values from your OpenCart install.
- For unique guest email, use a function, for example:

```text
${checkout_email_prefix}_${__time()}_${__threadNum}@testmail.com
```

---

## 7. Detailed flow plan

---

### Step 1 — Open top-level category

Purpose:

User lands on the top-level category page.

HTTP Request:

```text
GET /index.php
```

Parameters:

```text
route=product/category
language=${language}
path=${categoryId}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Open Category` | Load category page |
| `Response Assertion` | Validate HTTP 200 / category page loaded |
| `Response Assertion` | Check response does not contain `Page Not Found` |

Suggested assertions:

```text
NOT contains: Page Not Found
Contains: product/category
```

or, if page title/category heading is known, assert the heading text.

---

### Step 2 — Open subcategory if it exists

Only run this if:

```text
${subcategoryId} is not empty
```

JMeter component:

```text
If Controller - Has Subcategory
```

Condition example:

```javascript
"${subcategoryId}" != ""
```

HTTP Request:

```text
GET /index.php
```

Parameters:

```text
route=product/category
language=${language}
path=${path}
```

JMeter components:

| Component | Purpose |
|---|---|
| `If Controller` | Skip this step for top-level-only products |
| `HTTP Request - Open Subcategory` | Load subcategory page |
| `Response Assertion` | Validate product listing/subcategory loaded |

Important:

- For rows with `path=20`, `${path}` equals `${categoryId}`.
- For rows with `path=20_27`, `${path}` opens the real subcategory page.

---

### Step 3 — Open/select product

HTTP Request:

```text
GET /index.php
```

Parameters:

```text
route=product/product
language=${language}
path=${path}
product_id=${productId}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Open Product` | Open selected product detail page |
| `Response Assertion` | Validate product page loaded |
| `CSS Selector Extractor` | Extract product name, price, quantity, hidden ID |
| `Regular Expression Extractor` | Fallback for product model/price fields |
| `Response Assertion` | Validate page contains `name="product_id" value="${productId}"` |

Product page HTML references:

Template:

```text
upload/catalog/view/template/product/product.html
```

Important fields:

```html
<h1>{{ heading_title }}</h1>
<li>Model: <strong>{{ model }}</strong></li>
<span class="price-new"><x-currency code="..." amount="..."></x-currency></span>
<input type="text" name="quantity" value="{{ minimum }}" id="input-quantity" />
<input type="hidden" name="product_id" value="{{ product_id }}" id="input-product-id" />
<button id="button-cart">Add to Cart</button>
```

Suggested extractors:

| Variable | Extractor | Suggested expression |
|---|---|---|
| `productName` | CSS Selector Extractor | `h1` text |
| `productModel` | Regex Extractor | `Model:\s*<strong>([^<]+)</strong>` |
| `unitPriceAmount` | Regex Extractor | `class="price-new".*?amount="([^"]+)"` |
| `minimumQuantity` | Regex Extractor | `name="quantity" value="([0-9]+)"` |
| `pageProductId` | Regex Extractor | `id="input-product-id"[^>]*value="([0-9]+)"` |

Assertions:

```text
${pageProductId} == ${productId}
productName is not empty
minimumQuantity is not empty
```

Optional limitation:

If product page contains required options:

```html
<div class="mb-3 required">
```

then the product cannot be added without also selecting options. Initial implementation can skip products with required options or assert that no required options exist.

---

### Step 4 — Set quantity

Use either:

```text
quantity=${purchase_quantity}
```

or respect product minimum:

```text
quantity=${minimumQuantity}
```

Recommended first implementation:

```text
quantity=${minimumQuantity}
```

JMeter components:

| Component | Purpose |
|---|---|
| `JSR223 PreProcessor` or `User Defined Variable` | Set `cartQuantity` |

Simple plan:

```text
cartQuantity=${minimumQuantity}
```

If you want random quantity later:

```text
cartQuantity = random value between minimumQuantity and a small max, e.g. 3
```

---

### Step 5 — Add product to cart

OpenCart endpoint:

```text
POST /index.php?route=checkout/cart.add&language=${language}
```

Form parameters:

```text
product_id=${productId}
quantity=${cartQuantity}
```

If options are required, also send:

```text
option[product_option_id]=product_option_value_id
subscription_plan_id=...
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Add to Cart` | POST product to cart |
| `Header Manager` | `X-Requested-With: XMLHttpRequest`, `Accept: application/json` |
| `JSON Assertion` | Response has success and no error |
| `Response Assertion` | Contains `success` |
| `Response Assertion` | NOT contains `error` |
| `JSON Extractor` | Extract `success` text if needed |

Expected JSON success shape:

```json
{
  "success": "Success: You have added <a href=...>Product Name</a> to your <a href=...>shopping cart</a>!"
}
```

Validation:

```text
Contains: "success"
NOT Contains: "error"
Contains: ${productName}
```

---

### Step 6 — Check item is added using cart JSON

OpenCart endpoint:

```text
GET /index.php?route=checkout/cart.json&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Cart JSON` | Fetch structured cart data |
| `JSON Extractor` | Extract cart product details |
| `JSR223 Assertion` | Verify selected product exists |

Cart JSON includes:

```json
{
  "products": [
    {
      "product_id": 40,
      "name": "MacBook",
      "model": "Product 16",
      "quantity": 1,
      "price": "...",
      "total": "...",
      "href": "...product_id=40"
    }
  ],
  "totals": []
}
```

Suggested validation:

```text
At least one products[*].product_id == ${productId}
Matching product quantity == ${cartQuantity}
Matching product name contains ${productName}
```

Useful variables to extract:

| Variable | Source |
|---|---|
| `cartProductId` | JSON products array |
| `cartProductName` | JSON products array |
| `cartProductModel` | JSON products array |
| `cartProductQuantity` | JSON products array |
| `cartProductPrice` | JSON products array |
| `cartProductTotal` | JSON products array |

---

### Step 7 — Click cart items / open cart popup

OpenCart route:

```text
GET /index.php?route=common/cart.info&language=${language}
```

This returns the cart dropdown/popup HTML.

Template:

```text
upload/catalog/view/template/common/cart.html
```

Important HTML structure:

```html
<ul class="dropdown-menu dropdown-menu-end p-2">
  <table class="table table-striped mb-2">
    <tr>
      <td><a href="...product_id=40">Product Name</a></td>
      <td class="text-end text-nowrap">x 1</td>
      <td class="text-end">...</td>
    </tr>
  </table>
  <a href="...route=checkout/cart">View Cart</a>
  <a href="...route=checkout/checkout">Checkout</a>
</ul>
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Cart Popup` | Load cart info popup HTML |
| `Response Assertion` | Product name exists |
| `Response Assertion` | Quantity text `x ${cartQuantity}` exists |
| `Response Assertion` | View Cart link exists |
| `Response Assertion` | Checkout link exists |

Suggested assertions:

```text
Contains: ${productName}
Contains: x ${cartQuantity}
Contains: route=checkout/cart
Contains: route=checkout/checkout
```

---

### Step 8 — Click View Cart

OpenCart route:

```text
GET /index.php?route=checkout/cart&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - View Cart Page` | Open full shopping cart page |
| `Response Assertion` | Product name exists |
| `Response Assertion` | Product model exists |
| `Response Assertion` | Quantity exists |
| `CSS Selector Extractor` | Extract displayed totals if needed |

Suggested validations:

```text
Contains: ${productName}
Contains: ${productModel}
Contains: value="${cartQuantity}"
Contains: Checkout
```

If model extraction is unreliable, validate only product name and product ID/href.

---

### Step 9 — Click Checkout

OpenCart route:

```text
GET /index.php?route=checkout/checkout&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Open Checkout` | Load checkout page |
| `Response Assertion` | Checkout page loaded |

Suggested assertions:

```text
Contains: Checkout
NOT Contains: Your shopping cart is empty
```

---

## 8. Checkout flow plan

OpenCart checkout is AJAX-heavy. The exact available shipping/payment methods depend on configuration.

Recommended initial approach:

- Use guest checkout if enabled.
- Use `account=0` in `checkout/register.save`.
- Use `address_match=1` to copy payment address to shipping address.
- Select first available shipping method from quote response.
- Select preferred payment method, typically `cod.cod`, if installed/enabled.

---

### Step 10 — Complete Personal Details and Address

Endpoint:

```text
POST /index.php?route=checkout/register.save&language=${language}
```

Important form parameters:

```text
account=0
customer_group_id=${checkout_customer_group_id}
firstname=${checkout_firstname}
lastname=${checkout_lastname}
email=${checkoutEmail}
telephone=${checkout_telephone}
payment_address_1=${checkout_address_1}
payment_city=${checkout_city}
payment_postcode=${checkout_postcode}
payment_country_id=${checkout_country_id}
payment_zone_id=${checkout_zone_id}
address_match=${checkout_address_match}
```

If `address_match=0`, also send:

```text
shipping_firstname=${checkout_firstname}
shipping_lastname=${checkout_lastname}
shipping_address_1=${checkout_address_1}
shipping_city=${checkout_city}
shipping_postcode=${checkout_postcode}
shipping_country_id=${checkout_country_id}
shipping_zone_id=${checkout_zone_id}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Checkout Register Save` | Submit guest/personal/address data |
| `Header Manager` | JSON/AJAX headers |
| `Response Assertion` | Contains `success` |
| `Response Assertion` | NOT contains `error` |
| `JSON Assertion` | No `error`, no `redirect` unless expected |

Expected success examples from controller:

```json
{
  "success": "Success: Guest account details have been saved!"
}
```

Notes:

- If guest checkout is disabled, the endpoint forces `account=1` and password/agreement may be required.
- If captcha is enabled for register checkout, this flow may fail unless captcha is disabled for performance testing.

---

### Step 11 — Get shipping quotes

Endpoint:

```text
GET /index.php?route=checkout/shipping_method.quote&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Get Shipping Quotes` | Retrieve available shipping methods |
| `JSON Extractor` or `JSR223 PostProcessor` | Extract shipping method code |
| `Response Assertion` | Contains `shipping_methods` |

Response shape:

```json
{
  "shipping_methods": {
    "flat": {
      "quote": {
        "flat": {
          "code": "flat.flat",
          "title": "Flat Shipping Rate"
        }
      }
    }
  }
}
```

Recommended extraction:

```text
shippingMethodCode=flat.flat
```

or dynamically extract the first available `code`.

---

### Step 12 — Save shipping method

Endpoint:

```text
POST /index.php?route=checkout/shipping_method.save&language=${language}
```

Form parameter:

```text
shipping_method=${shippingMethodCode}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Save Shipping Method` | Save selected shipping method |
| `Response Assertion` | Contains `success` |
| `Response Assertion` | NOT contains `error` |

---

### Step 13 — Get payment methods

Endpoint:

```text
GET /index.php?route=checkout/payment_method.getMethods&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Get Payment Methods` | Retrieve available payment methods |
| `JSON Extractor` or `JSR223 PostProcessor` | Extract payment method code |
| `Response Assertion` | Contains `payment_methods` |

Response structure validates method code as:

```text
payment_methods[method_group].option[method_code]
```

Examples:

```text
cod.cod
bank_transfer.bank_transfer
cheque.cheque
free_checkout.free_checkout
```

Recommended first implementation:

```text
paymentMethodCode=cod.cod
```

If COD is not enabled, dynamically select first available method.

---

### Step 14 — Save payment method

Endpoint:

```text
POST /index.php?route=checkout/payment_method.save&language=${language}
```

Form parameter:

```text
payment_method=${paymentMethodCode}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Save Payment Method` | Save selected payment method |
| `Response Assertion` | Contains `success` |
| `Response Assertion` | NOT contains `error` |

Optional if terms are required:

Endpoint:

```text
POST /index.php?route=checkout/payment_method.agree&language=${language}
```

Form parameter:

```text
agree=1
```

---

### Step 15 — Confirm order summary

Endpoint:

```text
GET /index.php?route=checkout/confirm.confirm&language=${language}
```

This renders the order confirmation HTML.

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Checkout Confirm Summary` | Load order confirmation HTML |
| `Response Assertion` | Product name exists |
| `Response Assertion` | Quantity exists |
| `Response Assertion` | Total/order table exists |
| `Response Assertion` | Payment confirm button or payment block exists |

Suggested assertions:

```text
Contains: ${productName}
Contains: ${cartQuantity}
Contains: ${productModel}
NOT Contains: error
```

---

### Step 16 — Confirm payment/order

For COD payment:

```text
POST or GET /index.php?route=extension/opencart/payment/cod.confirm&language=${language}
```

Controller response:

```json
{
  "redirect": "...route=checkout/success..."
}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Payment Confirm` | Confirm order through selected payment extension |
| `JSON Assertion` | `redirect` exists |
| `Response Assertion` | Contains `checkout/success` |
| `Regex Extractor` or `JSON Extractor` | Extract success redirect URL if needed |

If using another payment method, route changes:

| Payment | Confirm route |
|---|---|
| COD | `extension/opencart/payment/cod.confirm` |
| Bank Transfer | `extension/opencart/payment/bank_transfer.confirm` |
| Cheque | `extension/opencart/payment/cheque.confirm` |
| Free Checkout | `extension/opencart/payment/free_checkout.confirm` |

---

### Step 17 — Open success page

Endpoint:

```text
GET /index.php?route=checkout/success&language=${language}
```

JMeter components:

| Component | Purpose |
|---|---|
| `HTTP Request - Checkout Success` | Load final success page |
| `Response Assertion` | Order success text exists |
| `Response Assertion` | Cart empty or success state confirmed |

Suggested assertion text:

```text
Your order has been placed!
```

or equivalent language-specific success text.

---

## 9. Product details to collect

Collect these after opening product page and/or cart JSON:

| Variable | Source | Purpose |
|---|---|---|
| `productName` | Product page `h1` | Validate cart popup/cart/confirm |
| `productModel` | Product page model row | Validate cart/cart popup/order |
| `pageProductId` | Hidden product input | Verify page matches selected CSV row |
| `minimumQuantity` | Quantity input default value | Use valid quantity |
| `cartQuantity` | Derived variable | Add-to-cart and assertions |
| `unitPriceAmount` | Product page price | Optional price validation |
| `cartProductTotal` | Cart JSON | Optional total validation |

Recommended minimum set for first implementation:

```text
productName
productModel
minimumQuantity
cartQuantity
```

---

## 10. Assertions summary

### Category/subcategory

```text
HTTP 200
NOT contains Page Not Found
```

### Product page

```text
Contains product_id hidden input
Extracted pageProductId == ${productId}
Extracted productName not empty
```

### Add to cart

```text
Contains "success"
NOT contains "error"
Contains ${productName}
```

### Cart popup

```text
Contains ${productName}
Contains x ${cartQuantity}
Contains route=checkout/cart
Contains route=checkout/checkout
```

### Cart page

```text
Contains ${productName}
Contains ${productModel}
Contains Checkout
```

### Checkout register save

```text
Contains success
NOT contains error
NOT contains redirect, unless expected
```

### Shipping/payment methods

```text
Shipping quote response contains shipping_methods
Shipping save response contains success
Payment methods response contains payment_methods
Payment save response contains success
```

### Confirm order

```text
Confirm page contains ${productName}
Confirm page contains ${cartQuantity}
Payment confirm response contains redirect
Redirect contains checkout/success
```

### Success page

```text
Contains order success text
```

---

## 11. Recommended JMeter component tree

Use this hierarchy principle:

```text
Thread Group = scenario boundary
Transaction Controller = measurable business transaction / phase
Simple Controller = readability-only grouping inside a phase
If Controller = conditional branch
HTTP Request = one browser/AJAX request
Extractors / Assertions = attached directly under the sampler they parse/validate
```

Recommended Transaction Controller settings:

| Setting | Recommended value | Reason |
|---|---|---|
| `Generate parent sample` | `true` | Produces one summarized row for the business transaction in listeners/reports |
| `Include duration of timer and pre/post processors` | `true` for user-flow realism | Includes think time and extraction/validation overhead in end-to-end flow timing |

Keep `CSV Data Set Config - Products` at Thread Group level, not inside the Transaction Controller, so the row variables are available before the flow starts.

### 11.1 Full controller hierarchy scheme

```text
Thread Group: User Thread Group - Product Purchase Flow
├── CSV Data Set Config: CSV Data Set Config - Products
│   └── Provides: ${productId}, ${path}, ${categoryId}, ${subcategoryId}
├── HTTP Cookie Manager
├── HTTP Header Manager: Browser defaults
├── User Defined Variables: checkout defaults / language / quantity defaults
├── Uniform Random Timer: User think time, optional
└── Transaction Controller: TX - Product Purchase Flow - End to End
    ├── Transaction Controller: TX - Browse and Select Product
    │   ├── Simple Controller: Step - Open Category
    │   │   └── HTTP Request: GET Category Page
    │   │       ├── Response Assertion: HTTP/page is not 404
    │   │       └── Response Assertion: Category page loaded
    │   ├── If Controller: Has Subcategory
    │   │   └── Simple Controller: Step - Open Subcategory
    │   │       └── HTTP Request: GET Subcategory Page
    │   │           ├── Response Assertion: HTTP/page is not 404
    │   │           └── Response Assertion: Subcategory/product listing loaded
    │   └── Simple Controller: Step - Open Product
    │       └── HTTP Request: GET Product Page
    │           ├── CSS Selector Extractor: productName
    │           ├── Regex Extractor: productModel
    │           ├── Regex Extractor: unitPriceAmount, optional
    │           ├── Regex Extractor: minimumQuantity
    │           ├── Regex Extractor: pageProductId
    │           ├── Response Assertion: Product page loaded
    │           └── JSR223/Response Assertion: pageProductId == productId
    ├── Simple Controller: Step - Prepare Cart Input
    │   ├── JSR223 PreProcessor or User Parameters: Set cartQuantity
    │   └── Debug Sampler: Product variables, disabled by default
    ├── Transaction Controller: TX - Add Product To Cart
    │   └── HTTP Request: POST Add to Cart
    │       ├── Header Manager: AJAX JSON headers
    │       ├── JSON Extractor: addCartSuccess, optional
    │       ├── Response Assertion: Contains success
    │       └── Response Assertion: Not contains error
    ├── Transaction Controller: TX - Validate Cart
    │   ├── HTTP Request: GET Cart JSON
    │   │   ├── JSON Extractor: cart product fields
    │   │   ├── JSON Assertion: products array exists
    │   │   └── JSR223 Assertion: selected product exists with expected quantity
    │   ├── HTTP Request: GET Cart Popup
    │   │   ├── Response Assertion: Product name present
    │   │   ├── Response Assertion: Quantity present
    │   │   ├── Response Assertion: View Cart link present
    │   │   └── Response Assertion: Checkout link present
    │   └── HTTP Request: GET View Cart Page
    │       ├── CSS Selector Extractor: cart totals, optional
    │       ├── Response Assertion: Product name/model present
    │       ├── Response Assertion: Quantity present
    │       └── Response Assertion: Checkout action present
    ├── Transaction Controller: TX - Checkout Details
    │   ├── HTTP Request: GET Checkout Page
    │   │   └── Response Assertion: Checkout loaded and cart not empty
    │   ├── HTTP Request: POST Checkout Register Save
    │   │   ├── Header Manager: AJAX JSON headers
    │   │   ├── JSON Assertion: No error
    │   │   └── Response Assertion: Contains success
    │   ├── HTTP Request: GET Shipping Quotes
    │   │   ├── JSON Extractor / JSR223 PostProcessor: shippingMethodCode
    │   │   └── Response Assertion: Contains shipping_methods
    │   ├── HTTP Request: POST Save Shipping Method
    │   │   └── Response Assertion: Contains success and not error
    │   ├── HTTP Request: GET Payment Methods
    │   │   ├── JSON Extractor / JSR223 PostProcessor: paymentMethodCode
    │   │   └── Response Assertion: Contains payment_methods
    │   ├── HTTP Request: POST Save Payment Method
    │   │   └── Response Assertion: Contains success and not error
    │   └── If Controller: Payment Terms Required
    │       └── HTTP Request: POST Payment Agree
    │           └── Response Assertion: Contains success and not error
    ├── Transaction Controller: TX - Confirm Order
    │   ├── HTTP Request: GET Checkout Confirm Summary
    │   │   ├── Response Assertion: Product name present
    │   │   ├── Response Assertion: Quantity present
    │   │   └── Response Assertion: Total/order table present
    │   ├── HTTP Request: POST/GET Payment Confirm
    │   │   ├── JSON Extractor: successRedirect, optional
    │   │   ├── JSON Assertion: redirect exists
    │   │   └── Response Assertion: Contains checkout/success
    │   └── HTTP Request: GET Checkout Success
    │       └── Response Assertion: Order success text present
    └── Simple Controller: Debug - Disabled By Default
        └── Debug Sampler: Final variables
```

### 11.2 Compact hierarchy for first implementation

For the first milestone, implement only up to cart validation:

```text
Thread Group: User Thread Group - Product Purchase Flow
├── CSV Data Set Config - Products
├── HTTP Cookie Manager
└── Transaction Controller: TX - Product Purchase Flow - Cart Only
    ├── Transaction Controller: TX - Browse and Select Product
    │   ├── HTTP Request: GET Category Page
    │   ├── If Controller: Has Subcategory
    │   │   └── HTTP Request: GET Subcategory Page
    │   └── HTTP Request: GET Product Page
    │       ├── Extractors: productName, productModel, minimumQuantity, pageProductId
    │       └── Assertions: product loaded and product ID matches CSV
    ├── Transaction Controller: TX - Add Product To Cart
    │   └── HTTP Request: POST Add to Cart
    │       └── Assertions: success and no error
    └── Transaction Controller: TX - Validate Cart
        ├── HTTP Request: GET Cart JSON
        ├── HTTP Request: GET Cart Popup
        └── HTTP Request: GET View Cart Page
```

Once this is stable, add the `TX - Checkout Details` and `TX - Confirm Order` phases.

### 11.3 Controller usage rules

| Controller/component | Use for | Do not use for |
|---|---|---|
| `Transaction Controller` | Measuring a complete business flow or major phase | Small visual grouping only |
| `Simple Controller` | Readability grouping without separate timing semantics | Validation or branching |
| `If Controller` | Conditional subcategory/payment-agree/optional branches | General grouping |
| `CSV Data Set Config` | Supplying one product row per thread/iteration | Random access to arbitrary rows |
| `Regex/CSS/JSON Extractor` | Parsing one sampler response | Parsing unrelated later responses |
| `Response/JSON/JSR223 Assertion` | Validating the sampler response or extracted values | Global validation detached from the step |
| `Debug Sampler` | Temporary troubleshooting | Permanent load test execution |

---

## 12. Initial implementation constraints

For the first implementation, keep the flow simple:

1. Use products without required options.
2. Use `quantity=${minimumQuantity}`.
3. Use guest checkout if enabled.
4. Use `address_match=1`.
5. Use one known shipping method, e.g. `flat.flat`, if enabled.
6. Use one known payment method, e.g. `cod.cod`, if enabled.
7. Use CSV row order or pre-shuffled CSV for random behavior.

After the basic flow works, add:

- dynamic shipping method selection,
- dynamic payment method selection,
- required product option support,
- random quantity,
- stronger price/total validations,
- negative validations for unavailable products.

---

## 13. Key risk areas

| Risk | Mitigation |
|---|---|
| Product requires options | Skip those products initially or extract/select required options |
| Guest checkout disabled | Use account registration/login checkout flow |
| Captcha enabled | Disable captcha for performance test environment |
| Shipping method not available | Extract first available method dynamically |
| Payment method not available | Extract first available method dynamically |
| Country/zone invalid | Use valid IDs from local OpenCart database/config |
| CSV is sequential, not random | Pre-shuffle CSV before test or add random row JSR223 logic |
| Product out of stock | Filter CSV to purchasable products or assert and skip |

---

## 14. Recommended first milestone

Implement only through cart validation first:

```text
CSV row → Category → Subcategory → Product → Add to Cart → Cart JSON → Cart Popup → View Cart
```

Once that is stable, add checkout:

```text
Checkout → Register Save → Shipping → Payment → Confirm → Success
```

