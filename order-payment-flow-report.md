# تقرير تدفق الطلب والدفع

## نظرة عامة

هذا الملف يوثق **البيانات المرسلة** عند وضع الطلب (دفع عند الاستلام vs دفع أونلاين)، و**طريقة إرجاع البيانات** من الباك إند، والفرق بين المسارات والـ APIs المستخدمة.

---

## 1. بناء بيانات الطلب (PlaceOrderBody)

في **كلا** نوعي الدفع يتم بناء نفس الـ **PlaceOrderBody** من نفس الكود في `confirm_button_view.dart`.

### 1.1 الحقول المشتركة

| الحقل | النوع | الوصف |
|-------|--------|--------|
| `cart` | مصفوفة | عناصر السلة |
| `coupon_discount_amount` | number | قيمة خصم الكوبون |
| `coupon_discount_title` | string | عنوان خصم الكوبون |
| `order_amount` | number | مجموع السلة (بدون التوصيل) |
| `order_type` | string | `"delivery"` أو `"take_away"` |
| `delivery_address_id` | number | معرف عنوان التوصيل |
| `payment_method` | string | طريقة الدفع |
| `order_note` | string | ملاحظة الطلب |
| `coupon_code` | string | كود الكوبون |
| `delivery_time` | string | وقت التوصيل (مثل `"11:01"`) |
| `delivery_date` | string | تاريخ التوصيل (مثل `"2026-02-03"`) |
| `branch_id` | number | معرف الفرع |
| `distance` | number | المسافة (كم) |
| `is_partial` | string | `"0"` أو `"1"` |
| `payment_info` | object | اختياري، للدفع اليدوي فقط |

### 1.2 هيكل عنصر السلة (Cart)

كل عنصر في `cart` له الشكل التالي:

```json
{
  "product_id": "12",
  "price": "23.5",
  "variant": [],
  "variations": [
    {
      "name": "نوع الرز",
      "values": [
        {
          "label": "رز شعبي",
          "optionPrice": "23.5",
          "optionCalories": "1563"
        }
      ]
    }
  ],
  "discount_amount": 0,
  "quantity": 3,
  "tax_amount": 0,
  "add_on_ids": [],
  "add_on_qtys": []
}
```

| الحقل | الوصف |
|-------|--------|
| `product_id` | معرف المنتج (نص) |
| `price` | سعر الوحدة كما يحسبه التطبيق (نص) |
| `variant` | مصفوفة فارغة أو قيم الـ variant |
| `variations` | مصفوفة تنوعات؛ كل عنصر: `name` + `values` (مصفوفة كائنات: `label`, `optionPrice`, `optionCalories`) |
| `discount_amount` | خصم على المنتج |
| `quantity` | الكمية |
| `tax_amount` | يُرسل 0 من التطبيق |
| `add_on_ids` | مصفوفة معرفات الإضافات |
| `add_on_qtys` | مصفوفة كميات الإضافات |

---

## 2. الدفع عند الاستلام (Cash on Delivery)

### 2.1 التدفق

1. المستخدم يختار **الدفع عند الاستلام** ويضغط "تأكيد الطلب".
2. يُبنى **PlaceOrderBody** مع `payment_method: "cash_on_delivery"` وبدون `transaction_reference`.
3. يُستدعى مباشرة: `orderProvider.placeOrder(placeOrderBody, callBack)`.
4. طلب **واحد** يذهب إلى الباك إند: `POST /api/v1/customer/order/place`.

### 2.2 الطلب المرسل (مثال كامل)

```json
{
  "cart": [
    {
      "product_id": "12",
      "price": "23.5",
      "variant": [],
      "variations": [
        {
          "name": "نوع الرز",
          "values": [
            {
              "label": "رز شعبي",
              "optionPrice": "23.5",
              "optionCalories": "1563"
            }
          ]
        }
      ],
      "discount_amount": 0,
      "quantity": 3,
      "tax_amount": 0,
      "add_on_ids": [],
      "add_on_qtys": []
    }
  ],
  "coupon_discount_amount": 0,
  "coupon_discount_title": "",
  "order_amount": 70.5,
  "order_type": "delivery",
  "delivery_address_id": 109,
  "payment_method": "cash_on_delivery",
  "order_note": "",
  "coupon_code": "",
  "delivery_time": "11:01",
  "delivery_date": "2026-02-03",
  "branch_id": 1,
  "distance": 4.994,
  "is_partial": "0"
}
```

