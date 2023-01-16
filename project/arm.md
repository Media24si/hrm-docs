# ARM

The ARM package contains shared models and components used across all of the ARM subprojects. This includes (as of the time of writing) Customers (and their related models such as Banks, Contacts, Countries, Trr accounts...), Users, Permissions and Media.

## Models

Below are global models used in the ARM package, that can be used or extended by your application.

### Customer

This model contains the customer data of ARM.
Inner customers are marked by the `issuer` property.
A customer has:
- Many Contacts
- One Country
- Many Morphed ContactValues
- Many TRR accounts

### Role

The Role and Permission models are somewhat based on Spatie's [Permission](https://spatie.be/docs/laravel-permission/v5/introduction) library but with small changes due to our structure.
A Role is always tied to an application (CRM, HRM, BPP, EYE, etc.).
A Role contains:
- is tied to Many Users
- contains Many Permissions

The Role model also exposes additional helper methods such as:
`create`

```php
<?php

// Automatically sets the app_id based on the current application
Role::create(['M24 - Basic Permission']);
```

`givePermissionTo(\ARM\Models\Permission | string $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
use ARM\Models\Role;

$regulars = Role::create([
    'name' => 'Media24 - Komerciala',
]);
$regulars->givePermissionTo('createOrder');
$regulars->givePermissionTo('issueOrderForIssuer', \App\Models\Customer::class, 11262);
```

`revokePermissionTo(\ARM\Models\Permission | string $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
use ARM\Models\Customer;

// Get role $regulars = $role
$regulars->revokePermissionTo('createOrder');
$regulars->revokePermissionTo('issueOrderForIssuer', Customer::class, 11262);
```

`hasPermissionTo(\ARM\Models\Permission | string | int $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
use ARM\Models\Customer;
use ARM\Models\Permission;

// Get role $regulars = $role
$regulars->hasPermissionTo('createOrder');
$regulars->hasPermissionTo('issueOrderForIssuer', Customer::class, 11262);

// Or pass in the permission model
$permission = Permission::firstWhere('name', 'createOrder');
$regulars->hasPermissionTo($permission);

// Or by the permission id
$permission = Permission::firstWhere('name', 'createOrder');
$regulars->hasPermissionTo($permission->id);
```


### Permission

The Permission Model exposes the `create` method, similar to the `Role` model.

```php
<?php
use ARM\Models\Permission;

// Automatically sets the app_id based on the current application
Permission::create(['createOrder']);
```

### User

The default user model class of an ARM user.
Exposes application access and specific user functionality, profile image, signature and permissions / roles.
A user has:
- access to Many Apps
- Many Roles
- Many Permissions

The `User` model exposes the `hasAppAccess` method, which can check if a user has access to the current application.

```php
<?php
$hasAccess = $user->hasAppAccess(); //  true | false
```

The user model contains a scope `withAccessToApplication` which you can use in your eloquent query builder to only get users that are given access to a specific app (the specific application you are currently developing).
So for example in your app, you would like to list only users that have access to the application:

```php
<?php
use ARM\Models\User;

$users = User::withAccessToApplication()
    ->orderBy('name', 'asc')
    ->when(
        $request->has('q'),
        fn($q) => $q->where('name', 'like', "%" . $request->input('q') . "%")
    )
    ->paginate(200)
```

The `User` model also exposes similar methods to help with setting permissions as the `Role` model described above.

`assignRole(Role|string $role)`
```php
<?php

use ARM\Models\Role;

$role = Role::create('Media24 - Regular');
$regularUser->assignRole($role);

// Or assign a role by the name if the role already exists
$regularUser->assignRole('Media24 - Regular');
```

`removeRole(Role|string $role)`
```php
<?php

use ARM\Models\Role;

$regularUser->removeRole($role);

// Or remove a role by the name if the role already exists
$regularUser->removeRole('Media24 - Regular');
```

`hasRole(Role|string $role)`
```php
$regularUser->hasRole('Media24 - Regular'); // true | false
```

