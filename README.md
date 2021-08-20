# AJAX in Rails

---

## Keep the page context

Let's fix the *scrolling back to the top* issue using JavaScript views in Rails.


### Render JS views

Rails controllers can render different formats based on the request’s “Accept” header:

```rb
# app/controllers/reviews_controller.rb
# [...]
  def create
    # ...
    if @review.save
      respond_to do |format|
        format.html { redirect_to restaurant_path(@restaurant) }
        format.js
      end
    else
    # ...
  end
```

It will respond to requests that have `Accept: text/javascript` in their headers with a `js.erb` view.


### Implement the `create.js.erb` view

Add some JavaScript to insert the review partial at the beginning of the `#reviews` div.

```js
document.getElementById("reviews").insertAdjacentHTML("afterBegin", "<%=j render @review %>")
document.getElementById("reviews-count").innerText = "<%= pluralize @restaurants.reviews.size, 'review' %>"
```

*Make sure to add `id="reviews-count"` to the reviews counter.*

### What are the `js.erb` views?

Don't think of the `js.erb` files as actual views (because we're not reloading the page or rendering new HTML). Think of it simply as **some JS code you run on the current page** instead of sending HTML at the end of your controller action.

---

# Creating JavaScript components with Stimulus


## Simple Component: Collapsible Form

Let's create a "collapsible form" component for the `reviews#new` form, that expands when we click on a button.


Let's do it in pure **JavaScript**


### Stimulus JS

A JavaScript [library](https://github.com/stimulusjs/stimulus) that pairs well with Turbolinks.

Read the [Handbook](https://stimulusjs.org/handbook/origin)


### Stimulus in Rails
Quick setup in rails:

```shell
rails webpacker:install:stimulus
```

Check your app/javascript/packs/application.js


### Stimulus Controller

```shell
touch app/javascript/controllers/collapsible_controller.js
```
```js
import { Controller } from "stimulus";

export default class extends Controller {
  connect() {
    console.log(`Hello from the ${this.identifier} controller!`);
  }
}
```


### Data-controller

Connect the component to the `collapsible` controller by adding a `data-controller` attribute.

```erb
<!-- app/views/restaurants/show.html.erb -->

<div data-controller="collapsible"> <!-- create a container for our new component -->
  <button class="btn btn-outline-primary">Leave a review</button>

  <%= simple_form_for([ @restaurant, @review ], remote: true) do |f| %>
    <!-- [...] -->
  <% end %>
</div>
```
Set the `data-controller` in a div that contains both:
- the element listening to an event (the button)
- the element you want to update (the form)


### Data-target

`data-target` is the equivalent of `document.querySelector`

```erb
<%= simple_form_for([ @restaurant, @review ],
                    html: { data: { collapsible_target: 'form' } },
                    remote: true) do |f| %>

<!-- Simple form will generate a form tag like this: -->
<form action="..." data-collapsible-target="form" ... >
```

Syntax: `data-<controller-name>-target="targetName"`


### Targets

```js
import { Controller } from "stimulus";

export default class extends Controller {
  static targets = [ 'form' ];

  connect() {
    console.log(this.formTarget);
  }
}
```

`this.formTarget` returns the first one, `this.formTargets` returns them all


### Connect

Set the initial state of the form in the `connect()` action.

```js
export default class extends Controller {
  static targets = [ 'form' ];

  connect() {
    this.formTarget.style.height = "0"
    this.formTarget.style.overflow = "hidden"
    this.formTarget.style.transition = "height 0.2s ease-in"
  }
}
```


### Data-action

Listening to the `click` event on the button (`addEventListener`):

```erb
<!-- app/views/restaurants/show.html.erb -->

<div data-controller="collapsible">
  <button class="btn btn-outline-primary"
          data-action="click->collapsible#expandForm">Leave a review</button>
  <!-- [...] -->
</div>
```

Syntax: `event->controller-name#actionName`


### Action

```js
import { Controller } from "stimulus";

export default class extends Controller {
  static targets = [ 'form' ];
  // ...
  expandForm(event) {
    console.log(event);
  }
}
```

Let’s expand the form!


### Settings

Use Stimulus values to add settings to your component

```erb
  <div data-controller="collapsible" data-collapsible-height-value="150px">
```

```js
export default class extends Controller {
  static targets = [ 'form' ];
  static values = { height: String }; // <-- Add this
  // ...
  expandForm(event) {
  this.formTarget.style.height = this.heightValue
  event.currentTarget.remove() // Remove the button after expanding the form
}
}
```


## More examples (optional)


### New Action: Submit on Enter

```js
export default class extends Controller {
  // ...
  submitOnEnter(event) {
    if (event.key === 'Enter' && !event.shiftKey) {
      event.preventDefault()
      this.formTarget.style.height = "0"
      Rails.fire(this.formTarget, 'submit')
    }
  }
}
```


We use `Rails.fire` (instead of `form.submit`) in order to submit the form *remotely*.

*Make sure to add the `rails-ujs` plugin to Webpack in your `environment.js` file.*

```js
environment.plugins.prepend('Provide',
  new webpack.ProvidePlugin({
    // ...
    Rails: ['@rails/ujs']
  })
);
```


### Adding the `data-action` in the view

```erb
<div data-controller="collapsible" data-expanded-height=150>
  <!-- [...] -->
  <%= f.input :content, as: :text,
    label: false,
    input_html: {
      placeholder: 'Press enter to submit your review.',
      data: { action: 'keydown->collapsible#submitOnEnter' }
    } %>
</div>
```


## Stimulus takeaways

- the `data-controller` should wrap all the Stimulus elements from the component
- `querySelector` is replaced by `data-<controller-name>-target="targetName"`
- `querySelector` + `addEventListener` is replaced by `data-action`


### Stimulus syntax recap

- `data-controller="controller-name"`
- `data-<controller-name>-target="targetName"`
- `data-action="event->controller-name#actionName"`

And a bit more advanced:
- `data-<controller-name>-<value-name>-value="<value>"`

---

## HAPPY AJAXIFICATION!

Note:
To push this app on Heroku yourself, if you have cloned the lectures repository, navigate to the root of the repo and run the following lines:
```shell
heroku create --remote heroku_restaurants-ajaxified
git subtree push --prefix lectures/restaurants-ajaxified heroku_restaurants-ajaxified main
```