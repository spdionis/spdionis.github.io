---
layout: post
title: "Add additional data to serialized entity using JMSSerializerBundle"
date: 2014-11-27
categories: symfony jmsserializer
---

At my workplace we are building a REST Api using Symfony 2. A few days ago I had a need for which I couldn't find much information online so I decided to write something up on this.

We use the ever-present [JMSSerializer Bundle](https://github.com/schmittjoh/JMSSerializerBundle) to serialize our responses. For one of our entities we needed to add some additional data to the serialized output. This data needed some non-trivial processing so I couldn't use a `@VirtualProperty` annotation.

Of course I needed to hook up a service in the serialization process somewhere somehow. My first thought was to decorate the serializer handler so I tried to create a custom handler but I quickly realized I couldn't easily get the serialized output and just add there my additional data.

It was my first time using the JMS Serializer bundle for a non-trivial task so I was not very familiar with it but after a quick look at the docs I found what i needed: events!

Unfortunately not much is explained about using them in symfony 2 in the [docs](http://jmsyst.com/bundles/JMSSerializerBundle). After a quick search I found somewhere that the tag to register a symfony service as a JMS event subscriber is called `jms_serializer.event_subscriber`. So basically registering an event subscriber service looks like this in yaml:

```yaml
services:
    bundle.my_entity_serializer_subscriber:
        class: Vendor\MyBundle\Serializer\MyEntitySerializerSubscriber
        arguments:
            - @bundle.my_very_needed_service
        tags:
            - { name: jms_serializer.event_subscriber }
```

So finally I had my service that from which I could modify the serialized output. The question now is: how? Let's see that.

As said in the [library docs](http://jmsyst.com/libs/serializer/master/event_system) you have to implement `JMS\Serializer\EventDispatcher\EventSubscriberInterface` in your service. Be careful not to confuse it with Symfony's EventDispatcher component's EventSubscriberInterface and write the correct `use` statement.

To implement the `EventSubscriberInterface` we need to have a `getSubscribedEvents` method that should look like below:

```php
public static function getSubscribedEvents()
{
    return [
        ['event' => 'serializer.post_serialize', 'method' => 'myOnPostSerializeMethod', 'class' => MyEntity::class]
    ];
}
```

In this case we need to subscribe to the `serializer.post_serialize` method. Be careful to the multi-dimensional array structure, that's a good place for a stupid but hard to detect bug to hide.

Now let's look at the `myOnPostSerializeMethod` signature (the method signature can change for other types of events, as written in the docs linked above):

```php
public function myOnPostSerializeMethod(ObjectEvent $event);
```

The catch here that took me a bit of time to figure out is that JMS Serializer uses a `Visitor` object to actually serialize and store the serialized data. For people that have worked with this bundle this maybe seems obvious but for me using the serializer for something non-trivial for the first time this wasn't obvious at all.
  
To change the actual output of the serializer we have to interact with the `Visitor` object. I didn't look that much into it after I found how to solve my need so I don't know all the possibilities this object gives you but here's something similar to what I ended up with:

```php
public function myOnPostSerializeMethod(ObjectEvent $event)
{
    $data = $this->getAdditionalData();
        
    $visitor = $event->getVisitor();
    $visitor->addData('key-of-data', $data);
}
```

Here's the complete class we end up with:

```php
namespace Vendor\MyBundle\Serializer;

use JMS\Serializer\EventDispatcher\EventSubscriberInterface;
use JMS\Serializer\EventDispatcher\ObjectEvent;
use Vendor\MyBundle\Whatever\MyDependency;

class MyEntitySerializerSubscriber implements EventSubscriberInterface
{
    /** @var MyDependency */
    private $dependency;

    /**
     * @var MyDependency $dependency
     */
    public function __construct(MyDependency $dependency)
    {
        $this->dependency = $dependency;
    }
    
    public static function getSubscribedEvents()
    {
        return [
            ['event' => 'serializer.post_serialize', 'method' => 'myOnPostSerializeMethod', 'class' => MyEntity::class]
        ];
    }
    
    public function myOnPostSerializeMethod(ObjectEvent $event)
    {
        $data = $this->getAdditionalData();
            
        $visitor = $event->getVisitor();
        $visitor->addData('key-of-data', $data);
    }
    
    public function getAdditionalData()
    {
        // whatever, use dependency here
        
        return $data;
    }
}
```

In the end it really is just that simple, but it took me a few hours to find all the information I needed to deal with this very simple task.

I hope this is helpful to someone :).