`givePermissionTo(\ARM\Models\Permission | string $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
$user->givePermissionTo('createOrder');
$user->givePermissionTo('issueOrderForIssuer', \App\Models\Customer::class, 11262);
```

`revokePermissionTo(\ARM\Models\Permission | string $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
use ARM\Models\Customer;

$user->revokePermissionTo('createOrder');
$user->revokePermissionTo('issueOrderForIssuer', Customer::class, 11262);
```

`hasPermissionTo(\ARM\Models\Permission | string | int $permission, \Illuminate\Database\Eloquent\Model | string $entity = null, $entity_id = null)`

```php
<?php
use ARM\Models\Customer;
use ARM\Models\Permission;

$user->hasPermissionTo('createOrder'); // true | false
$user->hasPermissionTo('issueOrderForIssuer', Customer::class, 11262); // true | false

// Or pass in the permission model
$permission = Permission::firstWhere('name', 'createOrder');
$user->hasPermissionTo($permission); // true | false

// Or by the permission id
$permission = Permission::firstWhere('name', 'createOrder');
$user->hasPermissionTo($permission->id); // true | false
```

## Implementing ACL

The ARM package handles the basic ACL of which user has access to which application.
Each application then takes care of it's own specific ACL through `Policies` and `Models`.

So for example a `view` policy for a model would look like this:
```php
<?php
namespace App\Policies;

use App\Models\Order;
use App\Models\User;

class OrderPolicy
{
    public function view(User $user, Order $order)
    {
        if ($user->id == $order->user_id) {
            return true;
        }

        // Go through each role from the user of the order and check if the current
        // user has rights to view orders from that role
        if ($order->user->roles->first(fn ($role) => $user->hasPermissionTo('viewOrderFromRole', $role))) {
            return true;
        }

        // If the user has global viewOrder permissions or has access to
        // the order through a single order permission or has access to
        // the order through an assigned specific user
        return $user->hasPermissionTo('viewOrder')
            || $user->hasPermissionTo('viewOrder', $order)
            || $user->hasPermissionTo('viewOrder', $order->user);
    }
}
```

An example of a view query including permissions on a model would look like this:

```php
<?php

class Order extends Model implements HasMedia
{
    public function scopeViewable($query): \Illuminate\Database\Eloquent\Builder
    {
        return $this->buildViewQueryFromPermissions($query);
    }

    public function buildViewQueryFromPermissions(\Illuminate\Database\Eloquent\Builder $query): \Illuminate\Database\Eloquent\Builder
    {
        $this->appendFilterToQueryBasedOnPermissions($query);

        return $query;
    }

    public function appendFilterToQueryBasedOnPermissions(\Illuminate\Database\Eloquent\Builder $query): void
    {
        // Get all permissions pertaining to the user (roles and user permissions)
        $permissionsCollection = $this->getAllPermissionsForTheCurrentUser();

        // Return without filtering for global view permissions
        if ($permissionsCollection->first(fn ($permission) => $permission->name == 'viewOrder' && $permission->pivot->entity_type == null)) {
            return;
        }

        // Filter order view by roles
        $roleIds = $permissionsCollection->filter(fn ($permission) => $permission->name == 'viewOrderFromRole')->pluck('pivot.entity_id');

        // Filter order view by custom added users
        $userIds = $permissionsCollection->filter(fn ($permission) => $permission->name == 'viewOrder' && in_array($permission->pivot->entity_type, [\ARM\Models\User::class, \App\Models\User::class]))->pluck('pivot.entity_id');

        // Filter order view by custom specific order selections
        $orderIds = $permissionsCollection->filter(fn ($permission) => $permission->name == 'viewOrder' && $permission->pivot->entity_type == \App\Models\Order::class)->pluck('pivot.entity_id');

        $query->where(
            fn ($q) => $q
                ->where('user_id', Auth::id())
                ->when($roleIds->count(), fn ($q) => $q->orWhereRaw('user_id IN (
                    SELECT u.id
                    FROM users u
                    INNER JOIN user_roles ur ON ur.user_id = u.id
                    WHERE ur.role_id IN (' . $roleIds->map(fn () => '?')->implode(',') . ')
                )', $roleIds->toArray()))
                ->orWhereIn('user_id', $userIds)
                ->orWhereIn('id', $orderIds)
        );
    }

    public function getAllPermissionsForTheCurrentUser(): \Illuminate\Support\Collection
    {
        return \Auth::user()->getAllPermissions();
    }
}

class User extends \ARM\Models\User
{
    public function getAllPermissions(): \Illuminate\Support\Collection
    {
        return $this->roles()->with('permissions')->get()->map(fn ($role) => $role->permissions)->flatten()->merge($this->permissions);
    }
}
```


