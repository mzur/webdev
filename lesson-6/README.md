# Lesson 6: A Basic Image Annotation Tool

In this final lesson we will implement a basic image annotation tool. It will work like this: Users can open a local image and explore the image interactively. They can place point annotations on the image which are automatically saved in the database. Whenever the user opens the same image again, the annotations are loaded and displayed.

TODO: MP4 demo screencast

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

As a first step, we will implement the opening of an image file. This is fully clinet-side for now. A new image should be opened with a click on a button in the navbar, so head over to `resources/viws/layout/app.blade.php` and add this:

```diff
  <a class="navbar-brand" href="{{ url('/') }}">
    {{ config('app.name', 'Laravel') }}
  </a>

+ <form class="form-inline" v-on:submit.prevent="openImage">
+   <button class="btn btn-outline-success">Open New Image</button>
+   <input class="invisible" ref="fileInput" type="file" accept="image/jpeg,image/png">
+ </form>
```

This is a form containing a button and an invisible file input field. The file input field is restricted to JPEG and PNG images and has the special `ref="fileInput"` attribute which will be important to identify it later on. When the button is pressed, the form is submitted. The form itself has a registered event handler that calls a `openImage()` method on submit. The event handler includes the `prevent` modifier to prevent the form from executing its default submit action (i.e. sending a HTTP request).

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

Next, we create a new Vue component that handles the display of the image using [OpenLayers](https://openlayers.org). OpenLayers is a library to visualize interactive maps but it is great for displaying (and annotating) images as well.

- npm install ol

## Image Database Model

## Annotation Display

## Annotation Database Model

## Wrapping It Up

- set up app according to lesson 5 "project setup" and "user interface scaffolding"

- configure tests with database

- tool: select local image. explore image with OpenLayers. create image model for user in DB. local image path is unique ID (can we get the full path?). user can add point annotations to image (with free-hand labels). user can load new local image, if it has annotations, annotations are displayed. test everything while adding features!