**ملاحظة:** لا يُرسل حقل `transaction_reference` في الدفع عند الاستلام.

---

## 3. الدفع أونلاين (مثل ClickPay)

### 3.1 التدفق

1. المستخدم يختار **الدفع أونلاين** ويضغط "تأكيد الطلب".
2. يُبنى **نفس** PlaceOrderBody مع `payment_method: "clickpay"` (أو اسم البوابة).
3. **مرحلة بدء الدفع:**
   - استدعاء: `POST /api/v1/payment-mobile/initiate` مع مبلغ الدفع وبيانات العميل.
   - الـ API يرجع **paymentUrl**.
4. **حفظ الطلب محلياً:**  
   `placeOrderBody.toJson()` → JSON → **base64** → يُحفظ عبر `setPlaceOrder(...)`.
5. المستخدم يُوجّه إلى صفحة الدفع (paymentUrl).
6. بعد نجاح الدفع، البوابة تعيد التطبيق (redirect أو deep link) مع `payment_id`.
7. التطبيق يقرأ الـ place order المحفوظ، يعدّله بـ `copyWith(paymentMethod: 'clickpay', transactionReference: paymentId)`.
8. استدعاء: `orderProvider.placeOrder(_placeOrderBody, _callback)` → **نفس** `POST /api/v1/customer/order/place` مع body محدّث.

### 3.2 الطلب الأول: بدء الدفع (Initiate)

**المسار:** `POST /api/v1/payment-mobile/initiate`

**Body (مثال):**

```json
{
  "payment_method": "clickpay",
  "payment_platform": "android",
  "order_amount": 90.48,
  "customer_id": "486",
  "is_guest": 0,
  "order_id": "ORD-1770072042225",
  "app_scheme": "makifood://payment/return"
}
```

| الحقل | الوصف |
|-------|--------|
| `payment_method` | اسم البوابة (مثل `clickpay`) |
| `payment_platform` | `"android"` أو `"ios"` أو `"web"` |
| `order_amount` | **المبلغ المطلوب من العميل** = orderAmount + deliveryCharge |
| `customer_id` | معرف العميل أو الضيف |
| `is_guest` | 0 أو 1 |
| `order_id` | معرف طلب مؤقت (مثل `ORD-...`) |
| `app_scheme` | scheme للعودة للتطبيق |

### 3.3 الطلب الثاني: وضع الطلب (بعد نجاح الدفع)

**المسار:** `POST /api/v1/customer/order/place`

**Body (مثال كامل):**

```json
{
  "cart": [
    {
      "product_id": "12",
      "price": "23.5",
      "variant": [],
      "variations": [
        {
          "name": "نوع الرز",
          "values": [
            {
              "label": "رز شعبي",
              "optionPrice": "23.5",
              "optionCalories": "1563"
            }
          ]
        }
      ],
      "discount_amount": 0,
      "quantity": 3,
      "tax_amount": 0,
      "add_on_ids": [],
      "add_on_qtys": []
    }
  ],
  "coupon_discount_amount": 0,
  "coupon_discount_title": "",
  "order_amount": 70.5,
  "order_type": "delivery",
  "delivery_address_id": 109,
  "payment_method": "clickpay",
  "transaction_reference": "a7c547b9-caa2-4802-aee6-61bf0a4761a0",
  "order_note": "",
  "coupon_code": "",
  "delivery_time": "11:01",
  "delivery_date": "2026-02-03",
  "branch_id": 1,
  "distance": 4.994,
  "is_partial": "0"
}
```

**الفرق عن الدفع عند الاستلام:**

- `payment_method`: `"clickpay"` بدل `"cash_on_delivery"`.
- وجود حقل **`transaction_reference`**: قيمة `payment_id` المرسلة من البوابة بعد نجاح الدفع.

باقي الحقول (cart, order_amount, delivery_time, ...) **نفسها** لأنها من الـ body المحفوظ قبل الدفع.

---

## 4. إرجاع البيانات من الباك إند

يوجد مساران لاسترجاع بيانات الطلب.

### 4.1 تتبع الطلب (Track)

| البند | القيمة |
|-------|--------|
| **المسار** | `GET /api/v1/customer/order/track?order_id={id}` |
| **نوع الاستجابة** | كائن **واحد** (الطلب + تفاصيله) |
| **الاستخدام في التطبيق** | `OrderModel.fromJson(response.data)` |

