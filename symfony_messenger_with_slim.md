# How to use Symfony Messenger component with Slim Framework

## Goal

To be able to use symfony queue consumers (workers) as standalone components (without having to use the whole Symfony 4 framework). The example will show how to set up a simple message passing through AMQP. The assumption is that you already have an up and working Slim app with a standard setup (`slim-skeleton`).

## Set up

The default Slim container is built on top of Pimple, which is very minimalistic and does not support YAML configuration. You will have to define everything in your `dependencies.php` file to make it work.

You will need the following packages (use `composer` to install them):

* `symfony/serializer`
* `symfony/property-access`
* `symfony/messenger`
* `symfony/console`

## Messages and handlers

First you need to define a message class, which will be passed through the message bus and transport of your choice. Message classes are POPOs - the have to be serializable. For example:

`src/Messages/Hello.php`

```php
<?php

namespace App\Messages;

class Hello
{
    private $name;

    public function __construct(string $name)
    {
        $this->name = $name;
    }

    public function getName(): string
    {
        return $this->name;
    }
}
```

Now that you have a message, you must define a handler that will perform some logic:

`src/Handlers/HelloHandler.php`

```php
<?php

namespace App\Handlers;

use App\Messages\Hello;
use Symfony\Component\Console\Output\OutputInterface;

class HelloHandler
{
    private $output;

    public function __construct(OutputInterface $output)
    {
        $this->output = $output;
    }

    public function __invoke(Hello $message)
    {
        // using OutputInterface for fancy coloring
        $this->output->writeln('<hello>' . $message->getName() . '</hello>');
    }
}
```

And that's it.

## Commands

Messages have to be sent somewhere, so handlers can pick them up. We can create a CLI command for sending the `Hello` message:

`src/Commands/SayHelloCommand.php`

```php
<?php

namespace App\Commands;

use App\Messages\Hello;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\MessageBusInterface;

class SayHelloCommand extends Command
{
    protected static $defaultName = 'say-hello';

    private $messageBus;

    public function __construct(MessageBusInterface $messageBus)
    {
        $this->messageBus = $messageBus;
        parent::__construct(); // important!
    }

    protected function configure()
    {
        $this->addArgument('name', InputArgument::REQUIRED);
    }

    protected function execute(InputInterface $input, OutputInterface $output)
    {
        $object = new Hello($input->getArgument('name'));
        $envelope = new Envelope($object);

        $this->messageBus->dispatch($envelope);
    }
}
```

Note that we use the `Envelope` class as a wrapper of the original message. For the detailed explanation what Envelopes (and Stamps) are, please refer to the Messenger documentation.

## CLI tool

Slim by default has no tool for performing CLI commands, so we have to define our own. Here is an example of a `bin/run` file:

`bin/run`

```php
#!/usr/bin/env php
<?php

use App\Commands\SayHelloCommand;
use Symfony\Component\Console\Application;
use Symfony\Component\Messenger\Command\ConsumeMessagesCommand;

require __DIR__ . '/../vendor/autoload.php';

$dotenv = new Dotenv\Dotenv(__DIR__ . '/../', '.env');
$dotenv->load();

// create a Slim app and define all dependencies
$settings = require __DIR__ . '/../src/settings.php';
$app = new \Slim\App($settings);
require __DIR__ . '/../src/dependencies.php';

$cli = new Application;
$cli->add($container->get(ConsumeMessagesCommand::class)); // notice this one
$cli->add($container->get(SayHelloCommand::class));
$cli->run();
```

As you might have noticed, apart from the `SayHelloCommand` there is another one: `ConsumeMessagesCommand`. This is a command that comes out of the box with the Messenger component. This is the worker command that will be responsible for reading messages from queues and delegating them to appropriate handlers.

## Dependency container

The first thing you can define is the `SayHelloCommand`

`src/dependencies.php`

```php
$container[\App\Commands\SayHelloCommand::class] = function ($c) {
    return new \App\Commands\SayHelloCommand(
        $c->get(\Symfony\Component\Messenger\MessageBusInterface::class)
    );
};
```

Now you have to define the `MessageBusInterface`:

`src/dependencies.php`

```php
$container[\Symfony\Component\Messenger\MessageBusInterface::class] = function ($c) {
    // this only for fancy colors for the handler class
    $output = new Symfony\Component\Console\Output\ConsoleOutput();
    $output->getFormatter()->setStyle(
        'hello', new OutputFormatterStyle('green')
    );

    return new \Symfony\Component\Messenger\MessageBus([
        new \Symfony\Component\Messenger\Middleware\SendMessageMiddleware(
            new \Symfony\Component\Messenger\Transport\Sender\SendersLocator([
                \GooGS\Model\Hello::class => ['amqp'],
            ], $c)
        ),
        new HandleMessageMiddleware(new HandlersLocator([
            \GooGS\Model\Hello::class => [new \GooGS\Handlers\HelloHandler($output)],
        ])),
    ]);
};
```

The important part here are these two middlewares.

The `SendMessageMiddleware` defines where messages will go. Without it, the message will try to handle these messages synchronously. If you are interested what is a `Sender`, please refer to the `Transports` part of the `symfony/messenger` documentation.

The `HandleMessageMiddleware` defines handlers for our messages. A message may have multiple handlers. Without it, the worker will be able to read the messages from the queue, but will not know what to do with them.

The next thing you have to configure is the `ConsumeMessagesCommand`:

`src/dependencies.php`

```php
$container[\Symfony\Component\Messenger\Command\ConsumeMessagesCommand::class] = function ($c) {
    // to be able to see the results in the console
    $logger = new \Monolog\Logger('stdout');
    $logger->pushHandler(new StreamHandler('php://stdout'));

    // retry strategy container requires a dedicated instance of a PSR-11 container
    // by default there is only one strategy: MultiplierRetryStrategy
    $retryContainer = new \Slim\Container();
    $retryContainer['amqp'] = new \Symfony\Component\Messenger\Retry\MultiplierRetryStrategy(3);

    return new \Symfony\Component\Messenger\Command\ConsumeMessagesCommand(
        $c->get(\Symfony\Component\Messenger\MessageBusInterface::class),
        $c,
        $logger,
        ['amqp'], // receivers to listen (for more details on Receivers please refer to the symfony/messenger documentation)
        $retryContainer
    );
};
```

Note two things:
* worker depends on `Receiver`s to read messages. A `Sender` + `Receiver` = `Transport`. This is why there are `amqp` thrown all around.
* failed commands can be retried. For now, the only strategy available is `MultiplierRetryStrategy` (please refer to the official documentation for more details how to configure it). The example above shows how to configure messages from the `amqp` transport so they have to be retried 3 times before discarding. You can pass `null` if you don't want your messages to be retried.


`src/dependencies.php`

```php
$container[\Symfony\Component\Messenger\Transport\TransportInterface::class] = function ($c) {
    $connection = \Symfony\Component\Messenger\Transport\AmqpExt\Connection::fromDsn($_ENV['AMQP_DSN']);

    return new \Symfony\Component\Messenger\Transport\AmqpExt\AmqpTransport($connection);
};

$container['amqp'] = function ($c) {
    return $c->get(TransportInterface::class);
};
```

## Test it

With everything set up, first run the worker:

`bin/run messenger:consume`

And then you can send a test message:

`bin/run say-hello John`

In the worker output you should see a result.

## Links

1. https://symfony.com/doc/current/messenger.html