## Components

### Dropdowns

Profile dropdown for logged in users.
ARM offers a simple component to create navigation bar dropdowns calls `dropdown`.
This component is also extended with the settings component for user specific profile dropdowns.
`settings-dropdown` is the desktop version of the component, while `settings-responsive` is the mobile version.

You can call every component with the prefix `x-arm:COMPONENT_NAME`.

Example use:

```html
<nav x-data="{ open: false }" class="bg-white border-b border-gray-100">
    <div class="container">
        <div class="flex justify-between h-16">
            <div class="flex">
                <div class="flex-shrink-0 flex items-center">
                    <a href="{{ route('home') }}">
                        Application logo
                    </a>
                </div>
            </div>

            {{-- Custom dropdown example --))
            <div class="hidden sm:flex sm:items-center sm:ml-6 sm:space-x-4">
                @if(count($navigationItems))
                    <x-arm::dropdown align="right" dropdown-classes="w-48">
                        <x-slot name="trigger">
                            <button class="flex items-center text-sm font-medium text-gray-500 hover:text-gray-700 hover:border-gray-300 focus:outline-none focus:text-gray-700 focus:border-gray-300 transition duration-150 ease-in-out">
                                <div>{{ __('Ciphers') }}</div>

                                <div class="ml-1">
                                    <x-icon icon="angle-down" size="3"/>
                                </div>
                            </button>
                        </x-slot>

                        <x-slot name="content">
                            <div class="divide-y divide-gray-100">
                                <div class="py-1">
                                    @foreach($navigationItems as $link)
                                        <x-dropdown-link href="{{ route($link['route']) }}">
                                            {{ $link['name'] }}
                                        </x-dropdown-link>
                                    @endforeach
                                </div>
                            </div>
                        </x-slot>
                    </x-arm::dropdown>
                @endif

                {{-- Desktop menu user settings --))
                <x-arm::settings-dropdown/>
            </div>

            <div class="-mr-2 flex items-center sm:hidden">
                <button @click="open = ! open"
                        class="inline-flex items-center justify-center p-2 rounded-md text-gray-400 hover:text-gray-500 hover:bg-gray-100 focus:outline-none focus:bg-gray-100 focus:text-gray-500 transition duration-150 ease-in-out">
                    <svg class="h-6 w-6" stroke="currentColor" fill="none" viewBox="0 0 24 24">
                        <path :class="{'hidden': open, 'inline-flex': ! open }" class="inline-flex" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/>
                        <path :class="{'hidden': ! open, 'inline-flex': open }" class="hidden" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
                    </svg>
                </button>
            </div>
        </div>
    </div>

    <!-- Responsive Navigation Menu -->
    <div :class="{'block': open, 'hidden': ! open}" class="hidden">
        {{--
        Mobile menu navigation items...
        <x-responsive-nav-link href="{{ route('navigation-route') }}" :active="Route::currentRouteNamed('navigation-route.*')">
          {{ __('Navigation item') }}
        </x-responsive-nav-link>
        --}}

        {{-- Mobile menu user settings --))
        <x-arm::settings-responsive/>
    </div>
</nav>
```