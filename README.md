# JphooiveldEventSauceBundle

## Info

This bundle integrates EventSauce and it's doctrine message repository into your symfony application.

It's strongly advised to know how EventSauce works or read up on the offical website (https://eventsauce.io) 
before you install this bundle.

## Installation

Use composer to install the bundle into your symfony project.

    $ composer require jphooiveld/eventsauce-bundle
    
If you want to use symfony messenger as dispatcher instead of the default dispatcher you must install 
the composer package **symfony/messenger**.

    $ composer require symfony/messenger


If you want to use the console command to generate the database table for you, you must install the 
composer package **symfony/console**.    

    $ composer require symfony/console

Normally symfony should be able to register the bundle automatically. If not you will need to add it 
to the bundles.php in your config directory

```php
<?php
return [
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],
    Symfony\Bundle\SecurityBundle\SecurityBundle::class => ['all' => true],
    Jphooiveld\Bundle\EventSauceBundle\JphooiveldEventSauceBundle::class => ['all' => true],
];
```

## Configuration

```yaml
jphooiveld_event_sauce:
    time_of_recording:
        # The timezone to use for recorded messages, defaults to PHP ini setting.
        timezone: UTC
    messenger:
        # Use symfony messenger instead of EventSauce's default dispatcher (needs package symfony/messenger installed).
        enabled: false
        # The name of the messenger command bus service to use.
        service_bus: messenger.bus.events
    message_repository:
        # Override if you don't want to use the default doctrine message repository but your own implementation. 
        # If doctrine is enabled this will be ignored.
        service: jphooiveld_eventsauce.message_repository.doctrine
        doctrine:
            # Use doctrine as default message repository. If you turn this off you must provide your own service.
            enabled: true
            # Service that implements a doctrine connection. We assume doctrine bundle default here.
            connection: doctrine.dbal.default_connection
            # The table name in the database to store the messages.
            table: event
        # Autoconfigure aggregate roots to use the default repository implemtations as created by the bundle. 
        # Turn this off if you want to create your own implementation.
        autoconfigure_aggregates: true
            
```

## Configuration Detail

### Messenger

By default the synchronous message dispatcher from EventSauce is used. If you enable symfony messenger and 
provide the service id of the event bus all messages will be handled by the messenger component. 

### Message repository

The doctrine message component is automatically installed as dependency. Otherwise there would be a non
functioning application. But if you want to install another type of repository or create your own it's 
easy to reference it in the service parameter. If doctrine is enabled (and it's by default) this will be
ignored.

Autoconfiguring aggregates is an easy way to bind your aggregates to a the doctrine message repository with
default EventSauce services. A compiler pass created the services automatically internally so you can bind 
them to parameters in your services.yaml.

Let's for example assume you have an aggregate root called Order
 
```php
<?php

namespace App\Domain;

use EventSauce\EventSourcing\AggregateRoot;

class Order implements AggregateRoot
{    
}
```

A compiler pass will automatically create a service called **jphooiveld_eventsauce.aggregate_repository.order**. 
After that you can bind it to a default parameter in your services yaml so you can inject it into your own services.

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        bind:
            EventSauce\EventSourcing\AggregateRootRepository $orderRepository: '@jphooiveld_eventsauce.aggregate_repository.order'
```

Let's say we have a command handles that is responsible for persisting the order aggregate. We can now inject the 
order repository automatically for further use.

```php
<?php

namespace App\CommandHandler;

use EventSauce\EventSourcing\AggregateRootRepository;

class AddOrderHandler
{
    private $orderRepository;

    public function __construct(AggregateRootRepository $orderRepository)
    {
        $this->orderRepository = $orderRepository;
    }
}
```

If you don't want to autoconfigure your repositories you can turn this off and take care of it yourself.

## Other auto configuration

EventSauce has a number of interfaces that are auto configured when this bundle is installed to make your life a lot easier.

### Consumers

Consumers are responsinble for handling events from the aggregates. The message dispatcher is responsible for delegating the
events to the consumers. Every class that you create and implements **EventSauce\EventSourcing\Consumer** will automatically 
receive events from the message dispatcher. However if you intent to use Symfony messenger you must implement the __invoke
method. To overcome this limitation you can use the **ConsumableTrait** provided in the bundle. This will make sure it will work
with the default message dispatcher from EventSauce as wel as the symfony messenger component. The trait will also let consumer's
handler the events the same way as the aggregate repository does. It will look for methods that start with apply and after that the
name of the event. Let's say we have a **OrderCreated** class which is an event that indicated ther order was created. Now we 
want to sent an email notifcation. For example:

```php
<?php

namespace App\Service;

use App\Event\OrderCreated;
use EventSauce\EventSourcing\Consumer;
use EventSauce\EventSourcing\Message;
use Jphooiveld\Bundle\EventSauceBundle\ConsumableTrait;

class SendMailNotification implements Consumer
{
        use ConsumableTrait;
        
        protected function applyOrderCreated(OrderCreated $event, Message $message)
        {
            // send mail
        }
}

```



### Message decorators

Message decorators allow you to add extra headers to a message. Every class that you create and implements 
**EventSauce\EventSourcing\MessageDecorator** will automatically add the headers that you define to the persisted message.

### Upcasting

Upcasters can transform messages in case events change. Every class that you create and implements 
**EventSauce\EventSourcing\Upcasting\DelegatableUpcaster** will automatically be used.

## Overriding default services

All services used in the bundles are actually aliases to real implementations. If you want to override services all you 
have to do is create your own services and override the aliases in the build method in your Kernel. 

Beware that if you start to override services stuff can and will break because auto configuration uses a lot of the default
implementations.

| Alias                                     |Interface                                                 |
|-------------------------------------------|----------------------------------------------------------|
| jphooiveld_eventsauce.message_dispatcher  | EventSauce\EventSourcing\MessageDispatcher               |
| jphooiveld_eventsauce.clock               | EventSauce\EventSourcing\Time\Clock                      |
| jphooiveld_eventsauce.message_decorator   | EventSauce\EventSourcing\MessageDecorator                |
| jphooiveld_eventsauce.inflector           | EventSauce\EventSourcing\ClassNameInflector              |
| jphooiveld_eventsauce.event_serializer    | EventSauce\EventSourcing\Serialization\EventSerializer   |
| jphooiveld_eventsauce.message_serializer  | EventSauce\EventSourcing\Serialization\MessageSerializer |

