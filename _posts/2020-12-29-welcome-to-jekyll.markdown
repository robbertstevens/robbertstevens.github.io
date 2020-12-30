---
layout: post
title:  "Generating XML files by using the strategy pattern"
date:   2020-12-29 15:42:55 +0100
categories: jekyll update
---
A while back we had a problem in a project of ours in which we had to send a couple of different
XML files to a remote server. The way we had to transfer was the same for all types of XML files.
We started out with various the classes that looked like this.

```php
final class XmlTransferCustomer
{
    private CustomerXmlGenerator $generator;

    public function transfer(Customer $entity)
    {
        /** @var Xml $xml */
        $xml = $this->generator->generate($entity);
        $this->storage->put($xml->getFilePath(), $xml->getContent());
    }
}

final class CustomerXmlGenerator
{
    public function generate(Customer $entity)
    {
        return new Xml('filepath.xml', '<?xml version="1.0" encoding="UTF-8"?>');
    }
}
```

As you can imagine after a while we got a lot of the same classes. So we searched for a solution.
We came up with something that uses the [strategy pattern](https://en.wikipedia.org/wiki/Strategy_pattern).

## Strategy Pattern
The strategy pattern is a design pattern in which you define a generic task with different ways to get
the same result, in our case an XML file. All the different ways to generate an XML file will be split in
different classes, called 'strategies'. The 'task' class will then select which strategy is the right one
to run the logic for a given input. Let me show you the gist of our solution.

## Implementation
First we started with defining an interface for the entities that had to be transferred,
`TransferableInterface`. This was just an empty interface, just for the purpose of recognizing
the entities that can be passed to the `XmlTransfer` class. The `XmlTransfer` class looks like
this now.

```php
final class XmlTransfer
{
    private XmlGenerator $generator;

    public function transfer(TransferableInterface $entity)
    {
        /** @var Xml $xml */
        $xml = $this->generator->generate($entity);
        $this->storage->put($xml->getFilePath(), $xml->getContent());
    }
}
```

As you can see the class we added is basically the same as `XmlTransferCustomer`, but instead of accepting a
`Customer`-object we accept the `TransferableInterface`. Now we had to create a new `XmlGenerator`-class
which accepts a `TransferableInterface`-object.

```php
final class XmlGenerator
{
    private XmlGeneratorStrategyInterface[] $generators;

    public function generate(TransferableInterface $entity): Xml
    {
        foreach ($this->generators as $generator) {
            if (!$generator->supports($entity)) {
                continue;
            }
            return $this->generator->generate($entity);
        }
    }
}
```

In the code you see that we loop over an array called generators, this is an array which only contains classes
of the type `XmlGeneratorStrategyInterface`, the interface looks like this.

```php
interface XmlGeneratorStrategyInterface {
    public function supports(TransferableInterface $entity): bool;
    public function generator(TransferableInterface $entity): Xml;
}
```
We keep looping over the `$this->generators` list until we find a generator that supports the given
entity. In the example implementation we won't do anything if none of the generators support the
given entity, but you could throw an exception or a [null object](https://en.wikipedia.org/wiki/Null_object_pattern).

An example implementation looks like this.

```php
final class CustomerXmlGeneratorStrategy implements XmlGeneratorStrategyInterface
{
    public function supports(TransferableInterface $entity): bool
    {
        return $entity instanceof Customer::class;
    }

    public function generator(TransferableInterface $entity): Xml
    {
        // You can inject other dependencies here to build a more advanced xml file
        return new Xml('filepath.xml', '<?xml version="1.0" encoding="UTF-8"?>');
    }

}
```

## Using this with Symfony
Last we had to inject classes that implement the `XmlGeneratorStrategyInterface` into the
`XmlGenerator` class. Since we used Symfony in this project, we can easily do this in the
service container by adding this to our `services.yml`

```yml
services:
    _instanceof:
        XmlGeneratorStrategyInterface:
            tags: ['xml.generators']

    XmlGenerator:
        class: XmlGenerator
        arguments:
            $generators: !tagged_iterator xml.generators
```

What this does is it will tag all classes which implement `XmlGeneratorStrategyInterface` with the
tag `xml.generators`. When we inject `XmlGenerator` in a class the service container will inject
all the implementation of `XmlGeneratorStrategyInterface` into the `$generators` property.

The result of this implementation that it's much easier to transfer other entities, because you
only have to implement the `TransferableInterface` on the entity and create a new class which
implements `XmlGeneratorStrategyInterface`.

