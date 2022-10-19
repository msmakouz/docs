# Application Memory
Spiral defines interface `MemoryInterface` to application shared memory. Default implementation use generated php files and relies on OpCache.

> See [runtime directory](/application/directories.md).

The general idea of memory is to speed up an application by caching executing of some functionality. 
The memory component is used to store the configuration cache, ORM and ODM schemas, loadmap (see Loader component), console commands and tokenizer cache; it can also be used to cache compiled routes and etc.
 
 > Application memory must never be used to store users data.
  

```php
interface MemoryInterface
{
    /**
     * Read data from long memory cache. Must return exacts same value as saved or null.
     *
     * @param string $section Non case sensitive.
     *
     * @return string|array|null
     */
    public function loadData(string $section);

    /**
     * Put data to long memory cache. No inner references or closures are allowed.
     *
     * @param string       $section Non case sensitive.
     * @param string|array $data
     */
    public function saveData(string $section, $data);
}
```

## Practical Example
Let's view an example of a service used to index available classes and generate set of operations based on it:

```php
abstract class Operation 
{
    /**
     * Execute some operation.
     */
    abstract public function perform($request);
}

class OperationService
{
    /**
     * List of operation associated with thier class.
     */
    protected $operations = [];

    /**
     * OperationService constructor.
     *
     * @param MemoryInterface  $memory
     * @param ClassesInterface $classes
     */
    public function __construct(MemoryInterface $memory, ClassesInterface $classes)
    {
        $this->operations = $memory->loadData('operations');

        if (is_null($this->operations)) {
            //This is slow operation
            $this->operations = $this->locateOperations($classes);
        }

        //We now can store data into long time memory
        $memory->saveData('operations', $this->operations);
    }

    /**
     * @param string $operation
     * @param mixed  $request
     */
    public function run($operation, $request)
    {
        //Perform operation based on $operations property
    }

    /**
     * @param ClassesInterface $locator
     * @return array
     */
    protected function locateOperations(ClassesInterface $classes)
    {
        //Generate list of available operations via scanning every available class
    }
}
```

> You can store any `var_export`able value in memory.

`MemoryInterface` is implemented in spiral bundle by `Memory` class, which lets you access it's functions using shortcut 'memory'.

```php
public function doSomething()
{
    dump($this->memory->loadData('something'));
}
```

You can implement your own version of `MemoryInterface` using APC, XCache or even Memcache. 

## Memory Rules
Before you will embed `MemoryInterface` into your component or service:
* Do not store any data related to user request, action or information. Memory is only for logic caching
* Assume memory can disappear at any moment