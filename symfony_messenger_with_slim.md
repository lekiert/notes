# How to use Symfony Messenger component with Slim Framework

## Goal

To be able to use symfony queue consumers (workers) as standalone components (without having to use the whole Symfony 4 framework).

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





## Links

1. https://symfony.com/doc/current/messenger.html
