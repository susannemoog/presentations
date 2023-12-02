## symfony/messenger standalone
#### Susi Moog @ SymfonyCon Brussels 2023
[@psychomieze@norden.social](https://norden.social/@psychomieze)

---

## Who am I?

- Susi Moog
- PHP developer since ~20 years
- Contributor to TYPO3 since ~15 years
- TYPO3 Association Board member
- Work @ team neusta, Bremen, Germany

---
## Agenda

- The background
- The use case
- Integration challenges
- Step-by-Step integration
- Summary
- End-User Result
- Q&A 

---

## The background

- TYPO3 Open Source Enterprise CMS
- 25 years old
- modernized step-by-step
- not based on Symfony
- using a lot of symfony components

Note: f.e. dependency injection, commands or routing

---

## The use case

- Outgoing and incoming webhooks 
- webhook execution (especially outgoing) asynchronous
- general purpose queue
- Use as little custom code as possible

Note: for easy 3rd party integrations - buzzword: "no-code", (we are also an extension framework that should make it possible for third parties to extend the system)

%% 

## Analysis

- see https://decisions.typo3.org/t/a-queue-in-core/739 
- Main contender: Enqueue
- Decision for symfony: in line with other components, command bus and queue, messages are full PHP objects

Note: 

---

## Integration Challenges

- Standalone documentation
- Commands rely on symfony event dispatcher
- Integrate into TYPO3 configuration system 

Note: for example TYPO3 already uses doctrine and has a database configured, does not use symfony/config

---

### symfony/messenger


| ![Overview](./assets/overview.svg) | 
|:--:| 
| Taken from the [official docs](https://symfony.com/doc/current/components/messenger.html) |


---

### The Bus

- uses symfony message bus directly
- factory used to wire middlewares configured in TYPO3 to bus
- default implementation: default bus

%% 

### The Bus
DI Config:
```yaml
messenger.bus.default:
  class: Symfony\Component\Messenger\MessageBusInterface
  factory: [ '@TYPO3\CMS\Core\Messenger\BusFactory', 'createBus' ]

Symfony\Component\Messenger\MessageBusInterface:
  alias: messenger.bus.default
```

%%

### The Bus

```php
namespace TYPO3\CMS\Core\Messenger;
use Symfony\Component\Messenger\MessageBus;
use Symfony\Component\Messenger\MessageBusInterface;
final class BusFactory
{
    public function __construct(private readonly array $middlewares) {}
    public function createBus(string $bus = 'default'): MessageBusInterface
    {
        return new MessageBus(
            $this->middlewares[$bus] ?? []
        );
    }
}
```


%%

### The Bus Middlewares

- Configuration and ordering via Symfony DI
- MVP: HandleMessage and SendMessage Middlewares
- MessengerMiddlewarePass as CompilerPass for the DI container

%%

### The Bus Middlewares: DI Config

```yaml
Symfony\Component\Messenger\Middleware\SendMessageMiddleware:
  arguments:
    $sendersLocator: '@Symfony\Component\...\SendersLocatorInterface'
    $eventDispatcher: '@Psr\EventDispatcher\EventDispatcherInterface'
  tags:
    - { name: 'messenger.middleware' }
Symfony\Component\Messenger\Middleware\HandleMessageMiddleware:
  arguments:
    $handlersLocator: '@Symfony\Component\...\HandlersLocatorInterface'
  tags:
    - name: 'messenger.middleware'
      after: 'Symfony\Component\...\SendMessageMiddleware'
```

%%

Excerpt compilerpass:
```php
 public function process(ContainerBuilder $container): void
{
    $busFactory = $container->findDefinition(BusFactory::class);
    $groupedMiddlewares = $this->collectMiddlewares($container);
    $middlewares = [];
    foreach ($groupedMiddlewares as $bus => $unorderedMiddlewares) {
        $middlewares[$bus] = [];
        foreach ($this->orderer->orderByDependencies($unorderedMiddlewares) as $middleware) {
            $middlewares[$bus][] = new Reference($middleware['service']);
        }
    }
    $busFactory->setArgument('$middlewares', array_map(
        fn (array $busMiddlewares): IteratorArgument => new IteratorArgument($busMiddlewares),
        $middlewares
    ));
}
```

%%

### HandlersLocator and SendersLocators 

- middlewares need handlers and senders locators respectively
- also configured via DI: wiring Symfony interfaces to TYPO3 concrete implementation

%%

### DI Config

```yaml
TYPO3\CMS\Core\Messenger\HandlersLocator:
 factory: [ '@TYPO3\CMS\Core\Messenger\HandlersLocatorFactory', 'createHandlersLocator' ]
TYPO3\CMS\Core\Messenger\TransportLocator:
 arguments:
  $sendersLocator: !tagged_locator { tag: 'messenger.sender', index_by: 'identifier' }
Symfony\Component\Messenger\Handler\HandlersLocatorInterface:
 alias: TYPO3\CMS\Core\Messenger\HandlersLocator
Symfony\Component\Messenger\Transport\Sender\SendersLocatorInterface:
 alias: TYPO3\CMS\Core\Messenger\TransportLocator
```

%%

### Configuring Handlers

- tagged in DI configuration
- message handled is inferred
- Compilerpass "MessageHandlerPass" to find handlers
- Future: Use PHP Attributes in addition to DI config

Note: handlers are added to handlersLocator

%%

### DI config 

```yaml
  TYPO3\CMS\Workspaces\MessageHandler\StageChangeNotificationHandler:
    tags:
      - name: 'messenger.message_handler'
```

%%

### Example Handler:

```php

    public function __invoke(StageChangeMessage $message): void
    {
        $this->notificationService->notifyStageChange(
            $message->workspaceRecord,
            ...
        );
    }
```
--- 

## Transports

%%

### Default transport

- By default: synchronous for backwards compatibility
- Configure via DI: tag & name

```yaml
  Symfony\Component\Messenger\Transport\Sync\SyncTransport:
    tags:
      - name: 'messenger.sender'
        identifier: 'default'
        ...
```

%% 

### Doctrine transport

- asynchronous transport
- TYPO3 uses doctrine dbal --> use `symfony/doctrine-messenger` as default async
- wrapper factory bridging the config
- table generation happens via TYPO3 schema management
- Future: allow configuration of connection type

%% 

### Doctrine Transport Factory

```php

public function createTransport(array $options = []): DoctrineTransport
{
  $options['table_name'] ??= 'sys_messenger_messages';
  if ($options['table_name'] === 'sys_messenger_messages') {
      $options['auto_setup'] = false;
  }
  // use native doctrine dbal connection instead of TYPO3s overwritten one
  // as the overwritten querybuilder is not fully compatible with symfony messenger
  $connection = DriverManager::getConnection($GLOBALS['TYPO3_CONF_VARS']['DB']['Connections']['Default']);
  $doctrineTransportConnection = new Connection($options, $connection);
  return new DoctrineTransport($doctrineTransportConnection, $this->serializer);
}
```

%%

### DI Config

```yaml
Symfony\Component\Messenger\Bridge\Doctrine\Transport\DoctrineTransport:
  factory: [ '@TYPO3\CMS\Core\Messenger\DoctrineTransportFactory', 'createTransport' ]
  arguments:
    $options:
      queue_name: 'default'
  tags:
    - name: 'messenger.sender'
      identifier: 'doctrine'
    - name: 'messenger.receiver'
      identifier: 'doctrine'
```

%%

### AMQP transport

- if no config bridge is needed, transports can be configured via DI directly from symfony

```yaml
Symfony\Component\Messenger\Bridge\Amqp\Transport\AmqpTransportFactory:

Symfony\Component\Messenger\Bridge\Amqp\Transport\AmqpTransport:
  factory: [ '@Symfony\Component\Messenger\Bridge\Amqp\Transport\AmqpTransportFactory', 'createTransport' ]
  arguments:
    $dsn: 'amqp://rabbitmq:rabbitmq@rabbitmq:5672/%2f/messages'
    $options:
      ...
  tags:
    - name: 'messenger.sender'
      identifier: 'amqp'
    - name: 'messenger.receiver'
      identifier: 'amqp'
```

---

### The Command

- async messages need to be send
- command in the package cannot be used
- copied command and stripped it down
- using the symfony worker from the package in the command

Note: as event subscribers are not PSR-14 and wired in how the symfony framework handles requests, no retry, no memory limit, only time based stop event

%%

### The Command

```php
...
$worker = new Worker($receivers, $this->messageBus, $this->eventDispatcher, new ConsoleLogger($output), $rateLimiters);
$options = [
    'sleep' => $input->getOption('sleep') * 1000000,
];
$queues = $input->getOption('queues');
if ($queues) {
    $options['queues'] = $queues;
}
$worker->run($options);
return $this->stopWorkerOnTimeLimitListener->hasStopped() ? $exitCodeOnLimit : 0;
...
```

---

## Summary

- 1303 lines of code to integrate the messenger in an MVP into TYPO3
- only possible because of symfony/dependency-injection usage
- main challenges: 
  - custom configuration system 
  - wiring the different components together (understanding them first)
  - command needs to be completely rewritten

%%

### End-user result

- happy TYPO3 extension developers
- Add a PHP class as message object

```php 
<?php
namespace TYPO3\CMS\Queue\Message;
final class DemoMessage
{
    public function __construct(public readonly string $content)
    {
    }
}
```


%% 

### End-user result

-   Inject `\Symfony\Component\Messenger\MessageBusInterface` into your class
-   Call `dispatch()` method with a message as argument

```php
        public function __construct(private readonly MessageBusInterface $bus)
        {
        }
        public function yourMethod(): void
        {
            // ...
            $this->bus->dispatch(new DemoMessage('test'));
            // ...
        }
```

%%

### End-user result

- Write a handler
```php
    namespace TYPO3\CMS\Queue\Handler;
    use TYPO3\CMS\Queue\Message\DemoMessage;
    class DemoHandler
    {
      public function __invoke(DemoMessage $message): void
        {
          // do something with $message
        }
    }
```

%%

### End-user result

- Use a tag to register a handler.

```yaml
    TYPO3\CMS\Queue\Handler\DemoHandler:
      tags:
        - name: 'messenger.message_handler'
    TYPO3\CMS\Queue\Handler\DemoHandler2:
      tags:
        - name: 'messenger.message_handler'
          before: 'TYPO3\CMS\Queue\Handler\DemoHandler'
```

---

## Thank you

--- 

## Resources / Links

- https://symfony.com/doc/current/messenger.html
- https://review.typo3.org/c/Packages/TYPO3.CMS/+/77232
- https://usetypo3.com/messages-in-typo3.html
- https://docs.typo3.org/m/typo3/reference-coreapi/main/en-us/ApiOverview/MessageBus/Index.html
- https://dokuwiki.knallimall.org/typo3-message-bus-rabbitmq 

