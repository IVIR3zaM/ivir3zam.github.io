---
layout: post
title:  "Using Composite Design Pattern in ORM Models in PHP 7 (Part 1: implementing Composite Design Pattern alone)"
date:   2017-05-10 08:01:23 +0430
categories: php7
---

As may you now about the best practice of designing a recursive ordering system, the best technique is using a composite design pattern. So for getting a better insight about the project area lets talk about a sample project. Imagine a **coffee shop** that serves many types of coffees and many types of add-ins for coffees with specific prices. You want an ordering system that can handle the total amount of order and a list of order items. The best practice is using a unified abstract class. Lets call it **Orderable** (maybe you can find a better name for that but this is only an example). The implementation is:

{% highlight php %}
<?php
namespace CoffeeShop;
abstract class Orderable implements Countable
{
    abstract public function getTitle() : string;
    abstract public function getAmount() : float;
    abstract public function count() : int;
    /**
     * @return Orderable[]
     */
    abstract public function getItems() : array;
}
{% endhighlight %}

so now we need the abstraction of attachable items. Coffee is attachable and can hold some add-ins but add-in is not attachable so we need this **OrderableAttachable** abstract layer:

{% highlight php %}
<?php
namespace CoffeeShop;
abstract class OrderableAttachable extends Orderable
{
    /**
     * @var Orderable[]
     */
    protected $items = [];
    public function addItem(Orderable $item)
    {
        $this->items[] = $item;
    }
    public function removeItem(Orderable $item)
    {
        foreach ($this->items as $index => $object) {
            if ($object === $item) {
                unset($this->items[$index]);
                break;
            }
        }
    }
    public function getItems() : array
    {
        return $this->items;
    }
    public function count() : int
    {
        return count($this->items);
    }
    public function getAmount() : float
    {
        $amount = 0;
        foreach ($this->getItems() as $item) {
            $amount += $item->getAmount();
        }
        return $amount;
    }
}
{% endhighlight %}


as you can see we implemented the object stack that is useful for our order class and coffee abstract layer. This is for following DRY. No we need another abstraction of any items in the order that must implement the title and price of each item. This abstraction will use in both coffee and add-in abstraction. But there is a trick. PHP don’t support multi extends so our coffee class can’t extend the **OrderableAttachable** and this new OrderableItem. So we must to use this new class as a **trait**.


{% highlight php %}
<?php
namespace CoffeeShop;
trait OrderableItem
{
    /**
     * @var string The name of Item
     */
    protected $title = '';
    /**
     * @var float The price of item
     */
    protected $price = 0;
    public function setTitle(string $title) : self
    {
        $this->title = $title;
        return $this;
    }
    public function setPrice(float $price) : self
    {
        $this->price = $price;
        return $this;
    }
    public function getPrice() : float
    {
        return $this->price;
    }
}
{% endhighlight %}


now we have all the main abstraction layers. Lets make a implementation for **coffee** and **add-in**.

{% highlight php %}
<?php
namespace CoffeeShop;
class Coffee extends OrderableAttachable
{
    use OrderableItem;
    public function getTitle() : string
    {
        $titles = [];
        foreach ($this->getItems() as $item) {
            $titles[] = $item->getTitle();
        }
        return $this->title . (!empty($titles) ? ' with ' . implode(' and ', $titles) : '');
    }
    public function count() : int
    {
        return parent::count() + 1;
    }
    public function getAmount() : float
    {
        return $this->getPrice() + parent::getAmount();
    }
}
{% endhighlight %}


{% highlight php %}
<?php
namespace CoffeeShop;
class Addin extends Orderable
{
    use OrderableItem;
    public function getTitle() : string
    {
        return $this->title;
    }
    public function getAmount() : float
    {
        return $this->getPrice();
    }
    public function count() : int
    {
        return 1;
    }
    public function getItems() : array
    {
        return [];
    }
}
{% endhighlight %}


now we need the final class for **order** itself.

{% highlight php %}
<?php
namespace CoffeeShop;
class Order extends OrderableAttachable
{
    public function getTitle() : string
    {
        $titles = [];
        foreach ($this->getItems() as $item) {
            $titles[] = $item->getTitle() . ' : $' .$item->getAmount();
        }
        return 'Order of ' . PHP_EOL . '  - ' . (!empty($titles) ? implode(PHP_EOL . '  - ', $titles) : '');
    }
    public function count() : int
    {
        $count = 0;
        foreach ($this->getItems() as $item)
        {
            $count += $item->count();
        }
        return $count;
    }
}
{% endhighlight %}


so now we must to test it before moving forward and change our system to work in and ORM system. So lets make some concrete coffees and add-ins.

{% highlight php %}
<?php
namespace CoffeeShop;
class TurkishCoffee extends Coffee
{
    public function __construct()
    {
        $this->title = 'Turkish Coffee';
        $this->price = 5;
    }
}
class FranceCoffee extends Coffee
{
    public function __construct()
    {
        $this->title = 'France Coffee';
        $this->price = 7;
    }
}
class Sugar extends Addin
{
    public function __construct()
    {
        $this->title = 'Sugar';
        $this->price = 0.2;
    }
}
class Milk extends Addin
{
    public function __construct()
    {
        $this->title = 'Milk';
        $this->price = 0.7;
    }
}
{% endhighlight %}


and lets test it like this:

{% highlight php %}
<?php
namespace CoffeeShop;
$order = new Order();
$turkish = new TurkishCoffee();
$turkish->addItem(new Sugar());
$turkish->addItem(new Milk());
$order->addItem($turkish);
$france = new FranceCoffee();
$france->addItem(new Milk());
$order->addItem($france);
echo $order->getTitle() . PHP_EOL;
echo 'Total items: '.$order->count() . PHP_EOL;
echo 'Total amount: $'.$order->getAmount() . PHP_EOL;
{% endhighlight %}

the output will look likes this:

**
Order of
  - Turkish Coffee with Sugar and Milk : $5.9
  - France Coffee with Milk : $7.7
Total items: 5
Total amount: $13.6
**

Awesome! We use the power of composite design pattern in a coffee shop ordering system with php 7. but this is not enough. You can’t use this implementation in an ORM based system. Because ORMs have a Model Abstraction Layer that must be extends by each entity. And remember we are extending other class in our all three concrete classes: Coffee, Addin and Order. If you want to now the solution please wait a bit for me to write the **Part #2: Merge Composite Design Pattern with ORM Abstract Layer**