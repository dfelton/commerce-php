---
title: EAV and Extension Attributes | Commerce PHP Extensions
description: Learn about the main types of attributes you can use to extend Adobe Commerce and Magento Open Source.
---

# EAV and extension attributes

There are two types of attributes you can use to extend Adobe Commerce and Magento Open Source functionality:

*  Custom and Entity-Attribute-Value (EAV) attributes—Custom attributes are those added on behalf of a merchant. For example, a merchant might need to add attributes to describe products, such as shape or volume. A merchant can add these attributes in the [Admin](https://glossary.magento.com/magento-admin) panel. See the [merchant documentation](https://docs.magento.com/user-guide/stores/attributes.html) for information about managing custom attributes.

   Custom attributes are a subset of EAV attributes. Objects that use EAV attributes typically store values in several MySQL tables. The `Customer` and `Catalog` modules are the primary models that use EAV attributes. Other modules, such as `ConfigurableProduct`, `GiftMessage`, and `Tax`, use the EAV functionality for `Catalog`.

*  [Extension attributes](https://glossary.magento.com/extension-attribute). Extension attributes are new in Adobe Commerce and Magento Open Source. They are used to extend functionality and often use more [complex data](https://glossary.magento.com/complex-data) types than custom attributes. These attributes do not appear in the Admin.

## Custom attributes

`CustomAttributesDataInterface` defines the methods that are called to get and set custom attributes, including `getCustomAttributes()`.

A [module](https://glossary.magento.com/module) has a set of built-in attributes that are always available. The `Catalog` module has several attributes that are defined as EAV attributes, but are treated as built-in attributes. These attributes include:

*  attribute_set_id
*  created_at
*  group_price
*  media_gallery
*  name
*  price
*  [sku](https://glossary.magento.com/sku)
*  status
*  store_id
*  tier_price
*  type_id
*  updated_at
*  visibility
*  weight

In this case, when `getCustomAttributes()` is called, the system returns only custom attributes that are not in this list.

The `Customer` module provides a `system` option for its attributes. As a result, the `getCustomAttributes()` method only returns those EAV attributes that are not defined as `system` attributes. If you create custom attributes programmatically, set the `system` option to 'false' if you want to include the attribute in the `custom_attributes` array.

<InlineAlert variant="info" slots="text"/>

As of version 2.3.4, Adobe Commerce and Magento Open Source caches all system EAV attributes as they are retrieved. This behavior is defined in each affected module's `di.xml` file as the `attributesForPreload` argument for `<type name="Magento\Eav\Model\Config">`. Developers can cache custom EAV attributes by running the `bin/magento config:set dev/caching/cache_user_defined_attributes 1` command. This can also be done from the Admin while in Develop mode by setting **Stores** > Settings **Configuration** > **Advanced** > **Developer** > **Caching Settings** > **Cache User Defined Attributes** to **Yes**. Caching EAV attributes while retrieving improves performance as it decreases the amount of insert/select requests to the DB, but it increases the cache network size.

### Adding Customer EAV attribute for backend only

Customer EAV attributes are created using a [data patches](declarative-schema/patches.md).

<InlineAlert variant="warning" slots="text"/>

Both the `save()` and `getResource()` methods for `Magento\Framework\Model\AbstractModel` have been marked as `@deprecated` since 2.1 and should no longer be used.

```php

namespace Magento\Customer\Setup\Patch\Data;

use Magento\Customer\Model\Customer;
use Magento\Customer\Setup\CustomerSetupFactory;
use Magento\Framework\App\ResourceConnection;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use Magento\Framework\Setup\Patch\PatchVersionInterface;

/**
 * Class add customer example attribute to customer
 */
class AddCustomerExampleAttribute implements DataPatchInterface
{
    private ModuleDataSetupInterface $moduleDataSetup;

    private CustomerSetupFactory $customerSetupFactory;

    public function __construct(
        ModuleDataSetupInterface $moduleDataSetup,
        CustomerSetupFactory $customerSetupFactory
    ) {
        $this->moduleDataSetup = $moduleDataSetup;
        $this->customerSetupFactory = $customerSetupFactory;
    }

    public function apply(): void
    {
        $customerSetup = $this->customerSetupFactory->create(['setup' => $this->moduleDataSetup]);
        $customerSetup->addAttribute(Customer::ENTITY, 'attribute_code', [
            // Attribute options (list of options can be found below)
        ]);
    }

    public static function getDependencies(): array
    {
        return [
            UpdateIdentifierCustomerAttributesVisibility::class,
        ];
    }

    public function getAliases(): array
    {
        return [];
    }
}
```

<InlineAlert variant="success" slots="text"/>

The scope of the Customer custom attribute is Global only, while other entities support the Global, Website, and StoreView scopes.

## Extension attributes

Use `ExtensibleDataInterface` to implement extension attributes. In your code, you must define `getExtensionAttributes()` and `setExtensionAttributes(*ExtensionInterface param)`.

`public function getExtensionAttributes();`

Most likely, you will want to extend interfaces defined in the `Api/Data` directory of a module.

### Declare extension attributes

You must create a `<Module>/etc/extension_attributes.xml` file to define a module's extension attributes:

```xml
<config>
    <extension_attributes for="Path\To\Interface">
        <attribute code="name_of_attribute" type="datatype">
           <resources>
              <resource ref="permission"/>
           </resources>
           <join reference_table="" reference_field="" join_on_field="">
              <field>fieldname</field>
           </join>
        </attribute>
    </extension_attributes>
</config>
```

where:

|Keyword|Description|Example|
|--- |--- |--- |
| for | The fully-qualified type name with the namespace that processes the extensions. The value must be a type that implements `ExtensibleDataInterface`. The interface can be in a different module. | `Magento\Quote\Api\Data\TotalsInterface` |
| code | The name of the attribute. The attribute name should be in snake case (the first letter in each word should be in lowercase, with each word separated by an underscore). | `gift_cards_amount_used` |
| type | The data type. This can be a simple data type, such as string or integer, or complex type, such as an interface. | `float`<br />`Magento\CatalogInventory\Api\Data\StockItemInterface` |
| ref | Optional. Restricts access to the extension attribute to users with the specified permission. | `Magento_CatalogInventory::cataloginventory` |
| reference_table | The table involved in a join operation. See [Searching extension attributes](#searching-extension-attributes) for details. | `admin_user` |
| reference_field | Column in the `reference_table`. | `user_id` |
| join_on_field | The column of the table associated with the interface specified in the `for` keyword that will be used in the join operation. | `store_id` |
| field | One or more fields present in the interface specified in the `type` keyword.<br />You can specify the `column=""` keyword to define the column in the reference_table to use. The field value specifies the property on the `interface` which should be set. | `<field>firstname</field>`<br />`<field>lastname</field>`<br />`<field>email</field>`<br /><br />`<field column="customer_group_code">code</field>` |

### Searching extension attributes

The system uses a join directive to add external attributes to a collection and to make the collection filterable. The `join` element in the `extension_attributes.xml` file defines which object fields and the database table/column to use as the source of a search.

In the following example, an attribute named `stock_item` of type `Magento\CatalogInventory\Api\Data\StockItemInterface` is being added to the `Magento\Catalog\Api\Data\ProductInterface`.

```xml
<extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
    <attribute code="stock_item" type="Magento\CatalogInventory\Api\Data\StockItemInterface">
        <join reference_table="cataloginventory_stock_item" reference_field="product_id" join_on_field="entity_id">
            <field>qty</field>
        </join>
    </attribute>
</extension_attributes>
```

When `getList()` is called, it returns a list of `ProductInterface`s. When it does this, the code populates the `stock_item` with a joined operation in which the `StockItemInterface`’s `qty` property comes from the `cataloginventory_stock_item` table where the `Product`'s `entity_Id` is joined with the `cataloginventory_stock_item.product_id` column.

When you add search extension attributes, you must consider that this can cause ambiguity in the selection of fields in the resulting SQL query when using REST APIs.
In these cases, the REST call must explicitly specify both the table name and field to use for selecting.

For example, the following configuration may introduce ambiguity when getting orders via REST API. The configuration constructs a query like `SELECT .... FROM sales_order AS main_table LEFT JOIN sales_order`. This creates an ambiguity for all columns from the `sales_order` table in that MySQL cannot determine if it should take them from the `main_table` or from the `sales_order` from the `JOIN` clause.

```xml
<config>
    <extension_attributes for="Magento\Sales\Api\Data\OrderInterface">
        <attribute code="field1" type="int">
            <join reference_table="sales_order" join_on_field="entity_id" reference_field="entity_id">
                <field>field1</field>
            </join>
        </attribute>
    </extension_attributes>
</config>
```

**REST API Endpoint:**

`GET http://<host>/rest/default/V1/orders`

**Payload:**

```http
searchCriteria[filter_groups][0][filters][0]
[field]=main_table.created_at&searchCriteria
[filter_groups][0][filters][0][value]=2021-09-14%2000:00:00
&searchCriteria[filter_groups][0][filters][0]
[conditionType]=from
&searchCriteria[filter_groups][1][filters][0]
[field]=main_table.created_at
&searchCriteria[filter_groups][1][filters][0]
[value]=2021-09-14%2023:59:59
&searchCriteria[filter_groups][1][filters][0]
[conditionType]=to
&searchCriteria[pageSize]=10
&searchCriteria[currentPage]=86
```

### Extension attribute authentication

Individual fields that are defined as extension attributes can be restricted, based on existing permissions. This feature allows extension developers to restrict access to data. See [Web API authentication overview](https://developer.adobe.com/commerce/webapi/get-started/authentication/) for general information about authentication in Magento.

The following [code sample](https://github.com/magento/magento2/blob/2.4/app/code/Magento/CatalogInventory/etc/extension_attributes.xml) defines `stock_item` as an extension attribute of the `CatalogInventory` module. `CatalogInventory` is treated as a "third-party extension". Access to the inventory data is restricted because the quantity of in-stock item may be competitive information.

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Api/etc/extension_attributes.xsd">
    <extension_attributes for="Magento\Catalog\Api\Data\ProductInterface">
        <attribute code="stock_item" type="Magento\CatalogInventory\Api\Data\StockItemInterface">
            <resources>
                <resource ref="Magento_CatalogInventory::cataloginventory"/>
            </resources>
        </attribute>
    </extension_attributes>
</config>
```

In this example, the `stock_item` attribute is restricted to only the users who have the `Magento_CatalogInventory::cataloginventory` permission. As a result, an anonymous or unauthenticated user issuing a `GET <host>/rest/<store_code>/V1/products/<sku>` request will receive product information similar to the following:

```json
    {
      "sku": "tshirt1",
      "price": "20.00",
      "description": "New JSmith design",
      "extension_attributes": {
        "logo size": "small"
      },
      "custom_attributes": {
        "artist": "James Smith"
      }
    }
```

However, an authenticated user with the permission `Magento_CatalogInventory::cataloginventory` receives the additional `stock_item` field:

```json
    {
      "sku": "tshirt1",
      "price": "20.00",
      "description": "New JSmith design",
      "extension_attributes": {
        "logo size": "small",
        "stock_item" : {
          "status" : "in_stock"
          "quantity": 70
        }
      },
      "custom_attributes": {
        "artist": "James Smith"
      }
    }
```

This only works for extension attributes (those attributes defined in an `extension_attributes.xml` file). There are no permission restrictions on the rest of the returned data. For example, there is no way to restrict `custom_attributes`.

### Extension interfaces

An `ExtensionInterface` will be empty if no extension attributes have been added. In the following example—in an unmodified installation—`CustomerExtensionInterface` will be generated, but will be empty:

```php
use Magento\Framework\Api\ExtensionAttributesInterface;
interface CustomerExtensionInterface extends ExtensionAttributesInterface
{
}
```

However, if an extension similar to the following has been defined, the interface will not be empty:

```xml
<extension_attributes for="Magento\Customer\Api\Data\CustomerInterface">
    <attribute code="attributeName" type="Magento\Some\Type[]" />
</extension_attributes>
```

### Troubleshoot EAV attributes

- If you have issues when using `setup:upgrade`, verify `__construct` uses the method `EavSetupFactory` not `EavSetup`. You should not directly inject `EavSetup` in extension code. Check your custom code and purchased modules and extensions to verify. After changing the methods, you should be able to properly deploy.
- When invoking `EavSetup::addAttribute()`, the `$attr` array is passed to a `\Magento\Eav\Model\Entity\Setup\PropertyMapperInterface` object, and the valid array keys may differ from their column name in `eav_attribute`, `catalog_eav_attribute`, `customer_eav_attribute`, etc. When invoking `EavSetup::updateAttribute()`, and passing an array for the third parameter `$field`, the array is _not_ passed `PropertyMapperInterface::map()`, and array keys should directly correlate with their corresponding table column name.
  - Note: `addAttribute()` can also be used to update existing attributes.

## Add EAV attribute options reference

The following tables provide references for the `addAttribute` and `updateAttribute` methods of `Magento\Eav\Setup\EavSetup` method. They contains available options and mappings when creating and/or updating an attribute.

### Attribute Options: _(valid for all available `entity_type_code` entities)_

- Mapper: `\Magento\Eav\Model\Entity\Setup\PropertyMapper`

|Option|Map|Description|Default Value|
|--- |--- |--- |---
|`attribute_model`|`attribute_model`|EAV Attribute attribute_model|`null`|
|`backend_model`|`backend`|The backend class model for the attribute.|`null`|
|`default_value`|`default`|Default value for the attribute|`null`|
|`frontend_class`|CSS class name(s) added to the input field for the attribute during rendering of a form.|`null`|
|`frontend_input`|`input`|Type of input field to be used in forms. Valid values: `button|checkbox|checkboxes|collection|column|date|editor|fieldset|file|gallery|hidden|image|imagefile|label|link|multiline|multiselect|note|obsure|password|radio|radios|reset|select|submit|text|textarea|time`|`text`|
|`frontend_label`|`label`|Label to use on frontend|`null`|
|`frontend_model`|`frontend`|The frontend class model for the attribute.|`null`|
|`is_global`|`global`|Defines the scope(s) that the attirbute can be saved at. Valid value is one of `SCOPE_GLOBAL`, `SCOPE_WEBSITE`, `SCOPE_STORE` from the `\Magento\Eav\Model\Entity\Attribute\ScopedAttributeInterface` class.|`SCOPE_GLOBAL`|


### Attribute Options: _(`catalog_product` and `catalog_category` entities)_

- Mappers:
  - `\Magento\Catalog\Model\ResourceModel\Setup\PropertyMapper`
  - `\Magento\CatalogSearch\Model\ResourceModel\Setup\PropertyMapper`
  - `\Magento\ConfigurableProduct\Model\ResourceModel\Setup\PropertyMapper`

<InlineAlert variant="info" slots="text"/>

While all options below are saved for both `catalog_product` and `catalog_category` entities, many of these mentioned below have no effect for `catalog_category` entities, and are only applicable to entities of `catalog_product`.

|Option|Map|Description|Default Value
|--- |--- |--- |--- |---
|`apply_to`|`apply_to`|Correlates to `type_id` of `catalog_product` entities. Expects comma separated list of `type_id` value(s). `null` applies the attribute to all product types _(not applicable for `catalog_category` entities)_. |`null`|
|`is_comparable`|`comparable`|Defines if attribute is shown when comparing products.|`0`|
|`is_filterable`|`filterable`|Defines if attribute is added to the layered navigation block of category pages.|`0`|
|`is_filterable_in_grid`|`is_filterable_in_grid`|Defines if attribute can be used to filter on product grid in Admin|`0`|
|`is_filterable_in_search`|`filterable_in_search`|Defines if the attribute is added to the layered navigation block of search results pages.|`0`|
|`is_html_allowed_on_front`|`is_html_allowed_on_front`|Defines if HTML needs to be escaped when rendering frontend output.|`0`|
|`is_used_in_grid`|`is_used_in_grid`|Defines if the attribute will be added to the collection for frontend product listings such category pages and search results pages. Useful if your frontend design template(s) require this data for your them. If your theme does not require the attribute in this scenario, you may want to leave this off for performance reasons.|`0`|
|`is_visible_in_grid`|`is_visible_in_grid`|Defines if attribute can be added to the columns of the primary product grid of the adminhtml area.|`0`|
|`frontend_input_renderer`|`input_renderer`|If your attribute requires custom html markup when it's field for a form, a class may here for such needs.|`null`|


|attribute_set|Name of the attribute set the new attribute will be assigned to. Works in combination with **group** or empty **user_defined**||
|group|Attribute group name or ID||







|note|EAV Attribute note||
|option|EAV Attribute Option values||
|position|Catalog EAV Attribute position|0|
|required|EAV Attribute is_required|1|
|searchable|Catalog EAV Attribute is_searchable|0|
|sort_order|EAV Entity Attribute sort_order||
|source|EAV Attribute source_model||
|system|Declares the attribute as a system attribute.|1|
|table|EAV Attribute backend_table||
|type|EAV Attribute backend_type|varchar|
|unique|EAV Attribute is_unique|0|
|used_for_promo_rules|Catalog EAV Attribute is_used_for_promo_rules|0|
|used_for_sort_by|Catalog EAV Attribute used_for_sort_by|0|
|used_in_product_listing|Catalog EAV Attribute used_in_product_listing|0|
|user_defined|EAV Attribute is_user_defined|0|
|visible_in_advanced_search|Catalog EAV Attribute is_visible_in_advanced_search - defines if attribute will appear on the Advanced Search form|0|
|visible_on_front|Catalog EAV Attribute is_visible_on_front - defines attribute visibility on frontend|0|
|visible|Catalog EAV Attribute is_visible - defines visibility in Admin, won't be available for changing a value in the admin interface if set to 0|1|
|wysiwyg_enabled|Catalog EAV Attribute is_wysiwyg_enabled - used for enabling wysiwyg editor for an attribute. Works for textarea only|0|
