# Lesson 5: A Full Laravel Application

Laravel is an "enterprise-level" PHP web application framework. The term "enterprise" is not always used with a positive meaning in the context of software (since you don't need complicated solutions for simple problems). But the way Laravel works makes it suitable for applications that are deployed on a large scale while still being approachable by newcomers. You don't need to understand all the details at first and you can trust that the framework does the right thing by default. This is particularly important for web applications, as there are lots of things that can be done wrong when it comes to security.

Developing with a framework such as Laravel is like playing a good game: it's easy to start but hard to master. Because of the nature of a web application and the many features that Laravel provides, it's impossible to understand everything in-depth when you are a beginner. But the good thing is that you don't need to understand everything in-depth to start getting productive. And that's wat we are going to do here, too. The [Laravel documentation](https://laravel.com/docs) is an excellent resource but you should not read past "The Basics" before you start developing. Everything else can be looked up afterwards when you need a particular feature during development.

Now we will start to set up a new Laravel project and the basic user interface scaffolding that you can get without having to implement anything yourself. Then we will recreate [lesson 2](/lesson-2) and [lesson 3](/lesson-3) using Laravel and Vue.js so you can see the differences and benefits of these frameworks to the vanilla approach.

## Project Setup

The setup of a new Laravel project requires the Composer PHP package manager that you should have installed as part of the [preparations](/#preparation) for these lessons. All you need to do is to run this command:

```bash
composer create-project laravel/laravel example-app
```

Now a new Laravel project is installed in the `example-app` directory. You can go ahead and enter the directory, then run `php artisan serve` which will start the PHP development server for the project. Visit <http://localhost:8000> and Laravel should show you the default start page.

## User Interface Scaffolding

Of yourse we don't want to keep just the Laravel start page but implement our own user interface for our application. In addition, most web applications share common features such as user accounts, registration, sign-up, email validation, passwort resets etc. Laravel provides all this fully configured in a variety of [Starter Kits](https://laravel.com/docs/starter-kits). Unfortunately, Laravel uses the [Tailwind CSS](https://tailwindcss.com) framework by default, nowadays, which is great but not very beginner friendly. Fortunately, [Laravel UI](https://github.com/laravel/ui) still exists as an alternative that uses the Bootstrap CSS framework that you already know from [lesson 4](/lesson-4). So this is what we are going to use.

First, we have to install the Laravel UI package using Composer:

```bash
composer require laravel/ui
```

Now we can instruct the package to set up the user interface scaffolding (i.e. login and registration forms etc.) with Vue.js:

```bash
php artisan ui vue --auth
```

Finally, before we can actually create user accounts, we have to configure the database. The simplest way is to use SQLite. With SQLite, the whole database is simply created as a single file in your project directory. This system is powerful enough to be used even in small- to medium-scale production scenarios. To use SQLite, modify the `DB_CONNECTION` variable in the `.env` file to `sqlite`. Also, remove the `DB_DATABASE` variable from the file, so laravel will use the default location for the database file (`database/database.sqlite`). Now you can run the database migrations which will create the database file and fill it with the tables for user accounts etc.:

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

In order to implement the features from [lesson 2](/lesson-2) we have to touch the subjects of [routing](https://laravel.com/docs/routing), [controllers](https://laravel.com/docs/controllers) and [templates](https://laravel.com/docs/blade). Routes define the URLs that the application can answer to, controllers implement the logic of what happens when a route is visited and templates help to define the HTML of a response page that might be returned by a controller. With the default scaffolding of Laravel UI, there already exists a `/home` route with a dashboard that you should see when you are logged in. You can find the route definition in `routes/web.php`, the controller in `app/Http/Controllers/HomeController.php` and the template in `resources/views/home.blade.php`.

## Recreating Lesson 3
