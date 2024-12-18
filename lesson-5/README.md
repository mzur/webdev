# Lesson 5: A Full Laravel Application

Laravel is an "enterprise-level" PHP web application framework. The term "enterprise" is not always used with a positive meaning in the context of software (since you don't need complicated solutions for simple problems). But the way Laravel works makes it suitable for applications that are deployed on a large scale while still being approachable by newcomers. You don't need to understand all the details at first and you can trust that the framework does the right thing by default. This is particularly important for web applications, as there are lots of things that can be done wrong when it comes to security.

Developing with a framework such as Laravel is like playing a good game: it's easy to start but hard to master. Because of the nature of a web application and the many features that Laravel provides, it's impossible to understand everything in-depth when you are a beginner. But the good thing is that you don't need to understand everything in-depth to start getting productive. And that's what we are going to do here, too. The [Laravel documentation](https://laravel.com/docs) is an excellent resource but you should not read past "The Basics" before you start developing. Everything else can be looked up afterwards when you need a particular feature during development.

Now we will start to set up a new Laravel project and the basic user interface scaffolding that you can get without having to implement anything yourself. Then we will recreate [lesson 2](/lesson-2) and [lesson 3](/lesson-3) using Laravel and Vue.js so you can see the differences and benefits of these frameworks to the vanilla approach.

## Project Setup

The setup of a new Laravel project requires the Composer PHP package manager that you should have installed as part of the [preparations](/#preparation) for these lessons. All you need to do is to run these commands:

```bash
composer create-project laravel/laravel example-app
php artisan install:api
```

Now a new Laravel project is installed in the `example-app` directory. Go ahead and enter the directory. Before we can start, one line must be updated in the `.env` file:

```diff
- APP_URL=http://localhost
+ APP_URL=http://localhost:8000
```

Now you can run `php artisan serve` which will start the PHP development server for the project. Visit <http://localhost:8000> and Laravel should show you the default start page.

## User Interface Scaffolding

Of course we don't want to keep just the Laravel start page but implement our own user interface for our application. In addition, most web applications share common features such as user accounts, registration, sign-up, email validation, password resets etc. Laravel provides all this fully configured in a variety of [Starter Kits](https://laravel.com/docs/starter-kits). Unfortunately, Laravel uses the [Tailwind CSS](https://tailwindcss.com) framework by default, nowadays, which is great but not very beginner friendly. Fortunately, [Laravel UI](https://github.com/laravel/ui) still exists as an alternative that uses the Bootstrap CSS framework that you already know from [lesson 4](/lesson-4). So this is what we are going to use.

First, we have to install the Laravel UI package using Composer:

```bash
composer require laravel/ui
```

Now we can instruct the package to set up the user interface scaffolding (i.e. login and registration forms etc.) with Vue.js:

```bash
php artisan ui vue --auth
```

Finally, before we can actually create user accounts, we have to configure the database. The simplest way is to use SQLite. With SQLite, the whole database is simply created as a single file in your project directory. This system is powerful enough to be used even in small- to medium-scale production scenarios. To use SQLite, modify the `.env` file:

```diff
- DB_CONNECTION=mysql
+ DB_CONNECTION=sqlite
  DB_HOST=127.0.0.1
  DB_PORT=3306
- DB_DATABASE=laravel
```

The default location of the database file will be `database/database.sqlite`. Now you can run the database migrations which will create the database file and fill it with the tables for user accounts etc.:

```bash
php artisan migrate
```

That's it!

Since we now do the CSS and JavaScript "enterprise-level", too, it is no longer included inline in the HTML. Instead, it is neatly split into separate files which you will find in the `resources/js` and `resources/sass` directories. Instead of plain CSS, Laravel uses [SASS](https://sass-lang.com) which has a few nice additional features. And instead of a single plain JavaScript file, Laravel uses multiple files and also Vue [single file components](https://vuejs.org/guide/scaling-up/sfc.html). However, the browser still expects plain CSS and JavaScript files when is should display a web page. So to get these files, the SASS and JavaScript/Vue component files are compiled using the [Vite](https://vitejs.dev) build tool. This tool offers some cool features for development, too. But first, we have to install the JavaScript packages using the NPM package manager:

```bash
npm install
```

Now we can use NPM to start Vite in development mode:

```bash
npm run dev
```

This will compile all assets (SASS/JavaScript/Vue) and also run a development server for so-called hot module replacement. This means that, if you open the application in your browser, the page in the browser will be automatically refreshed whenever you make any changes to your SASS or JavaScript files. How hot is that?

If you want to compile the assets for production, instead (which will make the files smaller and more optimized), run `npm run build`. With the PHP development server (`php artisan serve`) and Vite (`npm run dev`) up and running, we can start developing!

## Recreating Lesson 2

Here we want to implement the simple features of the server-side application of [lesson 2](/lesson-2) with Laravel. You will notice that the implementation will be much more abstract and verbose but this is a good thing for the sustainability of a large project! But before you start, create a new user account and log in to your application.

In order to implement the features from [lesson 2](/lesson-2), we have to touch the subjects of [routing](https://laravel.com/docs/routing), [controllers](https://laravel.com/docs/controllers) and [templates](https://laravel.com/docs/blade). Routes define the URLs that the application can answer to, controllers implement the logic of what happens when a route is visited and templates help to define the HTML of a response page that might be returned by a controller. With the default scaffolding of Laravel UI, there already exists a `/home` route with a dashboard that you should see when you are logged in. You can find the route definition in `routes/web.php`, the controller in `app/Http/Controllers/HomeController.php` and the template in `resources/views/home.blade.php`.

Let's implement a dynamic welcome message that uses information from the query string of the URL. To fetch information from the query string, we have to update the `index()` method of the dashboard controller in `app/Http/Controllers/HomeController.php`:

```php
public function index(Request $request)
{
    return view('home', [
        'name' => $request->input('name', 'dude'),
    ]);
}
```

Rather than just returning a view, this will get "input" from the request (which can be anything from the query string or from the request body). Here, we fetch the value of the `name` key again and fall back to `'dude'` if there is no value. This is passed on to the view as a variable called `name`. Now we can update the template to use this variable in `resources/views/home.blade.php`:

```diff
- {{ __('You are logged in!') }}
+ {{ "You are logged in, {$name}!" }}
```

For simplicity, we replace the `__()` helper function for [localization](https://laravel.com/docs/localization) with a plain string and include the name variable there. Now you can visit <http://localhost:8000/home?name=Joe> and see the dynamic welcome message in action.

But wait... this is now an enterprise-level application and we have actual user accounts with user names! So let's use the user name instead of the value from the query string in the controller:

```diff
- 'name' => $request->input('name', 'dude'),
+ 'name' => $request->user()->name,
```

Nifty! 😎

## Recreating Lesson 3

Now we have the very basics of a server-side application with Laravel figured out. Here comes the client-side application with Vue.js. Since we already found a good solution for setting the user name, we will use a nice demo package of Laravel that returns random inspiring quotes and load these dynamically via JavaScript and the API.

First, we define a new API route in `routes/api.php`:

```php
Route::middleware('auth:sanctum')->get('/quote', [App\Http\Controllers\Api\QuoteController::class, 'get']);
```

The `auth:sanctum` [middleware](https://laravel.com/docs/middleware) makes sure that the user is authenticated (i.e. providing an API token), so this route can only be used by registered users. In order to also allow logged-in users to access the route (without API token), we have to enable a middleware in `bootstrap/app.php`:

```diff
 ->withMiddleware(function (Middleware $middleware) {
+    $middleware->statefulApi();
 })
```

The new route is defined to listen to [`GET` requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) at the `/api/quote` URL (the `api` prefix is automatically added for all routes in `api.php`) and to process these requests in the `get()` method of a `QuoteController`. Let's continue to create this controller as `app/Http/Controllers/Api/QuoteController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Foundation\Inspiring;

class QuoteController extends Controller
{
    /**
     * Return an inspiring quote.
     *
     * @return string
     */
    public function get()
    {
        return json_encode(['quote' => Inspiring::quote()]);
    }
}
```

As this is the first time we write an actual PHP class, we'll quickly step through the lines in detail:

- `namespace ...` defines the "full path" to the class name, which is `App\Http\Controllers\Api\QuoteController`. This _must_ be reflected by the actual file path `app/Http/Controllers/Api/QuoteController.php`, as the namespace and class name is used to define which file must be opened to initialize the class when it is used.

- `use ...` statements define the full paths to other classes that are not within the same namespace of the current class.

- `class QuoteController extends Controller` begins the class definition of a new controller, which extends the default controller class that already includes some helpful methods.

- `public function get()` is a method of the controller that will be called when the `/api/quote` route is visited (as defined above).

With this, the API is finished already! Let's head over to the dashboard template and add some HTML elements:

```diff
  {{ "You are logged in, {$name}!" }}
+ <button class="btn btn-primary" v-on:click="getQuote">Get a quote</button>
+ <blockquote class="blockquote" v-if="quote" v-html="quote"></blockquote>
```

This is a button that can be clicked and a blockquote element. Both elements have these strange `v-...` attributes which are understood by Vue.js:

- `v-on:click="getQuote"` registers an event handler that calls the `getQuote()` method whenever the button is clicked.

- `v-if="quote"` shows the HTML element only if the `quote` variable has any content.

- `v-html="quote"` inserts the content of the `quote` variable into the HTML element.

But we haven't defined any `getQuote()` method or `quote` variable, yet, right? Vue knows this, too, and complains about this. You can see this by opening the browser developer tools (<kbd>F12</kbd>) and switching to the Console. There you will see warning messages:

```
 [Vue warn]: Property "getQuote" was accessed during render but is not defined on instance.
 [Vue warn]: Property "quote" was accessed during render but is not defined on instance.
```

All this only works because a Vue instance is already created and mounted as part of the Laravel UI scaffolding. The instance is defined in `resources/js/app.js`. At the bottom of the file you'll see that the instance is mounted to an HTML element with the `app` ID attribute. This element can be found in `resources/views/layouts/app.blade.php`.

Let's silence the Vue warnings and extend the Vue instance in `resources/js/app.js`:

```js
const app = createApp({
   data() {
      return {
         quote: '',
      };
   },
   methods: {
      getQuote() {
         axios.get('/api/quote').then(response => this.quote = response.data.quote);
      },
   },
});
````

This defines a new `quote` variable (in `data()`) and a new `getQuote()` function (in `methods`). When `getQuote()` is called, the [axios](https://axios-http.com) library is used to make a `GET` HTTP request to the new API endpoint (axios is bundled with Laravel and initialized in `resources/js/bootstrap.js`). When the request is successful, the `quote` is extracted from the response and assigned to the `quote` variable of the Vue instance. Vue then magically takes care of displaying the quote in the HTML. Give it a try!

You can press the button multiple times and get a new quote each time. To observe the HTTP requests that are sent in the background, open the developer tools (<kbd>F12</kbd>) again and switch to the Network tab.

As with Laravel, we can't get into great detail about Vue here. You should read the Introduction and Essentials of the [Vue documentation](https://vuejs.org/guide/introduction.html) before you continue to the next lesson.

## Tests

Tests are crucial when you start to implement a new application that should be maintained for a long time. It may seem cumbersome at first because you appear to write lots of code that does not add any features but you'll be thankful for this in a few months or years! Also think about new developers who join or take over a project and who can't know the consequences of the changes they make to the code. Tests for the rescue!

Here comes my personal opinion about testing a Laravel web application:

> Laravel suggests that you separate tests into Feature tests and Unit tests. I don't do this separation and just write "tests" for each backend class. This could be both unit tests (i.e. testing a single method of a class directly) or feature tests (i.e. testing an API route that is handled by a controller method) in the same file. I strive to fully test the backend and especially the API. Frontend tests, however, are only viable for larger teams, in my opinion. As long as the backend is fully tested, even if the frontend breaks, nothing bad can happen (e.g. database corruption, authentication or authorization failures).

Tests are often omitted from tutorials like these but they are an important part of learning web development. The default Laravel setup already includes a test setup as well and Laravel provides many features to help writing tests. So let's see how we can test the modifications to the dashboard and the new API route that we implemented previously.

The test files are located in the `tests` directory. Tests are executed with [PHPUnit](https://phpunit.de). Laravel comes with a few example tests so you can already run the command `./vendor/bin/phpunit` to execute the tests:

```
PHPUnit 9.5.26 by Sebastian Bergmann and contributors.

..                                                                  2 / 2 (100%)

Time: 00:00.090, Memory: 22.00 MB

OK (2 tests, 2 assertions)
```

Let's add a new test for the dashboard controller in `tests/Http/Controllers/HomeControllerTest.php`:

```php
<?php

namespace Tests\Http\Controllers;

use App\Models\User;
use Tests\TestCase;

class HomeControllerTest extends TestCase
{
    public function test_redirect_unauthenticated()
    {
        $this->get('/home')->assertRedirect('/login');
    }

    public function test_see_username_on_dashboard()
    {
        $user = User::factory()->make();
        $this->actingAs($user)
            ->get('/home')
            ->assertStatus(200)
            ->assertSee("You are logged in, {$user->name}!");
    }
}
```

This class includes tests to check if an unauthenticated visitor is redirected to the login page and if a logged-in user sees their own user name on the dashboard. You can learn more about the available methods for testing in the [Laravel documentation](https://laravel.com/docs/testing).

Let's execute the new tests `./vendor/bin/phpunit --filter HomeControllerTest`:

```
PHPUnit 9.5.26 by Sebastian Bergmann and contributors.

No tests executed!
```

Wait, no tests executed? That's because we didn't write the test either as Feature or as Uni test as Laravel suggested. We have to update the PHPUnit configuration in `phpunit.xml` first:

```diff
- <testsuite name="Unit">
-     <directory suffix="Test.php">./tests/Unit</directory>
- </testsuite>
- <testsuite name="Feature">
-     <directory suffix="Test.php">./tests/Feature</directory>
- </testsuite>
+ <testsuite name="Tests">
+    <directory suffix="Test.php">./tests</directory>
+ </testsuite>
```

Again `./vendor/bin/phpunit --filter HomeControllerTest`:

```
PHPUnit 9.5.26 by Sebastian Bergmann and contributors.

..                                                                  2 / 2 (100%)

Time: 00:00.112, Memory: 26.00 MB

OK (2 tests, 4 assertions)
```

Much better! You can try to change the welcome message on the dashboard to see the test fail.

Now we add another test for the API route in `tests/Http/Controllers/Api/QuoteControllerTest.php`:

```php
<?php

namespace Tests\Http\Controllers\Api;

use App\Models\User;
use Tests\TestCase;

class QuoteControllerTest extends TestCase
{
   public function test_get_deny_unauthenticated()
    {
        $this->getJson('/api/quote')->assertStatus(401);
    }

    public function test_get()
    {
        $user = User::factory()->make();
        $this->actingAs($user)
            ->getJson('/api/quote')
            ->assertStatus(200)
            ->assertJson(fn ($json) => $json->has('quote'));
    }
}
```

These tests now use the `getJson` method to simulate requests done by JavaScript. We check again that access is denied if the user is not authenticated and that we get a JSON object including a `quote` otherwise. A final `./vendor/bin/phpunit` runs all tests:

```
PHPUnit 9.5.26 by Sebastian Bergmann and contributors.

......                                                              6 / 6 (100%)

Time: 00:00.131, Memory: 26.00 MB

OK (6 tests, 10 assertions)
```

Now you should probably sit down and read a little more about [Laravel](https://laravel.com/docs) and [Vue](https://vuejs.org/guide/introduction.html) in their documentation. You can also experiment a little more with your new appplication and the tests. Next up we will build a real application, so bring some time and snacks! 🍿

**Next lesson:** [A Basic Image Annotation Tool](/lesson-6)
