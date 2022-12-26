# Events

The `Events` component provides tools that allow your application components to communicate with each other
by dispatching events and listening to them. The component makes it easy to register event listeners in your application. 
It also provides an easy way to integrate `PSR-14` compatible `EventDispatcher`.

## Installation

To enable the component, you need to add `Spiral\Events\Bootloader\EventsBootloader` to the bootloaders
list, which is located in the class of your application.

```php
namespace App;

use Spiral\Events\Bootloader\EventsBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        EventsBootloader::class,
        // ...
    ];
}
```

This bootloader registers the implementation of the `Spiral\Events\ListenerFactoryInterface` and 
`Spiral\Events\ListenerProcessorRegistry` in the application container. It also registers event listeners 
in the application. We'll describe below how to add them.

After that, let's install `PSR-14` implementation of a compatible `EventDispatcher`. We provide a 
`spiral-packages/league-event` package that integrates a PSR-14 compatible `league/event` package into 
an application based on the Spiral Framework.

```php
composer require spiral-packages/league-event
```

After the package is installed, you need to register a bootloader from the package.

```php
namespace App;

use Spiral\League\Event\Bootloader\EventBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        EventBootloader::class,
        // ...
    ];
}
```

## Configuration

The configuration file for `Events` component should be located at `app/config/events.php`. Within this file, you may
configure the event `listeners` and `processors`.

The `listeners` parameter represented an `array`, the key is the `fully qualified name of the event class`,
and the value must be an `array` that contains the `fully qualified name of the event listener class`,
or `Spiral\Events\Config\EventListener` instance that allows you to configure additional listener options.

The `processors` parameter represented an `array` and contains the `full name of the processor class`. 

> **Note**
> More detailed information about `processors` will be provided below. This section only describes the configuration.

For example, a configuration file might look like this:

```php
use App\Listener\RouteListener;
use Spiral\Events\Config\EventListener;
use Spiral\Events\Processor\AttributeProcessor;
use Spiral\Events\Processor\ConfigProcessor;
use Spiral\Router\Event\RouteMatched;

return [
    /**
     * -------------------------------------------------------------------------
     *  Listeners
     * -------------------------------------------------------------------------
     * 
     * The list of event listeners to be registered in the application.
     */
    'listeners' => [
        // without any options
        RouteMatched::class => [
            RouteListener::class,
        ],

        // OR

        // with additional options
        RouteMatched::class => [
            new EventListener(
                listener: RouteListener::class,
                method: 'onRouteMatched',
                priority: 1
            ),
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Processors
     * -------------------------------------------------------------------------
     * 
     * Array of all available processors.  
     */
    'processors' => [
        AttributeProcessor::class,
        ConfigProcessor::class,
    ],
];
```

## Usage

### Event

An event can be represented by a simple class.

```php
namespace App\Event;

use App\Database\User;

final class UserWasCreated
{
    public function __construct(
        public readonly User $user
    ) {
    }
}
```

Dispatching an event.

```php
$this->container->get(EventDispatcherInterface::class)->dispatch(new UserWasCreated($user));
```

### Listener

A listener can be represented by a simple class with a method that will be called to handle the event. 
The method name can be configured in the Listener attribute parameter or in the configuration file (`__invoke` by default).

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener]
class UserWasCreatedListener
{
    public function __invoke(UserWasCreated $event): void
    {
        // ...
    }
}
```

Using the attribute, you can configure additional parameters.

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener(event: UserWasCreated::class, method: 'onUserWasCreated', priority: 1)]
class UserWasCreatedListener
{
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

The attribute can be used `directly on the method`, then the method name can be omitted.

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

class UserWasCreatedListener
{
    #[Listener(event: UserWasCreated::class, priority: 1)]
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

The `Spiral\Events\Attribute\Listener` attribute is needed to automatically register an event listener in the application. 
If you prefer to register listeners via `config file`, you can remove this attribute 
and `Spiral\Events\Processor\AttributeProcessor` from the config file.

## Processors

Processors provide an ability to register event listeners in the application.
Two processors are available by default.

- `Spiral\Events\Processor\ConfigProcessor` registers event listeners from the `configuration file`.
- `Spiral\Events\Processor\AttributeProcessor` registers event listeners with the `Spiral\Events\Attribute\Listener` attribute.

> **Note**
> In the config file `app/config/events.php`, you can change this and use only one of them or add your own processor.

### Creating a Processor

A processor is a class that must implement the `Spiral\Events\Processor\ProcessorInterface` and implement 
the `process` method.

```php
namespace App\Processor;

