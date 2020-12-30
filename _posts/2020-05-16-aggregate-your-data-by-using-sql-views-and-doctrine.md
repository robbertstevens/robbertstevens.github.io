---
layout: post
title:  "Aggregate your data by using SQL views and Doctrine."
date:   2020-06-21 13:37:00 +0100
---
When you have to aggregate data from you database, one of the options you have is using a [view](https://dev.mysql.com/doc/refman/5.7/en/views.html). A view in SQL is a stored
query that produces a result set.

In this post I am going to explore the use of views with Doctrine, by showing an example how to find the minimum, maximum,
and average price for each category for the products in this category. 

## The setup.
I am going to assume you already have a project with Doctrine configured. So let's get started, I need to have data to 
aggregate. First we need an `Product` entity.

```php
class Product {
    private string $name;
    private string $category;
    private float $price;
}
```
<small>Let's not have a discussion why it's bad to use floats for prices and currency there are enough resources about this. That
discussion is out of the scope of this post.</small>

Now I have an entity we can start to aggregate data for this. In the beginning I said I wanted to know the minimum, maximum, 
and average prices per category.

If I would make an SQL query for this it would look like this:
```sql
SELECT category, min(price) as min_price, max(price) as max_price, avg(price) as average_price FROM product GROUP BY category
```

An SQL view is a stored query which serves as a virtual table. So in order to create the view I only have to prepend 
`CREATE VIEW view_category_prices` to the query.

```sql
CREATE VIEW view_category_prices AS
SELECT category, min(price) as min_price, max(price) as max_price, avg(price) as average_price FROM product GROUP BY category
```

I want to have a migration for this, but Doctrine doesn't know how to generate migrations for views. So I have to
create the migration by hand. I created an empty migration by running `php bin/console doctrine:migrations:generate`.

Then I added this code in my migration:
```php
final class Version20200706200511 extends AbstractMigration
{
    public function up(Schema $schema) : void
    {
        $this->addSql('CREATE VIEW view_product_prices_category AS SELECT category, min(price) as min_price, max(price) as max_price, avg(price) as average_price FROM product GROUP BY category');
    }

    public function down(Schema $schema) : void
    {
        $this->addSql('DROP VIEW product_prices_category');
    }
}
```

Now it's time to create the entity which I am going to use for view. I choose to name my entity the same as I named the view, 
but you can name it whatever you want, just remember to add the `@Table` annotation.

```php
/**
 * @ORM\Entity(repositoryClass=ViewProductPricesCategoryRepository::class, readOnly=true)
 */
class ViewCategoryPrices {
    /**
     * @ORM\Id()
     * @ORM\Column(type="string", length=255)
     */
    private string $category;
    
    /**
     * @ORM\Column(type="integer", length=255)
     */
    private int $minPrice;
    
    /**
     * @ORM\Column(type="integer", length=255)
     */
    private int $maxPrice;
    
    /**
     * @ORM\Column(type="decimal", length=255)
     */
    private float $averagePrice;
}
```

As you can see I've added a `@Id` annotation to the category. I had to do this because Doctrine wants all entities to have a
primary key, and `category` will always be unique because of the `GROUP BY category` in the query this was the most 
suited column to put it on. 

I've also added `readonly=true` to the `@Entity` annotation, because this view will be readonly
so Doctrine should even have to consider is while flushing other entities.

The repository `ViewCategoryPricesRepository` is the basic repository you get when you create an entity 
with the `php bin/console make:entity` command. 

The last step I have to do is added a few lines to `doctrine.yml`
 
```yaml
dbal:
  schema_filter: ~^(?!view_)~
```

This will filter out entities that start with `view` when creating migrations. I choose this as a prefix for my views, 
but you can use anything you want.

## Conclusion
Because of the nature of a view you can lose performance when querying against the view. Another thing to think about is
the extra maintenance of introducing views, when a table does not exist anymore or changes its name you have to change 
it in the view too.

So whenever you should use views or not, should be based on what your use case needs.
