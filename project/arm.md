# ARM

The ARM package contains shared models and components used across all of the ARM subprojects. This includes (as of the time of writing) Customers (and their related models such as Banks, Contacts, Countries, Trr accounts...), Users, Permissions and Media.

## Models

### User

### Customer

### Permission


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