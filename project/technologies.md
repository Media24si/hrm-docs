# Used technologies

## Livewire

Use [Livewire](https://laravel-livewire.com/) through the majority of the project. Most views should be Livewire (routes too).

An example route would be:
```php
Route::get('/', \App\Http\Livewire\Home\Index::class)
  ->name('home');
```

### Installation

Install livewire with `sail composer require livewire/livewire`.

## AlpineJS

Since Livewire is coupled extremely well by [AlpineJS](https://alpinejs.dev/), use Alpine for any additional JS work on the frontend / components.

## Tailwind

For CSS use [TailwindCSS](https://tailwindcss.com/) with [TailwindUI](https://tailwindui.com/) for design looks.