use Spiral\Events\ListenerFactoryInterface;
use Spiral\Events\ListenerRegistryInterface;
use Spiral\Events\Processor\AbstractProcessor;

final class MyCustomProcessor extends AbstractProcessor
{
    public function __construct(
        private readonly ListenerFactoryInterface $factory,
        private readonly ?ListenerRegistryInterface $registry = null,
    ) {
    }

    public function process(): void
    {
        // If the EventDispatcher implementation is not registered in the application,
        // this interface implementation will not be registered and the processor will stop working.
        if ($this->registry === null) {
            return;
        }

        // Using the ListenerRegistryInterface implementation, we can register event listeners.
        $this->registry->addListener(
            event: $event,
            listener: $this->factory->create($listener, $method),
            priority: $priority
        );
    }
}
```

Implementation of the `Spiral\Events\ListenerFactoryInterface` allows you to create a listener instance from 
a fully qualified class name and a method name, adding all the necessary dependencies to the constructor.

After that, we need to register the processor in the configuration file.

```php
// file app/config/events.php    
use App\Processor\MyCustomProcessor;

return [
    // ...
    'processors' => [
        MyCustomProcessor::class,
        // ...
    ],
    // ...
];
```

## Creating an EventDispatcher

As an implementation of EventDispatcher, we considered the package 
[The League Event bridge for Spiral Framework](https://github.com/spiral-packages/league-event). 
You can create your own implementation using `PSR-14 EventDispatcher`.

### ListenerRegistry

Let's create a class that will implement the `Spiral\Events\ListenerRegistryInterface` and provides an ability 
to register `event listeners`. The `addListener` method from this class is called by `processors`, 
passing the `event`, `event listener`, and `priority` as parameters.

```php
namespace App\EventDispatcher;

use Spiral\Events\ListenerRegistryInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

final class ListenerRegistry implements ListenerRegistryInterface, ListenerProviderInterface
{
    public function addListener(string $event, callable $listener, int $priority = 0): void
    {
        // ...
    }
    
    public function getListenersForEvent(object $event): iterable
    {
        // ...
    }
}
```

### EventDispatcher

Create an implementation of `Psr\EventDispatcher\EventDispatcherInterface`.

```php
namespace App\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;

final class EventDispatcher implements EventDispatcherInterface
{
    public function dispatch(object $event)
    {
        // ...
    }
}
```

### Bootloader

After that, we need to create a bootloader that will register the implementation of interfaces in the application container.

```php
namespace App\EventDispatcher\Bootloader;

use App\EventDispatcher\ListenerRegistry;
use App\EventDispatcher\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\EventDispatcher\ListenerProviderInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Events\Bootloader\EventsBootloader;
use Spiral\Events\ListenerRegistryInterface;

final class EventBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        EventsBootloader::class
    ];

    protected const SINGLETONS = [
        ListenerRegistryInterface::class => ListenerRegistry::class,
        ListenerRegistry::class => ListenerRegistry::class,
        ListenerProviderInterface::class => ListenerRegistry::class,
        EventDispatcherInterface::class => EventDispatcher::class,
        EventDispatcher::class => EventDispatcherInterface::class
    ];
}
```

> **Note**
> Don't forget to register the bootloader in the application.
