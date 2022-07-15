# Lesson 4: CSS (interlude)

Cascading style sheets (CSS) are used to define the presentation or look of the content of a web page (in contrast to the structure and semantics that are defined in the HTML). Almost all websites and web applications use CSS and, while there is some debate that it is sometimes used too extensively, it is a key requirement for web applications that offer a rich interactive user interface.

Here we will take only a brief look at CSS. For a more in-depth introduction, go [here](https://developer.mozilla.org/en-US/docs/Learn/CSS).

Let's dive right in and start with the following `index.html`:

```html
<div class="container">
   <h1>Hello <span class="name">there</span>!</h1>
   <form>
      <input type="text" name="name">
      <button type="submit">Submit</button>
   </form>
</div>
```

While the file should look mostly familiar, a new `div` element and some new `class` attributes were added. Similar to an invisible `span` inline element, a `div` element is an invisible "block" element. A block element takes up the entire width of the web page while an inline element is something like a word in a paragraph that can be broken across lines of text. The `class` attribute is used to match CSS rules with HTML elements (similar how we used IDs to identify elements with JavaScript).

As with JavaScript, the CSS of a document is usually defined in an extra `.css` file. But it can also be defined inline, which we will do here for simplicity:

```html
<style type="text/css">
   .container {
      max-width: 400px;
      margin: auto;
      font-family: "Fira Sans", sans-serif;
   }

   .name {
      font-weight: bold;
   }
</style>
<div class="container">
   <h1>Hello <span class="name">there</span>!</h1>
   <form>
      <input type="text" name="name">
      <button type="submit">Submit</button>
   </form>
</div>
```

The CSS rules of the `container` class will make the content of the page appear almost centered now. Also, the font family was changed from the default serif font to [Fira Sans](http://mozilla.github.io/Fira/) (if you have it) or another sans-serif font otherwise. Note that, in CSS, the reference to a class is defined with a prefixed `.`. There can be other types of references, too, e.g. for certain types of HTML elements:

```html
<style type="text/css">
   :root {
     --stable-green: #57ab1e;
   }

   .container {
      max-width: 400px;
      margin: auto;
      font-family: "Fira Sans", sans-serif;
   }

   .name {
      font-weight: bold;
   }

   input, button {
      border-radius: 2px;
   }

   input {
      padding: 5px 5px;
      border: 1px solid var(--stable-green);
   }

   button {
      padding: 6px 6px;
      background-color: var(--stable-green);
      border: none;
      color: white;
      cursor:  pointer;
   }
</style>
<div class="container">
   <h1>Hello <span class="name">there</span>!</h1>
   <form>
      <input type="text" name="name">
      <button type="submit">Submit</button>
   </form>
</div>
```

This looks a little prettier, doesn't it? Here you also see [CSS variables](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties) in use which are very useful to assign the same colors or other properties to different elements. But it's also quite obvious that CSS quickly is very verbose even for relatively small changes. It can also become hard to maintain once your website or web application gets large and there are many different elements and CSS classes to style. This is why it's a good idea to use a CSS framework most of the time. One popular choice is [Bootstrap](https://getbootstrap.com/). The framework already defines a set of standard styles and element classes that you can use without having to write any CSS yourself. Let's take a look at how this could work:

```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.2.0-beta1/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-0evHe/X+R7YkIZDRvuzKMRqM+OrBnVFBL6DOitfPri4tjfHxaWutUpFmBp4vmVor" crossorigin="anonymous">
<div class="container">
   <div class="row justify-content-md-center">
      <div class="col-md-auto">
         <h1>Hello <span class="fw-bold">there</span>!</h1>
         <form>
            <div class="mb-3">
               <input class="form-control" type="text" name="name">
            </div>
            <div class="mb-3">
               <button class="btn btn-primary" type="submit">Submit</button>
            </div>
         </form>
      </div>
   </div>
</div>
```

You see, no custom CSS any more, only the Bootstrap CSS file that is loaded from an external source. But now there are lots of new CSS classes and even a few invisible HTML elements that are required to style the page. Although styling and markup should be separated, this is not always possible in real life.

A framework (be it CSS, PHP or JavaScript) can be a big help if you are new to web development and have no strong (or any) opinion on how things should be done. If you follow the direction of the framework, you do most things right by default.

Next up we will have a closer look at the Laravel framework and see how the basics that you have learned so far are implemented there.

**Next lesson:** [A Full Laravel Application](/lesson-5)
