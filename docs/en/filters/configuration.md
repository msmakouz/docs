# Filter Objects

The framework component `spiral/filters` provides support for request validation, composite validation, an error message
mapping and locations, etc.

> **Note**
> If you are migrating from Spiral Framework 2.x and you want to continue using the old filters you can use
> [spiral/filters-bridge](https://github.com/spiral/filters-bridge) package.
> Read more about using the package [here](../filters/bridge.md).

## Installation

> **Note**
> The component relies on [Validation](/security/validation.md) library, make sure to read it first.

The component does not require any configuration and can be activated using the
bootloader `Spiral\Bootloader\Security\FiltersBootloader`:

```php
[
    // ...
    Spiral\Bootloader\Security\FiltersBootloader::class
    // ...
]
```

## Input Binding

The filter components operate using the `Spiral\Filter\InputInterface` as a primary data source:

```php
interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

By default, this interface is bound to [InputManager](/http/request-response.md) and allows to access
any request's attribute using a **source** and **origin** pair with dot-notation support. For example:

```php
namespace App\Controller;

use Spiral\Filters\InputInterface;

class HomeController
{
    public function index(InputInterface $input): void
    {
        dump($input->getValue('query', 'abc')); // ?abc=1

        // dot notation
        dump($input->getValue('query', 'a.b.c')); // ?a[b][c]=2

        // same as above
        dump($input->withPrefix('a')->getValue('query', 'b.c')); // ?a[b][c]=2
    }
}
```

Input binding is the primary way of delivering data into the filter object.

## Create Filter

The implementation of the filter object might vary from package to package. The default implementation is provided via  the abstract class
`Spiral\Filters\Model\Filter`. To create a custom filter to validate a query:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Query]
    public string $abc;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'abc' => ['string', 'required']
        ]);
    }
}
```

You can request the Filter as a method injection (it will be automatically bound to the current http input):

```php
namespace App\Controller;

use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter): void
    {     
        dump($filter->abc);
    }
}
```

> **Note**
> Try URL with `?abc=1`. The Filter will automatically pre-validate your request before delivering it to the controller.
