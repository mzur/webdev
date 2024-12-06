# Lesson 6: A Basic Image Annotation Tool

In this final lesson we will implement a basic image annotation tool. It will work like this: Users can open a local image and explore the image interactively. They can place point annotations on the image which are automatically saved in the database. Whenever the user opens the same image again, the annotations are loaded and displayed.

Here is what it will look like in the end:

https://user-images.githubusercontent.com/2457311/205314421-8f8e3635-5135-441d-9beb-32b01d8526df.mp4

## Project Setup

For this lesson, we start with a fresh Laravel project again. We use the same setup and user interface scaffolding than in [lesson 5](/lesson-5), so please go back there and set up a new project as described in [Project Setup](/lesson-5#project-setup) and [User Interface Scaffolding](/lesson-5#user-interface-scaffolding).

In addition, we have to modify the PHPUnit configuration again in `phpunit.xml`:

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
```diff
- <!-- <env name="DB_CONNECTION" value="sqlite"/> -->
- <!-- <env name="DB_DATABASE" value=":memory:"/> -->
+ <env name="DB_CONNECTION" value="sqlite"/>
+ <env name="DB_DATABASE" value=":memory:"/>
```

Note that we don't only update the test suites this time but also configure new environment variables for the database connection. This will create a new temporary in-memory database just for the tests.

## Image Display

As a first step, we will implement the opening of an image file. This is fully clinet-side for now. A new image should be opened with a click on a button in the navbar, so head over to `resources/views/layout/app.blade.php` and add this:

```diff
  <a class="navbar-brand" href="{{ url('/') }}">
    {{ config('app.name', 'Laravel') }}
  </a>

+ @if (Auth::user())
+   <form class="form-inline" v-on:submit.prevent="openImage">
+     <button class="btn btn-outline-success">Open New Image</button>
+     <input class="d-none" ref="fileInput" type="file" accept="image/jpeg,image/png">
+   </form>
+ @endif
```

This is a form containing a button and an invisible file input field. The form only appears if the user is authenticated (i.e. logged-in). The file input field is restricted to JPEG and PNG images and has the special `ref="fileInput"` attribute which will be important to identify it later on. When the button is pressed, the form is submitted. The form itself has a registered event handler that calls a `openImage()` method on submit. The event handler includes the `prevent` modifier to prevent the form from executing its default submit action (i.e. sending a HTTP request).

Now let's update the Vue instance in `resources/js/app.js` to handle the submit event:

```js
const app = createApp({
    data() {
        return {
            image: null,
        };
    },
    methods: {
        openImage() {
            this.$refs.fileInput.click();
        },
        loadImage(event) {
            const [file] = event.target.files;
            if (file) {
                const reader = new FileReader();
                reader.addEventListener("load", () => {
                    const image = new Image;
                    image.addEventListener('load', this.handleImageLoaded);
                    image.src = reader.result;
                });
                reader.readAsDataURL(file);
            }
        },
        handleImageLoaded(event) {
            this.image = event.target;
        },
    },
    mounted() {
        this.$refs.fileInput.addEventListener('change', this.loadImage);
    },
});
````

The `openImage()` method now simply executes a click action on the file input element (which is [referenced](https://vuejs.org/api/component-instance.html#refs) using its `ref` ID). When a file is selected, the input will fire a `changed` event. The event listener for this event is registered in the `mounted()` hook of the Vue instance. Whenever `change` is fired, the `loadImage()` method will be called. This method used the [File API](https://developer.mozilla.org/en-US/docs/Web/API/File_API) to read the contents of the image file and create a new Image object. This needs another event listener on the `load` event of the image object because images are loaded asynchronously. Finally, the `handleImageLoaded()` method is called when a new image is fully loaded and assigns the image object to the `image` variable of the Vue instance. Feel free to add a `console.log(this.image)` here and test it in your browser.

Next, we create a new Vue component that handles the display of the image using [OpenLayers](https://openlayers.org). OpenLayers is a library to visualize interactive maps but it is great for displaying (and annotating) images as well. The component should be displayed using the full width and height available on the dashboard, so let's start by removing the CSS padding in `resources/views/layouts/app.blade.php`:

```diff
- <main class="py-4">
+ <main>

```

Then we replace the current content of the dashboard with the new component in `resources/views/home.blade.php`:

```diff
- <div class="container">
-     <div class="row justify-content-center">
-         <div class="col-md-8">
-             <div class="card">
-                 <div class="card-header">{{ __('Dashboard') }}</div>
-
-                 <div class="card-body">
-                     @if (session('status'))
-                         <div class="alert alert-success" role="alert">
-                             {{ session('status') }}
-                         </div>
-                     @endif
-
-                     {{ __('You are logged in!') }}
-                 </div>
-             </div>
-         </div>
-     </div>
- </div>
+ <image-container :image="image"></image-container>
```

Before we implement the component, we register it with the Vue instance that runs the dashboard in `resources/js/app.js`:

```diff
  import { createApp } from 'vue';
+ import ImageContainer from './components/ImageContainer.vue';
```

```diff
+ components: {
+    ImageContainer: ImageContainer,
+ },
  methods: {
```

Also we need to add OpenLayers to the JavaScript dependencies by executing `npm install ol`. Now we can finally implement the ImageContainer in `resources/js/components/ImageContainer.vue` as a [Vue single-file component](https://vuejs.org/guide/scaling-up/sfc.html):


```vue
<template>
<div class="image-container"></div>
</template>

<script>
import "ol/ol.css";
import ImageLayer from 'ol/layer/Image';
import ImageStatic from 'ol/source/ImageStatic';
import Map from 'ol/Map';
import Projection from 'ol/proj/Projection';
import View from 'ol/View';
import {getCenter} from 'ol/extent';

const imageLayer = new ImageLayer();
const map = new Map({
    layers: [imageLayer],
});

export default {
    props: {
        image: {
            type: Image,
        },
    },
    methods: {
        updateImage(image) {
            const extent = [0, 0, image.width, image.height];
            const projection = new Projection({
                code: 'image',
                units: 'pixels',
                extent: extent,
            });

            imageLayer.setSource(new ImageStatic({
                url: image.src,
                projection: projection,
                imageExtent: extent,
            }));

            map.setView(new View({
                projection: projection,
                center: getCenter(extent),
                zoom: 1,
                maxZoom: 8,
            }));
        },
    },
    watch: {
        image(image) {
            this.updateImage(image);
        },
    },
    mounted() {
        map.setTarget(this.$el);
    },
};
</script>

<style lang="scss" scoped>
.image-container {
    height: calc(100vh - 55px);
}
</style>
```

This seems like a lot of code but essentially it is just a wrapper of the OpenLayers [example to display a static image](https://openlayers.org/en/latest/examples/static-image.html) as a Vue component. So let's break this down:

```vue
<template>
<div class="image-container"></div>
</template>

<style lang="scss" scoped>
.image-container {
    height: calc(100vh - 55px);
}
</style>
```

The template is as simple as it gets. Just a single `div` element with a single CSS class. The class assigns the full screen height minus the height of the Bootstrap navbar to the element.

```js
//...

export default {
    props: {
        image: {
            type: Image,
        },
    },
    methods: {
        // ...
    },
    watch: {
        image(image) {
            this.updateImage(image);
        },
    },
    // ...
};
```

The component has an `image` property which must be an HTML `Image` element. Whenever this property changes, the `updateImage()` method is called.

```js
const imageLayer = new ImageLayer();
const map = new Map({
    layers: [imageLayer],
});

export default {
    // ...
    mounted() {
        map.setTarget(this.$el);
    },
};
```

A new OpenLayers map is created and contains a single ImageLayer, according to the [static image example](https://openlayers.org/en/latest/examples/static-image.html). When the component is mounted, the HTML element of the component is assigned as the target element of the OpenLayers map. It's important to use the `mounted()` hook of the Vue component (instead e.g. `created()`) because only at this point the HTML element actually exists. This is required by OpenLayers.

```js
export default {
    // ...
    methods: {
        updateImage(image) {
            const extent = [0, 0, image.width, image.height];
            const projection = new Projection({
                code: 'image',
                units: 'pixels',
                extent: extent,
            });

            imageLayer.setSource(new ImageStatic({
                url: image.src,
                projection: projection,
                imageExtent: extent,
            }));

            map.setView(new View({
                projection: projection,
                center: getCenter(extent),
                zoom: 1,
                maxZoom: 8,
            }));
        },
    },
    // ...
};
```

The `updateImage()` method implements the display of the image according to the [static image example](https://openlayers.org/en/latest/examples/static-image.html). Instead of just using a URL of the image as in the example, here we have an actual HTML image object and use the `src` attribute of the object as URL. This is the "dataURL" from above which is the image file as a [base64](https://en.wikipedia.org/wiki/Base64) encoded string and works just as well as an actual URL.

Now you have a  fully functional client-side image viewer already. Go ahead and open an image and explore the default available OpenLayers interactions such as zooming and panning.

## Image Database Model

When we want to store image annotations in the database later, we also need a database table for the images to which the annotations belong. We will keep it simple here and don't implement anything like a file upload and serving of the image files from the backend. Instead, the images are only opened locally, as was already implemented in the previous section. Each newly opened image file is identified by it's [SHA256](https://en.wikipedia.org/wiki/SHA-2) file hash. This way, we can fetch the annotations for a given image whenever the file is opened.

Database interactions in Laravel are done mostly through the [Eloquent](https://laravel.com/docs/eloquent) object-relational mapper (ORM). This mechanism maps database tables to PHP classes (called "models") and entries in the tables to PHP objects. We will now create a new Image model by executing `php artisan make:model -cfmr Image`, which is a convenient helper command provided by Laravel. This command will create several files which we will now step through in order to implement the new Image model.

### The Migration

File: `database/migrations/xxxx_xx_xx_xxxxxx_create_images_table.php`

The [migration](https://laravel.com/docs/migrations) defines the changes to the database that are required for the new model (i.e. creating a new table). This file will have a unique timestamp as prefix because it is essential that multiple migration files are executed in the correct order. Each migration has an `up()` method that applies the changes to the database and a `down()` method which reverts all the changes. These are already filled by the helper command to create and drop a new database table, respectively. We only need to extend the `up()` method:

```php
Schema::create('images', function (Blueprint $table) {
    $table->id();
    $table->timestamps();
    $table->string('hash', 64); // SHA256 hash
    $table->foreignId('user_id')
        ->constrained()
        ->cascadeOnDelete();

    $table->unique(['hash', 'user_id']);
});
```

Here we define that the new database table should have the columns `id`, `created_at`, `updated_at` (from `timestamps()`), `hash` and `user_id`. The `user_id` should contain a foreign key from the `users` table with the additional constraint that an a row in `images` should be deleted whenever the respective row in `users` is deleted. Constraints like these are a great way to maintain consistency in the database (e.g. you could also say that a user is not allowed to be deleted as long as they have images). Finally we define a unique composite index with the `hash` and `user_id` columns. This will speed up queries like "find the image with hash x of user y" that we need later. Also it ensures that each user can create only a single database entry for a given image (hash).

Execute `php artisan migrate` to apply the new migration to your database.

### The Model

File: `app/Models/Image.php`

This file defines special properties of the Image model that are used by the Eloquent ORM. Since Eloquent can already do many things by default and the model file was also already filled by the helper command, we only need to add these few lines to the class definition:

```php
/**
 * The attributes that are mass assignable.
 *
 * @var array
 */
protected $fillable = ['hash'];
```

This is required in order to use the convenient `create()` method of an Eloquent model later on ([here are the details](https://laravel.com/docs/eloquent#mass-assignment)).

We also update the existing User model with a new [relationship](https://laravel.com/docs/eloquent-relationships) in `app/Models/User.php`:

```php
/**
 * The images that belong to this user.
 *
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
public function images()
{
    return $this->hasMany(Image::class);
}
```

This relationship can be used to conveniently fetch all images of a user.

### The Factory

File: `database/factories/ImageFactory.php`

A [model factory](https://laravel.com/docs/eloquent-factories) defines a method to create new instances of the Image model that contain fake data (for testing). This file is also conveniently filled by the helper command and we only have to add content to the `definition()` method:

```php
return [
    'user_id' => \App\Models\User::factory(),
    'hash' => fake()->sha256(),
];
```

When a new model instance should be created using the factory, a new user will be created, too (using its factory), and a fake hash is generated. The remaining columns (`id`, `created_at` and `updated_at`) are automatically set by Eloquent.

### The Tests

You could also create a test file with the helper command from above but we do this manually here because we don't put the tests in the `Feature` directory. We create an Image model test and implement tests for the API controller.

You might wonder why we write the tests for the controller before we implement the controller (below). This is a practice that can be very helpful during development (and is inspired by [Test-driven development](https://en.wikipedia.org/wiki/Test-driven_development)). You first write a test for something you want to implement next. This test will fail, of course. Next, you actually implement this something until the test succeeds. Then you update the test to also include more of what you want to implement. Then you continue the actual implementations until the test succeeds again etc... You don't have to implement the tests completely before you start with the actual implementation but you can do it back-and-forth as described above. Here, we define the complete tests, first, because it is easier to read.

First we create the test for the new Image model in `tests/Models/ImageTest.php`:

```php
<?php

namespace Tests\Models;

use App\Models\Image;
use Illuminate\Database\QueryException;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ImageTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_image_hash_unique()
    {
        $image1 = Image::factory()->create();
        $this->expectException(QueryException::class);
        $image2 = $image1->replicate();
        $image2->save();
    }
}

```

This class contains a single test case that checks the unique constraint of an Image `user_id` and `hash`. When we try to save an additional image with the same hash belonging to the same user, the database should throw an exception.

Now let's continue with the controller tests in `tests/Http/Controllers/ImageControllerTest.php`:

```php
<?php

namespace Tests\Http\Controllers;

use App\Models\User;
use App\Models\Image;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class ImageControllerTest extends TestCase
{
    use RefreshDatabase;
}
```

This class is empty for now but we'll add test methods one by one. First we test that the route can only be accessed if the user is authenticated (i.e. logged-in):

```php
public function test_store_authenticated()
{
    $this->postJson('/api/images')->assertStatus(401);
}
```

Next, we test the behavior when a an image is opened that doesn't exist in the database yet:

```php
public function test_store_new()
{
    $user = User::factory()->create();
    $this->be($user);
    // Validation error because no hash is provided.
    $this->postJson('/api/images')->assertStatus(422);

    $hash = fake()->sha256();
    $this->postJson('/api/images', ['hash' => $hash])
        ->assertStatus(201)
        ->assertJson(fn ($json) => $json->has('id'));

    $image = $user->images()->first();
    $this->assertNotNull($image);
    $this->assertEquals($hash, $image->hash);
}
```

The [response status code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) `201` tells us that a new image was created. No we test the case when an image is opened that already exists in the database:

```php
public function test_store_existing()
{
    $user = User::factory()->create();
    $image = Image::factory()->create(['user_id' => $user->id]);
    $this->be($user);
    $this->postJson('/api/images', ['hash' => $image->hash])
        ->assertStatus(200)
        ->assertJson(fn ($json) => $json->where('id', $image->id));

    $this->assertEquals(1, $user->images()->count());
}
```

Here we check if no new image is stored in the database (the count should still be 1) and the response status code should just be `200` this time. Finally, we can test what happens if a user opens an image that already exists for another user:

```php
public function test_store_duplicate_other_user()
{
    $user = User::factory()->create();
    $image = Image::factory()->create();
    $this->be($user);
    $this->postJson('/api/images', ['hash' => $image->hash])
        ->assertStatus(201)
        ->assertJson(fn ($json) => $json->whereNot('id', $image->id));

    $newImage = $user->images()->first();
    $this->assertNotNull($newImage);
    $this->assertEquals($image->hash, $newImage->hash);
    $this->assertNotEquals($image->id, $newImage->id);
}
```

The image should be newly created for the user and should not be the same image as for the other user. Note that if we use `Image::factory()` it will create a new user automatically, which is the "other user" in this test.

You can run the tests by executing `./vendor/bin/phpunit`.

### The Controller

File: `app/Http/Controllers/ImageController.php`

After writing the tests it's time to implement the actual (API) controller that handles actions involving the Image model. The helper command already filled this file but we only need to keep the `store()` method here. Let's implement the method to satisfy the tests above:

```php
public function store(Request $request)
{
    $request->validate([
        'hash' => 'required|regex:/^[A-Fa-f0-9]{64}$/',
    ]);

    $image = $request->user()->images()->firstOrCreate([
        'hash' => $request->input('hash'),
    ]);

    $image->makeHidden('hash', 'created_at', 'updated_at', 'user_id');

    return $image;
}
```

First, we [validate](https://laravel.com/docs/validation#main-content) the request to make sure that a `hash` is provided and the hash has the correct format. If the validation fails, an error message will be returned. Next, `firstOrCreate()` is used to create a new image as part of the images that belong to the current user *or* the image will be fetched from the database if it already exists with the same hash. Then we "hide" all the attributes of the image that are irrelevant for the response. In this case we are only interested in the image ID. Finally, the Image instance is returned. This object will be automatically converted to a JSON response by Laravel.

Before the controller actually works, a new route must be configured for it in `routes/api.php`:

```php
use App\Http\Controllers\ImageController;

//...

Route::middleware('auth:sanctum')->group(function () {
    Route::post('images', [ImageController::class, 'store']);
});
```

This route uses the `auth:sanctum` middleware again to make sure that the user is authenticated. As in the previous lesson, we have to enable the appropriate middleware to also allow authentication via log-in through the browser in `app/Http/Kernel.php`:

```diff
- // \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
+ \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
```

Now execute `./vendor/bin/phpunit` again to see your tests succeed!

This finishes the implementation of the new Image model in the backend. Next, we have to use it in the frontend, too.

### The Frontend

When a new image is opened we now want to perform a request to the backend to get the (new) image ID. When the request was successful, the image should be displayed. As we need the image hash to make the request, we first have to install the [crypto-js](https://www.npmjs.com/package/crypto-js) package, which provides a SHA256 hash function, by executing `npm install crypto-js`. Now we can update `resources/js/app.js`:

```diff
  import ImageContainer from './components/ImageContainer.vue';
+ import sha256 from 'crypto-js/sha256';
```
```diff
  data() {
      return {
          image: null,
+         imageId: null,
```
```diff
  handleImageLoaded(event) {
-     this.image = event.target;
+     let hash = sha256(event.target.src).toString();
+     axios.post('api/images', {hash})
+         .then(response => this.imageId = response.data.id)
+         .then(() => this.image = event.target);
```
In `handleImageLoaded()`, the hash is computed and the axios library (remember from the previous lesson) is used to perform a `POST` request to the backend. When the request was successful, the returned image ID is saved to `this.imageId` and the image file is set to `this.image`, which will trigger the display of the image by the ImageContainer component.

Whenever a user waits on a HTTP request like this, it is nice to display a loading indicator. The Bootstrap CSS framework makes it quite easy to add one to `resources/views/layouts/app.blade.php`:

```diff
- <form class="form-inline" v-on:submit.prevent="openImage">
+ <form class="form-inline me-3" v-on:submit.prevent="openImage">
      <button class="btn btn-outline-success">Open New Image</button>
      <input class="d-none" ref="fileInput" type="file" accept="image/jpeg,image/png">
  </form>
+ <div class="spinner-border" v-if="loading"></div>
```

The spinner is only shown if the `loading` variable is `true`. Let's implement this in `resources/ja/app.js`:

```diff
  data() {
      return {
          image: null,
          imageId: null,
+         loading: false,
```
```diff
  loadImage(event) {
+     this.loading = true;
```
```diff
  axios.post('api/images', {hash})
      .then(response => this.imageId = response.data.id)
-     .then(() => this.image = event.target);
+     .then(() => this.image = event.target)
+     .finally(() => this.loading = false);
```

Done! Now you can open an image and it will create a new entry in the database or retrieve an existing one.

## Annotation Database Model

Here we will get to the core feature of our little annotation tool: the annotations. Each annotation belongs to an image, has `x` and `y` coordinates (as it is a point annotation) as well as a descriptive `label` string. We implement the Annotation model in a similar way than the Image model. If you are feeling adventurous, you can try to implement it on your own before reading on 😉 But know that there is something new this time (The Policy).

We use the helper command again to create the files required for the new model: `php artisan make_model -cfmr Annotation`.

### The Migration

File: `database/migrations/xxxx_xx_xx_xxxxxx_create_annotations_table.php`

In the migration, we add the `x`, `y` and `label` columns as well as the `image_id` foreign key with a constraint to delete all annotations of an image if the image is deleted:

```php
Schema::create('annotations', function (Blueprint $table) {
    $table->id();
    $table->timestamps();
    $table->decimal('x', 10, 2);
    $table->decimal('y', 10, 2);
    $table->string('label', 128);
    $table->foreignId('image_id')
        ->constrained()
        ->cascadeOnDelete();
});
```

Don't forget to execute `php artisan migrate` to apply the migration to your database.

### The Model

File: `app/Models/Annotation.php`

Here we just have to add the fillable attributes again:

```php
/**
 * The attributes that are mass assignable.
 *
 * @var array
 */
protected $fillable = ['x', 'y', 'label'];
```

We can also extend the Image model with a new relationship in `app/Models/Image.php`:

```php
/**
 * The annotations that belong to this image.
 *
 * @return \Illuminate\Database\Eloquent\Relations\HasMany
 */
public function annotations()
{
    return $this->hasMany(Annotation::class);
}
```

### The Factory

File: `database/factories/AnnotationFactory.php`

Complete the `definition()` method:

```php
return [
    'x' => fake()->randomFloat(2, 0),
    'y' => fake()->randomFloat(2, 0),
    'label' => fake()->word(),
    'image_id' => \App\Models\Image::factory(),
];
```

### The Tests

Here we can first extend the Image model test with a check if all annotations are deleted if the image is deleted. This also checks the existence of the new relation we implemented above. Modify `tests/Models/ImageTest.php`:

```diff
+use App\Models\Annotation;
 use App\Models\Image;
```
```php
public function test_annotation_delete_cascade()
{
    $image = Image::factory()->create();
    $annotation = Annotation::factory()->create([
        'image_id' => $image->id,
    ]);

    $image->delete();
    $this->assertNull($annotation->fresh());
}
```

Now we create a controller test (again, before creating the actual controller) and fill it with a few test methods. Create `tests/Http/Controllers/AnnotationController.php`:

```php
<?php

namespace Tests\Http\Controllers;

use App\Models\User;
use App\Models\Image;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class AnnotationControllerTest extends TestCase
{
    use RefreshDatabase;
}
```

In the first test, we check for authentication again:

```php
public function test_store_authenticated()
{
    $image = Image::factory()->create();
    $this->postJson("/api/images/{$image->id}/annotations")->assertStatus(401);
}
```

Then we also need to check for *authorization*. Where each authenticated user is authorized to create a new image for themselves, they are not authorized to add annotations to any image!

```php
public function test_store_authorized()
{
    $image = Image::factory()->create();
    $this->be(User::factory()->make());
    $this->postJson("/api/images/{$image->id}/annotations")->assertStatus(403);
}
```

As the request parameters are a little more complex this time, we test the validation in an extra test:

```php
public function test_store_validation()
{
    $user = User::factory()->create();
    $image = Image::factory()->create(['user_id' => $user->id]);
    $this->be($user);

    $this->postJson("/api/images/{$image->id}/annotations")
        ->assertStatus(422)
        ->assertJsonValidationErrors(['x', 'y', 'label']);

    $this->postJson("/api/images/{$image->id}/annotations", [
            'x' => -1,
            'y' => -1,
            'label' => '',
        ])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['x', 'y', 'label']);
}
```

Finally we can check the "happy path" when a new annotation is created:

```php
public function test_store_new()
{
    $user = User::factory()->create();
    $image = Image::factory()->create(['user_id' => $user->id]);
    $this->be($user);

    $this->postJson("/api/images/{$image->id}/annotations", [
            'x' => 1,
            'y' => 2,
            'label' => 'object',
        ])
        ->assertStatus(201);

    $annotation = $image->annotations()->first();
    $this->assertNotNull($annotation);
    $this->assertEquals(1, $annotation->x);
    $this->assertEquals(2, $annotation->y);
    $this->assertEquals('object', $annotation->label);
}
```

But wait... we also want to display existing annotations whenever a new image is opened. The annotations could be returned directly with the image ID when a new image is created or an existing one is fetched. So let's update `tests/Http/Controllers/ImageControllerTest.php`:

```diff
  use App\Models\Image;
+ use App\Models\Annotation;

@@ -39,7 +43,10 @@ public function test_store_existing()
         $this->be($user);
         $this->postJson('/api/images', ['hash' => $image->hash])
             ->assertStatus(200)
-            ->assertJson(fn ($json) => $json->where('id', $image->id));
+            ->assertJson(fn ($json) =>
+                $json->where('id', $image->id)
+                    ->where('annotations', [])
+            );
```

And we also add a new test method for this case:

```php
public function test_store_existing_annotations()
{
    $user = User::factory()->create();
    $image = Image::factory()->create(['user_id' => $user->id]);
    $annotation = Annotation::factory()->create(['image_id' => $image->id]);
    $expectAnnotation = [
        'x' => $annotation->x,
        'y' => $annotation->y,
        'label' => $annotation->label,
    ];

    $this->be($user);
    $this->postJson('/api/images', ['hash' => $image->hash])
        ->assertStatus(200)
        ->assertJson(fn ($json) =>
            $json->where('id', $image->id)
                ->where('annotations', [$expectAnnotation])
        );
}
```

### The Controller

File: `app/Http/Controllers/AnnotationController.php`

Let's add the new route first in `routes/api.php`:

```diff
+use App\Http\Controllers\AnnotationController;
 use App\Http\Controllers\ImageController;

 Route::middleware('auth:sanctum')->group(function () {
     Route::post('images', [ImageController::class, 'store']);
+    Route::post('images/{id}/annotations', [AnnotationController::class, 'store']);
```

This route includes the hierarchy that annotations belong to an image. It starts with `image`, then includes the image ID and ends with `annotations` which can be translated to "annotations of image with ID `{id}`". The ID in the route will be automatically injected as an argument of the controller method. So let's move over to the AnnotationController. As before, we only need to keep the `store()` method and populate it:

```php
public function store(Request $request, $id)
{
    $image = Image::findOrFail($id);

    $request->validate([
        'x' => 'required|numeric|min:0',
        'y' => 'required|numeric|min:0',
        'label' => 'required|string|max:128',
    ]);

    return $image->annotations()->create($request->only(['x', 'y', 'label']));
}
```

First, the image is fetched using the ID from the route. The `findOrFail()` method will automatically return a "not found" response if no image with the ID exists. The we validate the request parameters and finally create a new annotation using the relationship of the image.

Now we need to update ImageController so the annotations of an image are returned as well (if there are any). This can be done quite conveniently with [eager loading](https://laravel.com/docs/eloquent-relationships#eager-loading) of the relationship:

```diff
- $image = $request->user()->images()->firstOrCreate([
-     'hash' => $request->input('hash'),
- ]);
+ $image = $request->user()
+     ->images()
+     ->with('annotations:id,x,y,label,image_id')
+     ->firstOrCreate(['hash' => $request->input('hash')]);
```

After this change you can run the tests with `./vendor/bin/phpunit` and see all but one test succeed. So what about the failing test? This is the one checking for authorization.

### The Policy

File: `app/Policies/ImagePolicy.php`

Authorization of models is implemented with [policies](https://laravel.com/docs/authorization#creating-policies) in Laravel. A policy for the Image model can be generated using the helper command `php artisan make:policy -m Image ImagePolicy`. In this policy we want to define the ability of a user to "update" an image, i.e. add annotations to it. The helper command helpfully created a few empty methods in the policy file but we only have to keep the `update()` method.

Before we actually implement the policy method, remember to write a test, first! This can be done in `tests/Models/ImageTest.php`:

```diff
 use App\Models\Image;
+use App\Models\User;
```
```php
public function test_can_update()
{
    $user1 = User::factory()->create();
    $user2 = User::factory()->create();
    $image = Image::factory()->create(['user_id' => $user1->id]);

    $this->assertTrue($user1->can('update', $image));
    $this->assertFalse($user2->can('update', $image));
}
```

Here you see one of the authorization methods in action, the `can()` method of a user, which takes the name of an ability and a model instance as arguments. Laravel will automagically determine the appropriate policy method.

Now we can implement the `update()` method in `app/Policies/ImagePolicy.php`:

```php
public function update(User $user, Image $image)
{
    return $user->id === $image->user_id;
}
```

Only the creator of an image is authorized to update it.

We still have to implement the authorization in the `update()` method of the AnnotationController. This can be done with another convenient authorization method provided by Laravel:

```diff
  $image = Image::findOrFail($id);
+ $this->authorize('update', $image);
```

Now all the tests succeed!

You may have noticed that even for a basic application like this one the tests begin to add up. You'll be thankful for this after a few months of development and maintenance!

## Annotation Display

In the client-side application, the annotations can be implemented completely by using OpenLayers features. In fact, there is an [OpenLayers example](https://openlayers.org/en/latest/examples/draw-features.html) that shows exactly how to implement annotations with the OpenLayers draw interaction. So let's add the draw interaction to `resources/js/components/ImageContainer.vue`:

```diff
  import "ol/ol.css";
+ import Draw from 'ol/interaction/Draw';
  import ImageLayer from 'ol/layer/Image';
  import ImageStatic from 'ol/source/ImageStatic';
  import Map from 'ol/Map';
  import Projection from 'ol/proj/Projection';
+ import VectorLayer from 'ol/layer/Vector';
+ import VectorSource from 'ol/source/Vector';
```
```diff
  const imageLayer = new ImageLayer();
+ const vectorSource = new VectorSource({wrapX: false});
+ const vectorLayer = new VectorLayer({source: vectorSource});
+ const draw = new Draw({
+     source: vectorSource,
+     type: 'Point',
+ });
+
  const map = new Map({
-     layers: [imageLayer],
+     layers: [imageLayer, vectorLayer],
  });
```
```diff
  watch: {
      image(image) {
          this.updateImage(image);
      },
  },
+ created() {
+     map.addInteraction(draw);
+ },
```

This doesn't look very complicated, right? We create a new vector layer (for the points) and a draw interaction that both use the same source. Then we add the layer and interaction to the OpenLayers map. Now you can already go ahead and draw points on an image!

But an annotation needs a label, too, which the user should type into an input field each time a new annotation is created. This can be implemented with an OpenLayers overlay, and there is [an example](https://openlayers.org/en/latest/examples/overlay.html) for this, too. The example uses a Bootstrap popover but we'll keep it simpler here and extend the template of `resources/js/components/ImageContainer.vue`:

```diff
- <div class="image-container"></div>
+ <div class="image-container">
+     <div v-show="showPopup" class="popup" ref="popup">
+         <form @submit.prevent="finishPendingAnnotation">
+             <input
+                 class="form-control"
+                 type="text"
+                 placeholder="Label name"
+                 v-model="pendingLabel"
+                 ref="labelInput"
+                 @keydown.esc="discardPendingAnnotation"
+                 >
+         </form>
+     </div>
+ </div>
```

This adds a new element that contains a basic HTML form with a single input element for the label. The element is only shown if `showPopup` is `true`. Also we add a `ref` attribute so we can identify the element later on. The form and input elements have event handlers to either finish a pending annotation (on <kbd>Enter</kbd>) or discard it (on <kbd>Esc</kbd>). Also we bind a variable to the input value with `v-model`. The indentation of the input element may look strange but it makes managing lots of attributes much easier.

Now we add the overlay code:

```diff
  import Map from 'ol/Map';
+ import Overlay from 'ol/Overlay';
  import Projection from 'ol/proj/Projection';
```
```diff
  const draw = new Draw({
      source: vectorSource,
      type: 'Point',
  });
+
+ const popup = new Overlay({});
```

Also, we add some more variables to the instance. `pendingAnnotation` will contain information about the annotation while the label is typed. `pendingLabel` will be synchronized with the text typed in the HTML input element. And `showPopup` is dynamically computed to be `true` whenever there is a pending annotation.

```diff
  props: {
      image: {
          type: Image,
      },
  },
+ data() {
+     return {
+         pendingAnnotation: null,
+         pendingLabel: '',
+     };
+ },
+ computed: {
+     showPopup() {
+         return this.pendingAnnotation !== null;
+     },
+ },
```

Next we add the OpenLayers overlay to the map and connect it to the HTML element from the template. Note that this has to wait until the `mounted()` hook is called. Otherwise, the HTML element would not exist yet. Also, we register an event handler to the `addfeature` event of the vector source. This event is fired whenever a new point annotation is drawn.

```diff
  created() {
      map.addInteraction(draw);
+     map.addOverlay(popup);
+     vectorSource.on('addfeature', this.handleFeatureAdded);
  },
  mounted() {
     map.setTarget(this.$el);
+    popup.setElement(this.$refs.popup);
  },
```

Finally we add some styling to the popup so it is positioned correctly to the right of the point annotation.

```diff
  <style lang="scss" scoped>
  .image-container {
      height: calc(100vh - 55px);
  }

+ .popup {
+     position: absolute;
+     transform: translateY(-50%) translateX(12px);
+     width: 200px;
+ }
  </style>
```

As you might have noticed, we have referenced the three methods `handleFeatureAdded()`, `discardPendingAnnotation()` and `finishPendingAnnotation()` but these were not defined yet. Here goes:

```js
handleFeatureAdded(event) {
    if (this.pendingAnnotation) {
        this.discardPendingAnnotation();
    }

    popup.setPosition(event.feature.getGeometry().getCoordinates());
    this.pendingAnnotation = event.feature;
    this.$nextTick(() => this.$refs.labelInput.focus());
}
```

If a new annotation should be created but an old one is still pending, the old one is discarded (see below). Then the position of the popup overlay is updated to the position of the point annotation and the OpenLayers feature object is set as `pendingAnnotation`. Finally, the HTML input field of the popup is focused so the user can start typing right away without having to click in the input field. This must be wrapped in a call of the Vue `$nextTick()` method because we have to wait one "Vue rendering cycle" for the popup to be displayed (`this.showPopup === true`) before calling `focus()` on the element works.

```js
discardPendingAnnotation() {
    vectorSource.removeFeature(this.pendingAnnotation);
    this.pendingAnnotation = null;
    this.pendingLabel = null;
}
```

This just removes the OpenLayers feature from the vector source (removing the point on the map) and resets the variables for the pending annotation and label.

```js
finishPendingAnnotation() {
    let [x, y] = this.pendingAnnotation
        .getGeometry()
        .getCoordinates()
        .map(x => Math.round(x * 100) / 100);
    let label = this.pendingLabel;
    this.$emit('annotation', {x, y, label});
    this.pendingAnnotation = null;
    this.pendingLabel = null;
}
````

This method is called when the HTML form of the popup is "submitted" and the label was entered. First, the coordinates of the annotation point are extracted from the OpenLayers feature object. We round the coordinates to two decimals because points can be set with sub-pixel accuracy on the OpenLayers map but more than two decimals would be overkill. Then we `$emit()` an event with all the information on the new annotation and reset the pending annotation and label variables.

The emitted event must be handled by the "parent" of the component which in our case is the Vue instance in `resources/js/app.js`. But first we have to register an event handler in `resources/views/home.blade.php`:

```diff
- <image-container :image="image"></image-container>
+ <image-container
+     :image="image"
+     v-on:annotation="handleNewAnnotation"
+     ></image-container>
```

Note the use of `v-on:annotation` instead of `@annotation` because the Blade template syntax also uses `@` which could lead to confusing errors. So you should use the `@` shorthand in component templates and `v-on:` in Blade templates. Now we can implement the event handler in `resources/js/app.js`:

```js
handleNewAnnotation(annotation) {
    if (!this.imageId) {
        return;
    }

    this.loading = true;
    axios.post(`api/images/${this.imageId}/annotations`, annotation)
        .finally(() => this.loading = false);
}
```

This sends the annotation information to the annotations API we have implemented previously and event displays the loading spinner while doing so!

On to the final leg of our journey!

Here we implement the display of existing annotations when an image is first loaded. First, the annotations array that can be returned from the backend has to be stored in `resources/js/app.js`:

```diff
 data() {
     return {
         image: null,
         imageId: null,
         loading: false,
+        annotations: [],
     };
 },
```
```diff
 handleImageLoaded(event) {
     let hash = sha256(event.target.src).toString();
     axios.post('api/images', {hash})
-        .then(response => this.imageId = response.data.id)
+        .then(response => {
+            this.imageId = response.data.id;
+            this.annotations = response.data.annotations || [];
+        })
```

Note that we also need to handle the case that no annotations array is returned at all when the image was newly created. Next, we hook up these annotations as a new property of the ImageContainer component. Update `resources/views/home.blade.php`:

```diff
 <image-container
     :image="image"
+    :annotations="annotations"
     v-on:annotation="handleNewAnnotation"
     ></image-container>
```

And now `resources/js/components/ImageContainer.vue`:

```diff
 props: {
     image: {
         type: Image,
     },
+    annotations: {
+        type: Array,
+    },
 },
```

These annotations must be drawn whenever a new image is displayed, which happens in the `updateImage()` method of the component. But first we need to import a few additional OpenLayers resources:

```diff
 import Draw from 'ol/interaction/Draw';
+import Feature from 'ol/Feature';
 import ImageLayer from 'ol/layer/Image';
 import ImageStatic from 'ol/source/ImageStatic';
 import Map from 'ol/Map';
 import Overlay from 'ol/Overlay';
+import Point from 'ol/geom/Point';
```

Then we can extend the `updateImage()` method with:

```js
vectorSource.clear();
vectorSource.un('addfeature', this.handleFeatureAdded);
vectorSource.addFeatures(this.annotations.map(annotation => {
    return new Feature(new Point([annotation.x, annotation.y]));
}));
vectorSource.on('addfeature', this.handleFeatureAdded);
```

First, the vector source containing any annotations of a previous image is cleared. Then we disable the `addfeature` event listener because we don't want this event to be handled while we add the initial annotations. Then we actually create the OpenLayers point features from the annotation objects and fill the vector source with them. Finally, the `addfeature` event listener is activated again.

Done! 🏁 Congratulations, the image annotation tool is fully functional now!

## Next Steps

You have now learned how to implement a basic web application for image annotation with Laravel and Vue.js. Do you feel ready for doing some coding on your own? Here are a few suggestions for additional features that can be implemented in the image annotation tool:

- Label display: Show the label of an existing annotation in a popup when the mouse touches the annotation.

- Deletion of annotations: Enable the OpenLayers select interaction and implement all features that are required to delete existing annotations again.

- More annotation shapes: The OpenLayers draw interaction also supports circles, lines or polygons. Do you know how the backend and database have to be updated to support these shapes?

**Contribute to BIIGLE:** Want to dig into a larger project? Incidentally, there is this image annotation tool called [BIIGLE](https://biigle.de) that uses the same technology stack that you have learned here and which is desperately looking for contributors. Take a look at the [contributing guide](https://github.com/biigle/core/blob/master/CONTRIBUTING.md) to get started.

Thanks for reading this far! If you have any feedback or suggestions for improvements, please [open an issue](https://github.com/mzur/webdev/issues/new).
