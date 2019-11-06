# Dependency Injection(Внедрение зависимостей)


## Проблема

```php
class Order {
    public $id;
}

class Order_Processing {
    public function create_new_order(): void {
      /* Логика выполнения заказа */
      $this->log('Order created!');
    }
    private function log( string $message ): void
    {
        echo "Save log with message: {$message}" . PHP_EOL;
    }
}

$order_processing = new Order_Processing();
$order_processing->create_new_order();
```

В классе Order_Processing принцип Single Responsibility т.к. кроме обработки заказа у нас происходит еще и логирование.Так же код логирования нельзя переиспользовать. Мы немного подумали и решили, что нужно создать класс логер, тогда его можно будет переиспользовать в других классах.

```php
class Logger {
    public function log( string $message ) {
        echo "Save log with message: {$message}" . PHP_EOL;
    }
}

class Order_Processing {
    public function create_new_order() {
        /* Логика выполнения заказа */
        $logger = new Logger();
        $logger->log('Order created!');
    }
}

$order_processing = new Order_Processing();
$order_processing->create_new_order();
```

Класс Logger теперь можно переиспользовать в других классах. В классе Order_Processing у нас появляется жесткая зависимость (Hard Dependency). Чтобы понять, от чего зависит класс Order_Processing нужно прочитать весь код. Объект класса Logger будет создавать каждый раз при создании заказа.

## В силу входит Dependency Injection

```php
class Order_Processing {
    private $logger;
    public function __construct( Logger $loger ) {
        $this->logger = $logger;
    }
    public function create_new_order() {
        /* Логика выполнения заказа */
        $this->logger->log('Order created!');
    }
}

$logger           = new Logger();
$order_processing = new Order_Processing( $logger );
$order_processing->create_new_order();
```

В целом уже все хорошо, уже применяется принцип Dependency Injection. Но нарушен принцип Dependency Inversion и мы можем вместо класса Logger в конструкторе использовать интерфейс, но мы опустим эту деталь.

## Зависимостей становится больше

```php
class Order_Processing {
    private $logger;
    public function __construct( Logger $loger, Order_Repository $repository, SMS_Notifier $sms_notifier ) {
        $this->logger       = $logger;
        $this->repository   = $repository;
        $this->sms_notifier = $sms_notifier;
    }
    public function create_new_order() {
        /* Логика выполнения заказа */
        $this->logger->log('Order created!');
    }
}

$repository       = new Order_Repository();
$sms_notifier     = new SMS_Notifier();
$logger           = new Logger();
$order_processing = new Order_Processing( $logger, $repository, $sms_notifier );
$order_processing->create_new_order();
```

Конструктор разрастается и теперь в любом месте, где мы используем Order_Processing нам приходится тащить за этим классом все его зависимости. При добавлении и изменении зависимостей необходимо менять код везде(можно решить с помощью паттерна Factory, но это решит проблему не на долго).

## Применение Inversion Of Control

Inversion Of Control - говорит о том, что зависимостями должен заниматься фреймворк, а не разработчики.

Есть несколько подходов для реализиции Ioc. Рассмотрим самые известные, это:
 * Service Locator - плохой вариант
 * Dependency Injection - хороший вариант

### Service Locator

Подключим библиотеку `Pimple` и заменим весь конструктор на ServiceLocator. Создаем конфиг для Service Locator

```php
require __DIR__ . '/../vendor/autoload.php';
$container = new \Pimple\Container();
$container['logger'] = function ( $container ) {
    return new Logger();
};
$service_locator = new \Pimple\Psr11\ServiceLocator( $container, ['logger'] );
```

и изменяем наш класс Order_Processing:

```php
class Order_Processing {
    private $service_locator;
    public function __construct(\Pimple\Psr11\ServiceLocator $service_locator) {
        $this->service_locator = $service_locator;
    }
    public function create_new_order() {
        // Здесь логика создания заказа
        $this->service_locator->get('logger')->log('Order created!');
    }
}
```

 Но почему же Сервис Локатор считается антипаттерном?
 
 Из минусов:
 - Зависимости указаны неявно. Смотря на код Order_Processing мы не понимаем какие у него есть зависимости.
 - Зависимости настроены неявно. Мы так же не понимаем что находится за идентификатором 'logger'
 - Сложно рефакторить. Попробуйте найти в большом проекте все вызовы метода log которые относятся к классу Logger.
 - Если используется он вот так `$this->serviceLocator->get('logger')->log('Order created!')`. Здесь разве что поиск в помощь. Но представте что вам нужно отрефакторить метод который называется например "save" и он у множества классов которые настроенны в Локаторе Служб.
- Здесь нарушен принцип Dependency Inversion. Зависимости создаются внтури самого класса.

### Что делать, если в проекте уже есть Service Locator и избавится от него не получится

Поправим немного конфиг:

```php
require __DIR__ . '/../vendor/autoload.php';
$container = new \Pimple\Container();
$container['logger'] = function ( $container ) {
    return new Logger();
};
$container['order.processing'] = function ( $container ) {
    return new Order_Processing( $container['logger'] );
};
$service_locator = new \Pimple\Psr11\ServiceLocator( $container, ['logger', 'order.processing'] );
```

И так же поправим наш класс Order_Processing и его вызов.

```php
class Order_Processing {
    private $logger;
    public function __construct(Logger $logger) {
        $this->logger = $logger;
    }
    public function create_new_order() {
        // Здесь логика создания заказа
        $this->logger->log('Order created!');
    }
}


$order_processing = $serviceLocator->get('order.processing');
$order_processing->create_new_order();
```

Таким образом мы решаем ряд следующих проблем:
- Теперь в классе OrderProcessing зависимости указаны явно. Мы видим от каких копоненов зависит наш класс.
- Стало полегче рефакторить. Мы слегкость сможем найти все использования метода log конкретного класса.
- Dependecy Inversion не нарушен (за исключением того что зависимость построена на реализацию, а не на абстракцию)
Из минусов:
- При разработке веб-приложения нам всеравно прийдется использовать локатор в контроллерах, консольных команда. И зависимости будут по прежнему указаны не явно.
- При добавлении или удалении зависимости, нужно править конфиги.

### Dependency Injection Container

Лучший вариант на данный момент это использование Dependency Injection Container(далее DIC). Для этого нужно поставить http://php-di.org/ и создать DIC:

```php
require __DIR__ . '/../vendor/autoload.php';
$container = new \DI\Container();
```

и теперь класс Order_Processing и его вызов выглядят так:

```php
class Order_Processing {
    private $logger;
    public function __construct( Logger $logger ) {
        $this->logger = $logger;
    }
    public function create_new_order() {
        // Здесь логика создания заказа
        $this->logger->log( 'Order created!' );
    }
}

$order_processing = $container->get(Order_Processing::class);
$order_processing->create_new_order();
```

Таким образом первый раз будет создан класс Order_Processing и Logger т.к. он требуется в конструкторе Order_Processing и оба класса будут записаны в DIC. При каждом вызове сначала проверяется наличие объекта и всех его зависистей в DIC и при их отстутствии они создаются.
