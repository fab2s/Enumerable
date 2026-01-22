# Enumerate

[![QA](https://github.com/fab2s/Enumerate/actions/workflows/qa.yml/badge.svg)](https://github.com/fab2s/Enumerate/actions/workflows/qa.yml) [![CI](https://github.com/fab2s/Enumerate/actions/workflows/ci.yml/badge.svg)](https://github.com/fab2s/Enumerate/actions/workflows/ci.yml) [![codecov](https://codecov.io/gh/fab2s/Enumerate/graph/badge.svg?token=M4PZ6Z6MqU)](https://codecov.io/gh/fab2s/Enumerate) [![Packagist Version](https://img.shields.io/packagist/v/fab2s/enumerate)](https://packagist.org/packages/fab2s/Enumerate) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat)](http://makeapullrequest.com) [![License](http://poser.pugx.org/fab2s/Enumerate/license)](https://packagist.org/packages/fab2s/Enumerate)

A lightweight PHP library that makes native enums practical for real-world applications.

## The Problem

PHP 8.1 enums are powerful but have limitations that create friction in production code:

- **No `Stringable` support** - Can't use enums directly in string contexts
- **Strict type restrictions** - Can't safely instantiate from mixed input (HTTP requests, databases)
- **No inheritance** - Can't share behavior across related enums
- **Type mismatch pain** - `IntBackedEnum::tryFrom('1')` throws instead of returning null

## The Solution

Enumerate provides a unified API to work with any enum type consistently:

```php
// Safely instantiate from any input type
$status = Status::tryFromAny($request->get('status')); // string, int, null, or enum instance

// Get a serializable value from any enum
$value = $status->toValue(); // Works for Unit, String, and Int backed enums

// Compare across enum boundaries
$status->compares(LegacyStatus::Active); // true if values match
```

## Installation

```shell
composer require fab2s/enumerate
```

**Requirements:** PHP 8.1+

## Quick Start

Add the trait to your enum:

```php
use fab2s\Enumerate\EnumerateTrait;
use fab2s\Enumerate\EnumerateInterface;

enum Status: string implements EnumerateInterface
{
    use EnumerateTrait;

    case Pending = 'pending';
    case Active  = 'active';
    case Closed  = 'closed';
}
```

Now you can:

```php
// From HTTP request (could be string, int, or null)
$status = Status::tryFromAny($request->input('status'));

// From database or JSON
$status = Status::fromAny($row['status']); // throws if invalid

// To database or JSON
$value = $status->toValue(); // 'active'
json_encode($status);        // '"active"'

// Validation
$status->equals('active', 'pending'); // true if matches any
```

## API Reference

### Instantiation Methods

| Method | Returns | Throws | Description |
|--------|---------|--------|-------------|
| `tryFromAny($value, $strict = true)` | `?static` | - | Safe instantiation from any type |
| `fromAny($value, $strict = true)` | `static` | `InvalidArgumentException` | Strict instantiation |
| `tryFromName($name)` | `?static` | - | Match by case name |
| `fromName($name)` | `static` | `InvalidArgumentException` | Strict match by case name |

### Value Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `toValue()` | `int\|string` | Get the backed value (or case name for UnitEnum) |
| `jsonSerialize()` | `int\|string` | Same as `toValue()`, implements `JsonSerializable` |

### Comparison Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `equals(...$values)` | `bool` | Strict match against provided values |
| `compares(...$values)` | `bool` | Loose match, allows cross-enum comparison by value |

### Type Inspection (Static Helper Only)

| Method | Returns | Description |
|--------|---------|-------------|
| `Enumerate::getType($enum)` | `?string` | Returns `'string'`, `'int'`, or `null` |
| `Enumerate::isStringBacked($enum)` | `bool` | Check if string-backed |
| `Enumerate::isIntBacked($enum)` | `bool` | Check if int-backed |
| `Enumerate::isBacked($enum)` | `bool` | Check if backed (not UnitEnum) |

## Detailed Usage

### Safe Instantiation with `tryFromAny`

The native `tryFrom()` only accepts the exact backing type. `tryFromAny()` handles real-world input:

```php
enum Priority: int implements EnumerateInterface
{
    use EnumerateTrait;

    case Low    = 1;
    case Medium = 2;
    case High   = 3;
}

// All return Priority::Medium
Priority::tryFromAny(2);
Priority::tryFromAny(Priority::Medium);

// All return null (no exception)
Priority::tryFromAny('2');     // string on int-backed
Priority::tryFromAny(null);
Priority::tryFromAny('invalid');
```

### Cross-Enum Matching

When migrating between enum versions or comparing related enums, use non-strict mode:

```php
enum LegacyRole: string {
    case Admin = 'admin';
    case User  = 'user';
}

enum Role: string implements EnumerateInterface
{
    use EnumerateTrait;

    case Admin     = 'admin';
    case User      = 'user';
    case Moderator = 'moderator';
}

// Strict mode (default): only accepts same enum type
Role::tryFromAny(LegacyRole::Admin);              // null
Role::tryFromAny(LegacyRole::Admin, strict: true); // null

// Non-strict mode: matches by value across enums
Role::tryFromAny(LegacyRole::Admin, strict: false); // Role::Admin
```

### Working with UnitEnums

UnitEnums (no backing value) are matched by case name:

```php
enum Color implements EnumerateInterface
{
    use EnumerateTrait;

    case Red;
    case Green;
    case Blue;
}

Color::tryFromAny('Red');   // Color::Red
Color::tryFromName('Red');  // Color::Red
Color::Red->toValue();      // 'Red' (case name as string)
json_encode(Color::Red);    // '"Red"'
```

### Comparison Methods

```php
enum Status: string implements EnumerateInterface
{
    use EnumerateTrait;

    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';
}

$status = Status::Published;

// equals: strict comparison, multiple values allowed
$status->equals('published');                    // true
$status->equals('draft', 'published');           // true (matches any)
$status->equals(Status::Published);              // true

// compares: allows cross-enum matching by value
$status->compares(OtherEnum::Published);         // true if values match
```

### Using the Static Helper

All methods are also available via the `Enumerate` static class, useful when you can't modify the enum:

```php
use fab2s\Enumerate\Enumerate;

// Works with any enum, even without the trait
Enumerate::tryFromAny(SomeEnum::class, $value);
Enumerate::toValue(SomeEnum::Case);
Enumerate::equals(SomeEnum::Case, 'value');
Enumerate::getType(SomeEnum::class); // 'string', 'int', or null
```

## Real-World Examples

### Form Request Handling

```php
class UpdateOrderRequest extends FormRequest
{
    public function validated(): array
    {
        return [
            'status'   => OrderStatus::fromAny($this->input('status')),
            'priority' => Priority::tryFromAny($this->input('priority')) ?? Priority::Medium,
        ];
    }
}
```

### Database Model Casting

```php
class Order extends Model
{
    protected $casts = [
        'status' => OrderStatus::class,
    ];

    public function scopeActive($query)
    {
        return $query->whereIn('status', [
            OrderStatus::Pending->toValue(),
            OrderStatus::Processing->toValue(),
        ]);
    }
}
```

### API Response

```php
return response()->json([
    'status' => $order->status, // Automatically serialized via JsonSerializable
]);
```

## Why This Exists

PHP enums prioritize type safety over practicality. While philosophically sound, this creates boilerplate in real applications where enums must interoperate with HTTP requests (always strings), databases, and JSON APIs.

Enumerate handles this complexity internally so you don't have to write defensive code everywhere enums cross system boundaries.

## Contributing

Contributions are welcome. Please open issues and submit pull requests.

## License

Enumerate is open-source software licensed under the [MIT license](http://opensource.org/licenses/MIT).
