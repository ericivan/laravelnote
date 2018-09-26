#Laravel 5.6 自定义缓存

### 主要是自己想把log存在mongodb里面去，翻了下资料，看看里面怎么写自定义log driver

> config 文件修改，logging.php文件增加如下

```php 
 'custom' => [
            'driver' => 'custom',
            'via' => App\Logging\CreateCustomLogger::class,
            'formatter' => 'default',
        ],
```

> 注意的是，一定要是custom，写成其他不行哦，至于为什么，是因为源码里面适配了一个driver的方法，createCustomDriver(),有兴趣的朋友可以自己去
翻源码看



> 第二，编写Class App\Logging\CreateCustomLogger::class,
```php
namespace App\Logging;

use MongoDB\Client;
use Monolog\Handler\MongoDBHandler;
use Monolog\Logger;
use Monolog\Processor\PsrLogMessageProcessor;

class CreateCustomLogger
{
    /**
     * Create a custom Monolog instance.
     *
     * @param  array  $config
     * @return \Monolog\Logger
     */
    public function __invoke(array $config)
    {
        $host = config('logging.mongodb.host');

        $port = config('logging.mongodb.port');

        $dns = "mongodb://" . $host . ":{$port}";

        $mongoClient = new Client($dns);

        $databasse = 'laravel-log';

        $collection = \carbon()->toDateString() . '-log';

        $mongoHandler=new MongoDBHandler($mongoClient, $databasse, $collection);


        $processor = new PsrLogMessageProcessor();
        return new Logger('cusmongo',[$mongoHandler],[$processor]);
    }
}

```

> 这里需要注意几点。
