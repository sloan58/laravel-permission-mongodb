# laravel-permission-mongodb

[![Latest Version on Packagist][ico-version]][link-packagist]
[![Software License][ico-license]](LICENSE.md)
[![Build Status][ico-travis]][link-travis]
[![Code Climate][ico-code-climate]][link-code-climate]
[![StyleCI][ico-styleci]][link-styleci]
[![Dependency Status][ico-dependency]][link-dependency]
[![Total Downloads][ico-downloads]][link-downloads]

This package allows you to manage user permissions and roles in a database.
It is inspired from [laravel-permission][link-laravel-permission]. Same code same every thing but it is compatible with [laravel-mongodb][link-laravel-mongodb]

Once installed you can do stuff like this:

```php
// Adding permissions to a user
$user->givePermissionTo('edit articles');

// Adding permissions via a role
$user->assignRole('writer');

$role->givePermissionTo('edit articles');
```

If you're using multiple guards we've got you covered as well. Every guard will have its own set of permissions and roles that can be assigned to the guard's users. Read about it in the [using multiple guards](#using-multiple-guards) section of the readme.

Because all permissions will be registered on [Laravel's gate](https://laravel.com/docs/5.4/authorization), you can test if a user has a permission with Laravel's default `can` function:

```php
$user->can('edit articles');
```

## Table of contents
* [Installation](#installation)
* [Usage](#usage)
    * [Using "direct" permissions](#using-direct-permissions)
    * [Using permissions via roles](#using-permissions-via-roles)
    * [Using Blade directives](#using-blade-directives)
* [Using multiple guards](#using-multiple-guards)
    * [Using permissions and roles with multiple guards](#using-permissions-and-roles-with-multiple-guards)
    * [Assigning permissions and roles to guard users](#assigning-permissions-and-roles-to-guard-users)
    * [Using blade directives with multiple guards](#using-blade-directives-with-multiple-guards)
* [Using a middleware](#using-a-middleware)
* [Using artisan commands](#using-artisan-commands)
* [Database Seeding](#database-seeding)
* [Extending](#extending)
* [Troubleshooting](#troubleshooting)
    * [Cache](#cache)
* [Need a UI?](#need-a-ui)
* [Change log](#change-log)
* [Testing](#testing)
* [Contributing](#contributing)
* [Security](#security)
* [Credits](#credits)
* [License](#license)

## Installation

This package can be used in Laravel 5.4 or higher.

You can install the package via composer:

``` bash
composer require mostafamaklad/laravel-permission-mongodb
```

Now add the service provider in `config/app.php` file:

```php
'providers' => [
    // ...
    Maklad\Permission\PermissionServiceProvider::class,
];
```

You can publish the config file with:

```bash
php artisan vendor:publish --provider="Maklad\Permission\PermissionServiceProvider" --tag="config"
```

When published, [the `config/permission.php` config file](https://github.com/mostafamaklad/laravel-permission-mongodb/blob/master/config/permission.php) contains:

```php
return [

    'models' => [

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * Moloquent model should be used to retrieve your permissions. Of course, it
         * is often just the "Permission" model but you may use whatever you like.
         *
         * The model you want to use as a Permission model needs to implement the
         * `Maklad\Permission\Contracts\Permission` contract.
         */

        'permission' => Maklad\Permission\Models\Permission::class,

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * Moloquent model should be used to retrieve your roles. Of course, it
         * is often just the "Role" model but you may use whatever you like.
         *
         * The model you want to use as a Role model needs to implement the
         * `Maklad\Permission\Contracts\Role` contract.
         */

        'role' => Maklad\Permission\Models\Role::class,

    ],

    'table_names' => [

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * table should be used to retrieve your roles. We have chosen a basic
         * default value but you may easily change it to any table you like.
         */

        'roles' => 'roles',

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * table should be used to retrieve your permissions. We have chosen a basic
         * default value but you may easily change it to any table you like.
         */

        'permissions' => 'permissions',

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * table should be used to retrieve your models permissions. We have chosen a
         * basic default value but you may easily change it to any table you like.
         */

        'model_has_permissions' => 'model_has_permissions',

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * table should be used to retrieve your models roles. We have chosen a
         * basic default value but you may easily change it to any table you like.
         */

        'model_has_roles' => 'model_has_roles',

        /*
         * When using the "HasRoles" trait from this package, we need to know which
         * table should be used to retrieve your roles permissions. We have chosen a
         * basic default value but you may easily change it to any table you like.
         */

        'role_has_permissions' => 'role_has_permissions',
    ],

    /*
     * By default all permissions will be cached for 24 hours unless a permission or
     * role is updated. Then the cache will be flushed immediately.
     */

    'cache_expiration_time' => 60 * 24,

    /*
     * By default we'll make an entry in the application log when the permissions
     * could not be loaded. Normally this only occurs while installing the packages.
     *
     * If for some reason you want to disable that logging, set this value to false.
     */

    'log_registration_exception' => true,
];
```

## Usage

First add the `Maklad\Permission\Traits\HasRoles` trait to your `User` model(s):

```php
use Illuminate\Auth\Authenticatable;
use Jenssegers\Mongodb\Eloquent\Model as Model;
use Illuminate\Foundation\Auth\Access\Authorizable;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
use Illuminate\Contracts\Auth\Access\Authorizable as AuthorizableContract;
use Maklad\Permission\Traits\HasRoles;

class User extends Model implements AuthenticatableContract, AuthorizableContract
{
    use Authenticatable, Authorizable, HasRoles;

    // ...
}
```

- note that if you need to use `HasRoles` trait with another model ex.`Page` you will also need to add `protected $guard_name = 'web';` as well to that model or you would get an error

```php
use Jenssegers\Mongodb\Eloquent\Model as Model;
use Maklad\Permission\Traits\HasRoles;

class Page extends Model
{
    use HasRoles;

    protected $guard_name = 'web'; // or whatever guard you want to use

    // ...
}
```

This package allows for users to be associated with permissions and roles. Every role is associated with multiple permissions.
A `Role` and a `Permission` are regular Moloquent models. They require a `name` and can be created like this:

```php
use Maklad\Permission\Models\Role;
use Maklad\Permission\Models\Permission;

$role = Role::create(['name' => 'writer']);
$permission = Permission::create(['name' => 'edit articles']);
```

If you're using multiple guards the `guard_name` attribute needs to be set as well. Read about it in the [using multiple guards](#using-multiple-guards) section of the readme.

The `HasRoles` trait adds Moloquent relationships to your models, which can be accessed directly or used as a base query:

```php
// get a list of all permissions directly assigned to the user
$permissions = $user->permissions;

// get all permissions inherited by the user via roles
$permissions = $user->getAllPermissions();

// get a collection of all defined roles
$roles = $user->roles->pluck('name'); // Returns a collection
```

The `HasRoles` trait also adds a scope to your models to scope the query to certain roles:

```php
$users = User::role('writer')->get(); // Returns only users with the role 'writer'
```

The scope can accept a string, a `\Maklad\Permission\Models\Role` object or an `\Illuminate\Support\Collection` object.

### Using "direct" permissions

A permission can be given to any user with the `HasRoles` trait:

```php
$user->givePermissionTo('edit articles');

// You can also give multiple permission at once
$user->givePermissionTo('edit articles', 'delete articles');

// You may also pass an array
$user->givePermissionTo(['edit articles', 'delete articles']);
```

A permission can be revoked from a user:

```php
$user->revokePermissionTo('edit articles');
```

Or revoke & add new permissions in one go:

```php
$user->syncPermissions(['edit articles', 'delete articles']);
```

You can test if a user has a permission:

```php
$user->hasPermissionTo('edit articles');
```

...or if a user has multiple permissions:

```php
$user->hasAnyPermission(['edit articles', 'publish articles', 'unpublish articles']);
```

Saved permissions will be registered with the `Illuminate\Auth\Access\Gate` class for the default guard. So you can
test if a user has a permission with Laravel's default `can` function:

```php
$user->can('edit articles');
```

### Using permissions via roles

A role can be assigned to any user:

```php
$user->assignRole('writer');

// You can also assign multiple roles at once
$user->assignRole('writer', 'admin');
// or as an array
$user->assignRole(['writer', 'admin']);
```

A role can be removed from a user:

```php
$user->removeRole('writer');
```

Roles can also be synced:

```php
// All current roles will be removed from the user and replaced by the array given
$user->syncRoles(['writer', 'admin']);
```

You can determine if a user has a certain role:

```php
$user->hasRole('writer');
```

You can also determine if a user has any of a given list of roles:

```php
$user->hasAnyRole(Role::all());
```

You can also determine if a user has all of a given list of roles:

```php
$user->hasAllRoles(Role::all());
```

The `assignRole`, `hasRole`, `hasAnyRole`, `hasAllRoles`  and `removeRole` functions can accept a
 string, a `\Maklad\Permission\Models\Role` object or an `\Illuminate\Support\Collection` object.

A permission can be given to a role:

```php
$role->givePermissionTo('edit articles');
```

You can determine if a role has a certain permission:

```php
$role->hasPermissionTo('edit articles');
```

A permission can be revoked from a role:

```php
$role->revokePermissionTo('edit articles');
```

The `givePermissionTo` and `revokePermissionTo` functions can accept a
string or a `Maklad\Permission\Models\Permission` object.

Permissions are inherited from roles automatically.
Additionally, individual permissions can be assigned to the user too. 

For instance:

```php
$role = Role::findByName('writer');
$role->givePermissionTo('edit articles');

$user->assignRole('writer');

$user->givePermissionTo('delete articles');
```

In the above example a role is given permission to edit articles and this role is assigned to a user.
Now the user can edit articles and additionally delete articles. The permission of 'delete articles' is the user's direct permission because it is assigned directly to them.
When we call `$user->hasDirectPermission('delete articles')` it returns `true`, but `false` for `$user->hasDirectPermission('edit articles')`.

This method is useful if one builds a form for setting permissions for roles and users in an application and wants to restrict or change inherited permissions of roles of the user, i.e. allowing to change only direct permissions of the user.

You can list all of these permissions:

```php
// Direct permissions
$user->getDirectPermissions() // Or $user->permissions;

// Permissions inherited from the user's roles
$user->getPermissionsViaRoles();

// All permissions which apply on the user (inherited and direct)
$user->getAllPermissions();
```

All these responses are collections of `Maklad\Permission\Models\Permission` objects.

If we follow the previous example, the first response will be a collection with the 'delete article' permission, the
second will be a collection with the 'edit article' permission and the third will contain both.

### Using Blade directives
This package also adds Blade directives to verify whether the currently logged in user has all or any of a given list of roles.

Optionally you can pass in the `guard` that the check will be performed on as a second argument.
#### Blade and Roles
Test for a specific role:
```php
@role('writer')
    I am a writer!
@else
    I am not a writer...
@endrole
```
is the same as
```php
@hasrole('writer')
    I am a writer!
@else
    I am not a writer...
@endhasrole
```
Test for any role in a list:
```php
@hasanyrole(Role::all())
    I have one or more of these roles!
@else
    I have none of these roles...
@endhasanyrole
// or
@hasanyrole('writer|admin')
    I am either a writer or an admin or both!
@else
    I have none of these roles...
@endhasanyrole
```
Test for all roles:
```php
@hasallroles(Role::all())
    I have all of these roles!
@else
    I do not have all of these roles...
@endhasallroles
// or
@hasallroles('writer|admin')
    I am both a writer and an admin!
@else
    I do not have all of these roles...
@endhasallroles
```

#### Blade and Permissions
This package doesn't add any permission-specific Blade directives. Instead, use Laravel's native `@can` directive to check if a user has a certain permission.

```php
@can('edit articles')
  //
@endcan
```
or
```php
@if(auth()->user()->can('edit articles') && $some_other_condition)
  //
@endif
```

## Using multiple guards

When using the default Laravel auth configuration all of the above methods will work out of the box, no extra configuration required.

However when using multiple guards they will act like namespaces for your permissions and roles. Meaning every guard has its own set of permissions and roles that can be assigned to their user model.

### Using permissions and roles with multiple guards

By default the default guard (`config('auth.default.guard')`) will be used as the guard for new permissions and roles. When creating permissions and roles for specific guards you'll have to specify their `guard_name` on the model:

```php
// Create a superadmin role for the admin users
$role = Role::create(['guard_name' => 'admin', 'name' => 'superadmin']);

// Define a `publish articles` permission for the admin users belonging to the admin guard
$permission = Permission::create(['guard_name' => 'admin', 'name' => 'publish articles']);

// Define a *different* `publish articles` permission for the regular users belonging to the web guard
$permission = Permission::create(['guard_name' => 'web', 'name' => 'publish articles']);
```

To check if a user has permission for a specific guard:

```php
$user->hasPermissionTo('publish articles', 'admin');
```

### Assigning permissions and roles to guard users

You can use the same methods to assign permissions and roles to users as described above in [using permissions via roles](#using-permissions-via-roles). Just make sure the `guard_name` on the permission or role matches the guard of the user, otherwise a `GuardDoesNotMatch` exception will be thrown.

### Using blade directives with multiple guards

You can use all of the blade directives listed in [using blade directives](#using-blade-directives) by passing in the guard you wish to use as the second argument to the directive:

```php
@role('super-admin', 'admin')
    I am a super-admin!
@else
    I am not a super-admin...
@endrole
```

## Using a middleware

This package comes with `RoleMiddleware` and `PermissionMiddleware` middleware. You can add them inside your `app/Http/Kernel.php` file.

```php
protected $routeMiddleware = [
    // ...
    'role' => \Maklad\Permission\Middlewares\RoleMiddleware::class,
    'permission' => \Maklad\Permission\Middlewares\PermissionMiddleware::class,
];
```

Then you can protect your routes using middleware rules:

```php
Route::group(['middleware' => ['role:super-admin']], function () {
    //
});

Route::group(['middleware' => ['permission:publish articles']], function () {
    //
});

Route::group(['middleware' => ['role:super-admin','permission:publish articles']], function () {
    //
});
```
You can protect your controllers similarly, by setting desired middleware in the constructor:

```php
public function __construct
{
    $this->middleware(['role:super-admin','permission:publish articles']);
}
```

## Using artisan commands

You can create a role or permission from console with artisan commands.

```bash
php artisan permission:create-role writer
```

```bash
php artisan permission:create-permission edit articles
```

When creating permissions and roles for specific guards you can specify the guard names as a second argument:

```bash
php artisan permission:create-role writer web
```

```bash
php artisan permission:create-permission 'edit articles' web
```

## Database Seeding

Two notes about Database Seeding:

1. It is best to flush the `Maklad.permission.cache` before seeding, to avoid cache conflict errors. This can be done from an Artisan command (see Troubleshooting: Cache section, later) or directly in a seeder class (see example below).

2. Here's a sample seeder, which clears the cache, creates permissions, and then assigns permissions to roles:
```php
use Illuminate\Database\Seeder;
use Maklad\Permission\Models\Role;
use Maklad\Permission\Models\Permission;

class RolesAndPermissionsSeeder extends Seeder
{
    public function run()
    {
        // Reset cached roles and permissions
        app()['cache']->forget('maklad.permission.cache');
        
        // create permissions
        Permission::create(['name' => 'edit articles']);
        Permission::create(['name' => 'delete articles']);
        Permission::create(['name' => 'publish articles']);
        Permission::create(['name' => 'unpublish articles']);
        
        // create roles and assign existing permissions
        $role = Role::create(['name' => 'writer']);
        $role->givePermissionTo('edit articles');
        $role->givePermissionTo('delete articles');
        
        $role = Role::create(['name' => 'admin']);
        $role->givePermissionTo('publish articles');
        $role->givePermissionTo('unpublish articles');
    }
}
```

## Extending

If you need to extend or replace the existing `Role` or `Permission` models you just need to
keep the following things in mind:

- Your `Role` model needs to implement the `Maklad\Permission\Contracts\Role` contract
- Your `Permission` model needs to implement the `Maklad\Permission\Contracts\Permission` contract
- You must publish the configuration with this command:
  ```bash
  php artisan vendor:publish --provider="Maklad\Permission\PermissionServiceProvider" --tag="config"
  ```
  And update the `models.role` and `models.permission` values

## Troubleshooting

### Cache

If you manipulate permission/role data directly in the database instead of calling the supplied methods, then you will not see the changes reflected in the application, because role and permission data is cached to speed up performance.

To manually reset the cache for this package, run:
```bash
php artisan cache:forget maklad.permission.cache
```

When you use the supplied methods, such as the following, the cache is automatically reset for you:

```php
// see earlier in the README for how these methods work:
$user->assignRole('writer');
$user->removeRole('writer');
$role->givePermissionTo('edit articles');
$role->revokePermissionTo('edit articles');
```

## Need a UI?

As we are based on [laravel-permission][link-laravel-permission]. The package doesn't come with any screens out of the box, you should build that yourself. To get started check out [this extensive tutorial](https://scotch.io/tutorials/user-authorization-in-laravel-54-with-spatie-laravel-permission) by [Caleb Oki](http://www.caleboki.com/).

## Change log

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Testing

``` bash
composer test
```

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) and [CONDUCT](CONDUCT.md) for details.

## Security

If you discover any security related issues, please email dev.mostafa.maklad@gmail.com instead of using the issue tracker.

## Credits

- [Freek Van der Herten][link-freekmurze]
- [Mostafa Maklad][link-author]
- [All Contributors][link-contributors]

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.

[ico-version]: https://img.shields.io/packagist/v/mostafamaklad/laravel-permission-mongodb.svg?style=flat-square
[ico-license]: https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square
[ico-travis]: https://img.shields.io/travis/mostafamaklad/laravel-permission-mongodb/master.svg?style=flat-square
[ico-scrutinizer-build]: https://scrutinizer-ci.com/g/mostafamaklad/laravel-permission-mongodb/badges/build.png?b=master
[ico-scrutinizer]: https://img.shields.io/scrutinizer/coverage/g/mostafamaklad/laravel-permission-mongodb.svg?style=flat-square
[ico-code-quality]: https://img.shields.io/scrutinizer/g/mostafamaklad/laravel-permission-mongodb.svg?style=flat-square
[ico-styleci]: https://styleci.io/repos/100894062/shield
[ico-downloads]: https://img.shields.io/packagist/dt/mostafamaklad/laravel-permission-mongodb.svg?style=flat-square
[ico-code-climate]:https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb/badges/gpa.svg
[ico-test-coverage]:https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb/badges/coverage.svg
[ico-issue-count]:https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb/badges/issue_count.svg
[ico-dependency]: https://gemnasium.com/badges/github.com/mostafamaklad/laravel-permission-mongodb.svg


[link-packagist]: https://packagist.org/packages/mostafamaklad/laravel-permission-mongodb
[link-travis]: https://travis-ci.org/mostafamaklad/laravel-permission-mongodb
[link-scrutinizer-build]: https://scrutinizer-ci.com/g/mostafamaklad/laravel-permission-mongodb/build-status/master
[link-scrutinizer]: https://scrutinizer-ci.com/g/mostafamaklad/laravel-permission-mongodb/code-structure
[link-code-quality]: https://scrutinizer-ci.com/g/mostafamaklad/laravel-permission-mongodb
[link-styleci]: https://styleci.io/repos/100894062
[link-downloads]: https://packagist.org/packages/mostafamaklad/laravel-permission-mongodb
[link-code-climate]: https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb
[link-test-coverage]: https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb/coverage
[link-dependency]: https://gemnasium.com/github.com/mostafamaklad/laravel-permission-mongodb
[link-issue-count]: https://codeclimate.com/github/mostafamaklad/laravel-permission-mongodb


[link-author]: https://github.com/mostafamaklad
[link-contributors]: ../../contributors
[link-laravel-permission]: https://github.com/spatie/laravel-permission
[link-laravel-mongodb]: https://github.com/jenssegers/laravel-mongodb
[link-freekmurze]: https://github.com/freekmurze