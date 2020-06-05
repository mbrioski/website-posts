---
title: "Delta migration from Magento1 to Magento2"
date: 2020-06-04T13:35:20+02:00
draft: false
tags: [ "Magento2", "Data migration"  ]
categories: [ "Magento2" ]
---

This article goes in details inside [Magento2 data migration tool](https://github.com/magento/data-migration-tool) and finds a way to create a correct configuration for delta migration since the beginning and how hack a little bit the code to achieve a really detail data migration.

If you have not read the article about the data migration tool, i will recommend you to have a look to [this post](../M1toM2DataMigration)

## Delta migration command

Inside *config.xml* file, defined for every magento1 version, the most important part is related to the definition of **steps**: the tag steps define, using the tag step, the classes responsable to obtain the data from Magento1 that has to be transferred.

Steps defined into **data steps** are strictly connected with the ones inside **steps delta**: it is possible infact have a delta migration of same data, if only a previous data migration for the same done has been executed. This happens because during data migration, data migration tool creates some supports tables inside the magento1 db, names with *m2_* that are later used by the delta migration to track what has to be migrated.

By default data e delta steps are defined in this way:

```
<steps mode="data">
        <step title="Data Integrity Step">
            <integrity>Migration\Step\DataIntegrity\Integrity</integrity>
        </step>
        <step title="EAV Step">
            <integrity>Migration\Step\Eav\Integrity</integrity>
            <data>Migration\Step\Eav\Data</data>
            <volume>Migration\Step\Eav\Volume</volume>
        </step>
        <step title="Customer Attributes Step">
            <integrity>Migration\Step\Customer\Integrity</integrity>
            <data>Migration\Step\Customer\Data</data>
            <volume>Migration\Step\Customer\Volume</volume>
        </step>
        <step title="Map Step">
            <integrity>Migration\Step\Map\Integrity</integrity>
            <data>Migration\Step\Map\Data</data>
            <volume>Migration\Step\Map\Volume</volume>
        </step>
        <step title="Url Rewrite Step">
            <integrity>Migration\Step\UrlRewrite\Version191to2000</integrity>
            <data>Migration\Step\UrlRewrite\Version191to2000</data>
            <volume>Migration\Step\UrlRewrite\Version191to2000</volume>
        </step>
        <step title="Log Step">
            <integrity>Migration\Step\Log\Integrity</integrity>
            <data>Migration\Step\Log\Data</data>
            <volume>Migration\Step\Log\Volume</volume>
        </step>
        <step title="Ratings Step">
            <integrity>Migration\Step\Ratings\Integrity</integrity>
            <data>Migration\Step\Ratings\Data</data>
            <volume>Migration\Step\Ratings\Volume</volume>
        </step>
        <step title="ConfigurablePrices step">
            <integrity>Migration\Step\ConfigurablePrices\Integrity</integrity>
            <data>Migration\Step\ConfigurablePrices\Data</data>
            <volume>Migration\Step\ConfigurablePrices\Volume</volume>
        </step>
        <step title="OrderGrids Step">
            <integrity>Migration\Step\OrderGrids\Integrity</integrity>
            <data>Migration\Step\OrderGrids\Data</data>
            <volume>Migration\Step\OrderGrids\Volume</volume>
        </step>
        <step title="Tier Price Step">
            <integrity>Migration\Step\TierPrice\Integrity</integrity>
            <data>Migration\Step\TierPrice\Data</data>
            <volume>Migration\Step\TierPrice\Volume</volume>
        </step>
        <step title="SalesIncrement Step">
            <integrity>Migration\Step\SalesIncrement\Integrity</integrity>
            <data>Migration\Step\SalesIncrement\Data</data>
            <volume>Migration\Step\SalesIncrement\Volume</volume>
        </step>
        <step title="Inventory Step">
            <integrity>Migration\Step\Inventory\Integrity</integrity>
            <data>Migration\Step\Inventory\Data</data>
            <volume>Migration\Step\Inventory\Volume</volume>
        </step>
        <step title="PostProcessing Step">
            <data>Migration\Step\PostProcessing\Data</data>
        </step>
    </steps>
    <steps mode="delta">
        <step title="Customer Attributes Step">
            <delta>Migration\Step\Customer\Delta</delta>
            <volume>Migration\Step\Customer\Volume</volume>
        </step>
        <step title="Map Step">
            <delta>Migration\Step\Map\Delta</delta>
            <volume>Migration\Step\Map\Volume</volume>
        </step>
        <step title="Log Step">
            <delta>Migration\Step\Log\Delta</delta>
            <volume>Migration\Step\Log\Volume</volume>
        </step>
        <step title="OrderGrids Step">
            <delta>Migration\Step\OrderGrids\Delta</delta>
            <volume>Migration\Step\OrderGrids\Volume</volume>
        </step>
        <step title="SalesIncrement Step">
            <delta>Migration\Step\SalesIncrement\Delta</delta>
            <volume>Migration\Step\SalesIncrement\Volume</volume>
        </step>
        <step title="Inventory Step">
            <delta>Migration\Step\Inventory\Delta</delta>
            <volume>Migration\Step\Inventory\Volume</volume>
        </step>
    </steps>
```

Like you can see the only possible data that can be migrated with the delta command is the one related to *customers, orders and inventory*

## Migrate only some tables

If you need to migrate only some data from Magento1 to Magento2, like for example only the products, is possible to customise the steps inside config.xml file, in order to keep your personalised classes containing only the data you want to migrate.

### Migrate only products 

You can define your config.xml file like:

```
<steps mode="data">
        <step title="Map Step">
            <data>Migration\Step\Map\Data</data>
        </step>
        <step title="ConfigurablePrices step">
            <data>Migration\Step\ConfigurablePrices\Data</data>
        </step>
    </steps>
```

And then check what happen inside the class *Migration\Step\ConfigurablePrices\Data*.

The core of this class is the method `public function perform()` and in particular the line `$sourceDocuments = $this->source->getDocumentList();`: this line has the responability to fill the variable **$sourceDocuments** with the tables used for retrieve the data.

If you get this, you can easily understand that could be enoght to replace the method `$this->source->getDocumentList()`with the tables you want to migrate:

```
$sourceDocuments =  [
            'catalog_product_bundle_option',
            'catalog_product_bundle_option_value',
            'catalog_product_bundle_price_index',
            'catalog_product_bundle_selection',
            'catalog_product_bundle_selection_price',
            'catalog_product_bundle_stock_index',
            'catalog_product_enabled_index',
            'catalog_product_entity',
            'catalog_product_entity_datetime',
            'catalog_product_entity_decimal',
            'catalog_product_entity_gallery',
            'catalog_product_entity_group_price',
            'catalog_product_entity_int',
            'catalog_product_entity_media_gallery',
            'catalog_product_entity_media_gallery_value',
            'catalog_product_entity_text',
            'catalog_product_entity_tier_price',
            'catalog_product_entity_varchar',
            'catalog_product_link',
            'catalog_product_link_attribute',
            'catalog_product_link_attribute_decimal',
            'catalog_product_link_attribute_int',
            'catalog_product_link_attribute_varchar',
            'catalog_product_link_type',
            'catalog_product_option',
            'catalog_product_option_price',
            'catalog_product_option_title',
            'catalog_product_option_type_price',
            'catalog_product_option_type_title',
            'catalog_product_option_type_value',
            'catalog_product_relation',
            'catalog_product_website',
        ];
```

Now you can remove all products from your Magento2 database and run again data migration:

```
​```php bin/magento migrate:data -r vendor/magento/data-migration-tool/etc/opensource-to-commerce/1.9.4.4/config.xml```
```



