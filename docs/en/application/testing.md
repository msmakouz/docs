# Application Tests
Testing your code is essential in order to develop reliable application. Besides classic unit tests for your services and/or controllers you are able to test your application as whole using isolated environment.

## BaseTest
Locate this file in `tests` directory of your application, by default it configured to work with default `.env`. Test built in a way so your application will be constructed and destroyed completely for each test case:

```php
abstract class BaseTest extends TestCase
{
    /**
     * @var \App
     */
    protected $app;

    public function setUp()
    {
        $root = dirname(__DIR__) . '/';

        $app = $this->app = \App::init([
            'root'        => $root,
            'libraries'   => $root . 'vendor/',
            'application' => $root . 'app/',
        ], null, null, false);

        //Monolog love to write to CLI when no handler is set
        $this->app->logs->debugHandler(new NullHandler());
    }

    /**
     * This method performs full destroy of spiral environment.
     */
    public function tearDown()
    {
        if (class_exists('Mockery')) {
            //Mockery defines it's own static container to be destructed
            \Mockery::close();
        }

        //Forcing destruction
        $this->app = null;

        gc_collect_cycles();
        clearstatcache();
    }
}
```

You can pass custom instance of `EnvironmentInterface` in order to define separate database credentials and values. 

## Testing Http Methods
Use `HttpTest` with ability to emulate user requests, cookies and sessions:

```php
class IndexTest extends HttpTest
{
    public function testSeeWelcome()
    {
        $response = $this->get('/');

        $this->assertSame(200, $response->getStatusCode());
        $this->assertContains('Welcome to Spiral Framework', (string)$response->getBody());
        $this->assertContains('welcome.dark.php', (string)$response->getBody());
    }
}
```

## Mock Services
To manipulate with application services while testing simply bind any specific interface, binding or class to your mock object:

```php
$this->app->container->bind(MyService::class, $myServiceMock);

//...
```

## Mock Databases
You can mock database to test your application code in more realisting environment. The nesessary change is to point your target database to another driver (usually runtime SQLite). Be aware, that in this case your tests might take much longer to execute.

```php
public function setUp()
    {
        $root = dirname(__DIR__) . '/';

        $app = $this->app = \App::init([
            'root'        => $root,
            'libraries'   => $root . 'vendor/',
            'application' => $root . 'app/',
        ], null, null, false);

        //Replace default database
        $this->app->dbal->addDatabase(new Database(
            $this->app->dbal->driver('runtime'),
            'default',
            'tests_'
        ));

        //Monolog love to write to CLI when no handler is set
        $this->app->logs->debugHandler(new NullHandler());
   
        //Schemas and seeds
        $this->app->console->run('orm:schema', ['-alter'=>true]);
        $this->app->console->run('app:seed');
}
```

You can destroy or empty database in tearDown method.

## Test Bootloaders
In order to replace application bootloaders list in your tests create App child with desired value:

```php
class TestApplication extends App
{
    const LOAD = []; //Bootloads nothing, manual initialization is required
}
```