**مثال على شكل الاستجابة:**

```json
{
  "id": 100294,
  "user_id": 486,
  "is_guest": "0",
  "order_amount": 70.5,
  "coupon_discount_amount": 0,
  "coupon_discount_title": null,
  "payment_status": "paid",
  "order_status": "confirmed",
  "total_tax_amount": 11.16,
  "payment_method": "clickpay",
  "transaction_reference": "a7c547b9-caa2-4802-aee6-61bf0a4761a0",
  "delivery_address_id": 109,
  "created_at": "2026-02-02T23:21:27.000000Z",
  "updated_at": "2026-02-02T23:21:27.000000Z",
  "checked": "0",
  "delivery_man_id": null,
  "delivery_charge": 19.98,
  "order_note": null,
  "coupon_code": null,
  "order_type": "delivery",
  "branch_id": "11",
  "callback": null,
  "delivery_date": "2026-02-03",
  "delivery_time": "11:31:00",
  "extra_discount": "0.00",
  "delivery_address": {
    "id": 109,
    "address_type": "Home",
    "contact_person_number": "+967774627574",
    "address": "PM7G+C4F, Al Olaya, Riyadh 12251, Saudi Arabia",
    "latitude": "24.713551649161143",
    "longitude": "46.67529568076134",
    "user_id": 486,
    "contact_person_name": "Sadiq Alotmi",
    "is_default": "0"
  },
  "preparation_time": "0",
  "table_id": null,
  "number_of_people": null,
  "table_order_id": null,
  "is_cutlery_required": "0",
  "add_on_ids": null,
  "details": [
    {
      "id": 452,
      "product_id": 12,
      "order_id": 100294,
      "price": 28.5,
      "product_details": {
        "id": 12,
        "name": "نصف دجاج على الفحم",
        "description": "...",
        "image": "2025-05-25-683309f32df10.png",
        "price": 23.5,
        "calories": 1563,
        "variations": [ ... ]
      },
      "variation": [
        {
          "name": "نوع الرز",
          "type": "single",
          "values": [
            {
              "label": "رز شعبي",
              "optionPrice": "23.5",
              "optionCalories": "1563",
              "addon_id": 0
            }
          ]
        }
      ],
      "discount_on_product": 0,
      "discount_type": "discount_on_product",
      "quantity": 3,
      "tax_amount": 3.72,
      "created_at": "2026-02-02T23:21:27.000000Z",
      "updated_at": "2026-02-02T23:21:27.000000Z",
      "add_on_ids": [],
      "variant": [],
      "add_on_qtys": [],
      "add_on_taxes": "[]",
      "add_on_prices": "[]",
      "add_on_tax_amount": 0
    }
  ],
  "delivery_man": null,
  "order_partial_payments": []
}
```

أي: **طلب واحد → كائن واحد → تفاصيل العناصر داخل `details[]`**.

---

### 4.2 تفاصيل الطلب (Order Details)

| البند | القيمة |
|-------|--------|
| **المسار** | `GET /api/v1/customer/order/details?order_id={id}` |
| **نوع الاستجابة** | **مصفوفة** (كل عنصر = صنف طلب) |
| **الاستخدام في التطبيق** | `response.data.forEach(... OrderDetailsModel.fromJson(...))` |

**مثال على شكل الاستجابة:**

```json
[
  {
    "id": 452,
    "product_id": 12,
    "order_id": 100294,
    "price": 28.5,
    "product_details": {
      "id": 12,
      "name": "نصف دجاج على الفحم",
      "description": "...",
      "image": "2025-05-25-683309f32df10.png",
      "price": 23.5,
      "calories": 1563,
      "variations": [ ... ],
      "add_ons": [],
      "tax": 15,
      "translations": [ ... ]
    },
    "variation": [
      {
        "name": "نوع الرز",
        "type": "single",
        "min": 0,
        "max": 0,
        "required": "on",
        "addon_id": 0,
        "values": [
          {
            "label": "رز شعبي",
            "optionPrice": "23.5",
            "optionCalories": "1563",
            "addon_id": 0
          }
        ]
      }
    ],
    "discount_on_product": 0,
    "discount_type": "discount_on_product",
    "quantity": 3,
    "tax_amount": 3.72,
    "created_at": "2026-02-02T23:21:27.000000Z",
    "updated_at": "2026-02-02T23:21:27.000000Z",
    "add_on_ids": [],
    "variant": [],
    "add_on_qtys": [],
    "add_on_taxes": [],
    "add_on_prices": [],
    "add_on_tax_amount": 0,
    "reviews_count": "0",
    "is_product_available": 1,
    "order": {
      "id": 100294,
      "user_id": 486,
      "order_amount": 70.5,
      "payment_status": "paid",
      "order_status": "confirmed",
      "payment_method": "clickpay",
      "transaction_reference": "a7c547b9-caa2-4802-aee6-61bf0a4761a0",
      "delivery_charge": 19.98,
      "delivery_address": { ... },
      "delivery_date": "2026-02-03",
      "delivery_time": "11:31:00"
    }
  }
]
```

