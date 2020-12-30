---
layout: post
title:  "Creating unique numbering sequences using a custom identifier strategy with Doctrine."
date:   2020-05-16 13:37:00 +0100
---
Recently in a project I am working on, I had to invoice a bunch of suppliers. These invoices had to be numbered with a 
prefix followed by a number starting from 1. This had to be done for every supplier. So for supplier X we would have 
invoices numbered like this: x.1, x.2, etc and for supplier Y we would have them numbered like: y.1, y.2, etc. 

I've created a small proof of concept how to implement this in [Symfony](https://symfony.com/) and 
[Doctrine](https://www.doctrine-project.org/index.html). In this post I am going to explain how I did this.

## Introduction to the problem.
Normally when I create an entity I run the `php bin/console make:entity` command and follow the wizard. This will 
generate an entity which looks roughly like this.

```php
class Invoice
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private int $id;

    public function getId(): ?int {
        return $this->id;
    }
}
```

This will create a table in the database which increments the id whenever you insert a new `Invoice`.
The problem we face now is that the ID will just increment all the invoices the same. We can't make the integer increment
based on the supplier.

## How I solved this.
I started out with two very simple classes. 

```php
class Supplier {
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private int $id;
    
    /**
     * @ORM\Column(type="string", length=255)
     */
    private string $prefix;
}

class Invoice {
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private int $id;
    
    /**
     * @ORM\ManyToOne(targetEntity=Supplier::class, inversedBy="invoices")
     * @ORM\JoinColumn(nullable=false)
     */
    private Supplier $supplier;
}
```
Then I created a class called `SupplierInvoiceSequence`
```php
class SupplierInvoiceSequence {
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private int $id;

    /**
     * @ORM\Column(type="integer")
     */
    private int $sequence_number;

    public function __construct() {
        $this->sequence_number = 0;
    }
}
```
The `SupplierInvoiceSequence` will hold the current invoice number for each individual supplier. I've started the sequence 
at 0, but you can start it at any number you want.

I had to change the `Supplier` class, so it automatically adds an entry in the sequence table. I added the following 
code to the `Supplier` class.

```php
class Supplier {
    // Same code as before
    
    /**
     * @ORM\OneToOne(targetEntity=InvoiceSequence::class, cascade={"persist", "remove"})
     * @ORM\JoinColumn(nullable=false)
     */
    private SupplierInvoiceSequence $supplierInvoiceSequence;
    
    public function __construct() {
        $this->supplierInvoiceSequence = new SupplierInvoiceSequence();
    }
}
```

The only thing I had to do now is change the 'GeneratedValue' from the invoice, so it uses a combined value of the prefix
and the value stored in the `SupplierInvoiceSequence` entity.

The way you can do this is change the [identifier generation strategy](https://www.doctrine-project.org/projects/doctrine-orm/en/2.6/reference/basic-mapping.html#identifier-generation-strategies) 
for the entity. While reading the documentation, I was really excited about the `TABLE` strategy because this looked 
exactly what I needed.

> Tells Doctrine to use a separate table for ID generation. This strategy provides full portability. 

Sadly this strategy has a note that it is not implemented yet. So I choose `CUSTOM` instead.
 
Because I choose `CUSTOM` I had to create my own ID generator. I did this by extending doctrine's 
`AbstractIdGenerator` class. This abstract class has two methods, `generate` and `isPostInsertGenerator`.

I only had to implement the `generate` method, because the `isPostInsertGenerator` is a flag whenever the generator
has to be ran before or after the entity is inserted into the database.

```php
class SupplierInvoiceSequenceGenerator extends AbstractIdGenerator
{
    public function generate(EntityManager $em, $entity)
    {
        $sequenceId = $entity->getSupplier()->getSupplierInvoiceSequence()->getId();

        $query = $em->createQuery("UPDATE SupplierInvoiceSequence i SET i.sequence_number = i.sequence_number + 1 WHERE i.id = :id");
        $query->setParameter("id", $sequenceId);
        $query->execute();

        $increment = $em->createQuery("SELECT i.sequence_number FROM SupplierInvoiceSequence i WHERE i.id = :id");
        $increment->setParameter("id", $sequenceId);

        return "{$entity->getSupplier()->getPrefix()}.{$increment->getSingleResult()['sequence_number']}";
    }
}
```
As you can see what I did in the `generate` method is I run a query which increments the sequence number by 1 for the
`SupplierInvoiceSequence` related to the supplier I am currently creating an `Invoice` for. Then I select the incremented 
ID and concatenate it with the supplier prefix.

Well you might think this is not going to work, he is return a `string` instead of an integer. You are completely right, 
there is one last think I have to do. 

I had to change the `@GeneratedValue()` annotation in the `Invoice` class to use my custom generator, I also had to add the 
`@CustomIdGenerator` annotation to define which ID generator Doctrine will use. 

Lastly I had to change the column type to `string` because the sequence number needs to have the format `prefix.number`. 

```php
class Invoice {
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue(strategy="CUSTOM")
     * @ORM\CustomIdGenerator(class="SupplierInvoiceSequenceGenerator")
     * @ORM\Column(type="string")
     */
    private int $id;
}
```

Now the invoice number will be generated based on the supplier prefix and unique sequence id.

## Conclusion
For my use case this works because the supplier will always exist before I create invoices for it. If you have another 
scenario where you create the supplier at the same time as an invoice this might not work in the same way. I hope you got 
a push in the right direction. 
 
