---
title: Sequential Order Numbering
created_at: 2020/09/24
excerpt: "In Workarea, orders (and other persisted data) are identified in a different manner from relational databases in order to avoid collisions. This guide explains how Workarea identifies orders, and how you might want to customize this for vanity purposes."
---

# Sequential Order Numbering

In relational database systems, identifiers are typically numbers that are automatically incremented. Many ecommerce systems, both in the past and still running today, are built around relational databases in order to persist their data. As a result, order identifiers tend to have a "numerical" look to them, so much so that the colloqualism "order number" has been adopted throughout the industry. In Workarea, orders (and other persisted data) are identified in a different manner to avoid collisions when the backing database store is distributed amongst many machines. This guide explains how Workarea identifies orders, and how you might want to customize this for vanity purposes.

## How Workarea Identifies Orders

When an order is created or instantiated, it is assigned a randomized 5-digit uppercase hexadecimal value as its ID. Workarea does this so order numbers are legible to human eyes, and will play nicely on a distributed database system such as MongoDB. This ID is used throughout the system to link orders to their Payment and Shipping information, as well as in the admin for display purposes. Although this order ID is "randomized", `SecureRandom` generation uses the current system time to determine a seed value for its algorithm, so when compared, they are actually sequential already:

```ruby
def test_sequential_order_numbering
  orders = [create_order, create_order]

  assert(orders.second < orders.first) # => true
end
```

However, Workarea's order IDs definitely don't look "numerical", and may be confusing to read for those used to relational systems. For this reason, you may want to customize how admins view orders on their site.

## Customizing Sequential Order Numbers

Some clients may want this display ID to be sequentially numbered. This is possible to customize, but there's one caveat to keep in mind when doing so: MongoDB is a **distributed database**, so in production there are many instances of Mongo running to make sure that your app's data is not only within reach, but can be retrieved efficiently. This can lead to gaps in the order numbers, as well as order numbers appearing out of order in the admin (since Workarea only shows you orders which have been placed).

If you do wish to customize this, however, we advise using the [Mongoid::Autoinc](https://github.com/suweller/mongoid-autoinc) plugin and decorating `Workarea::Order` like so:

```ruby
module Workarea
  decorate Order do
    decorated do
      include Mongoid::Autoinc

      field :number, type: Integer, default: 0

      increments :number

      # This will show the order number in the title of admin views.
      def name
        "#{super} (#{number})"
      end
    end
  end
end
```

You will probably also want to decorate places in the admin where this number would appear, such as the orders index page:

```diff
diff --git a/admin/app/views/workarea/admin/orders/index.html.haml b/admin/app/views/workarea/admin/orders/index.html.haml
index c6c4cb7d4e..d00f9f67de 100644
--- a/admin/app/views/workarea/admin/orders/index.html.haml
+++ b/admin/app/views/workarea/admin/orders/index.html.haml
@@ -87,6 +87,7 @@
                 = check_box_tag 'select_all', nil, false, id: 'select_all', class: 'checkbox__input', data: { bulk_action_select_all: '' }
                 = label_tag 'select_all', t('workarea.admin.bulk_actions.select_all'), class: 'checkbox__label'
             %th= t('workarea.admin.fields.id')
+            %th= t('workarea.admin.fields.number')
             %th= t('workarea.admin.fields.email')
             %th.align-right= t('workarea.admin.fields.total_price')
             %th= t('workarea.admin.fields.payment_status')
@@ -102,6 +103,7 @@
                   = label_tag dom_id(result), '', class: 'checkbox__label', title: t('workarea.admin.bulk_actions.add_summary_button')
               %td
                 = link_to result.id, order_path(result)
+                = result.number
                 = comments_icon_for(result)
                 = fraud_icon_for(result)
                 = append_partials('admin.order_index_icons', result: result)
```

However you wish to do this is up to you (and your client). The main goal here is to allow the sequential order numbering to live adjacent to the way orders are identified in the system. This is the most frictionless way to achieve what your admins are looking for, even though it may result in a duplicate number from time to time given a high level of traffic (such as thousands of orders per minute).

We still recommend working with your client to help them understand why Workarea's order IDs look the way they do, but if they push back enough, then using the above tips should allow you to make them happy without jeopardizing the performance of your application.