أي: **مصفوفة عناصر → كل عنصر = صنف طلب + `product_details` + `variation` + كائن `order` مكرر**.

---

## 5. مقارنة شاملة

### 5.1 الإرسال: الدفع عند الاستلام vs الدفع أونلاين

| البند | الدفع عند الاستلام | الدفع أونلاين |
|-------|---------------------|----------------|
| **بناء الـ body** | نفس PlaceOrderBody | نفس PlaceOrderBody |
| **عدد طلبات الـ API قبل وضع الطلب** | 0 | 1 (initiate) |
| **طلب وضع الطلب** | مرة واحدة: `POST .../order/place` | مرة واحدة بعد نجاح الدفع: `POST .../order/place` |
| **حفظ الطلب محلياً** | لا | نعم (base64) ثم استعادته بعد الدفع |
| **payment_method في body الـ place** | `cash_on_delivery` | `clickpay` (أو البوابة المختارة) |
| **transaction_reference** | لا يُرسل | يُرسل (من redirect/عميل البوابة) |
| **محتوى cart و order_amount** | نفسه | نفسه (من الـ body المحفوظ ثم المُحدَّث بـ copyWith فقط) |

### 5.2 الاستجابة: Track vs Order Details

| البند | Track (`/order/track?order_id=`) | Details (`/order/details?order_id=`) |
|-------|---------------------------------|--------------------------------------|
| **نوع الاستجابة** | كائن واحد | مصفوفة |
| **معلومات الطلب الرئيسي** | في جذر الكائن (id, order_amount, payment_method, ...) | مكررة داخل كل عنصر في الحقل `order` |
| **تفاصيل المنتجات** | داخل مصفوفة `details[]` | كل عنصر في المصفوفة = صنف طلب مع `product_details` و `variation` |
| **الاستخدام في التطبيق** | `OrderModel.fromJson(response.data)` | `response.data.forEach(... OrderDetailsModel.fromJson(...))` |

### 5.3 ملخص الـ APIs

| الغرض | المسار | الطريقة | ملاحظة |
|-------|--------|---------|--------|
| وضع الطلب | `/api/v1/customer/order/place` | POST | مشترك بين الدفع عند الاستلام وأونلاين |
| بدء الدفع أونلاين | `/api/v1/payment-mobile/initiate` | POST | فقط عند الدفع أونلاين |
| تتبع الطلب | `/api/v1/customer/order/track?order_id=` | GET | يرجع كائن طلب واحد مع `details` |
| تفاصيل الطلب | `/api/v1/customer/order/details?order_id=` | GET | يرجع مصفوفة عناصر تفاصيل مع `order` داخل كل عنصر |

---

## 6. مراجع الكود

| الملف | الدور |
|-------|--------|
| `lib/view/screens/checkout/widget/confirm_button_view.dart` | بناء PlaceOrderBody، اختيار مسار الدفع (place مباشر أو initiate + حفظ + place لاحقاً) |
| `lib/data/model/body/place_order_body.dart` | تعريف PlaceOrderBody، Cart، OrderVariation، OrderVariationValueItem، toJson/fromJson |
| `lib/view/screens/checkout/payment_screen.dart` | معالجة redirect/deep link بعد الدفع، فك base64، copyWith، استدعاء placeOrder |
| `lib/data/repository/order_repo.dart` | placeOrder، initiatePaymentMobile، trackOrder، getOrderDetails |
| `lib/provider/order_provider.dart` | placeOrder، trackOrder، getOrderDetails، setPlaceOrder، getPlaceOrder |
| `lib/utill/app_constants.dart` | placeOrderUri، trackUri، orderDetailsUri، paymentMobileInitiateUri |

---

*آخر تحديث: بناءً على الكود الحالي في المشروع.*
