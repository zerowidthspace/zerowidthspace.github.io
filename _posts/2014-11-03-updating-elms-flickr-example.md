---
layout: post
title: Updating Elm's Flickr Example
date: 2014-11-03 15:00:00
disqus: y
tags:
- elm
- programming
- tutorial
---

Recently Elm received a new library called [elm-html](http://elm-lang.org/blog/Blazing-Fast-Html.elm), which allows for external HTML/CSS to be used with Elm code. I very much prefer this style of writing programs as it allows existing HTML/CSS designs and talent to be used, lowers the barrier for entry into Elm programming. On top of that, elm-html supports updating the DOM via reference-equals, a strategy which allows for highly performant UI, at least an order of magnitude faster than conventional MVC frameworks. Using elm-html comes with caveats, notably the inability to use the `Graphics.Input` interfaces. In this post I’ll update [Elm’s Flickr image search example](http://elm-lang.org/edit/examples/Intermediate/Flickr.elm) to use elm-html.

Since we’re only using Elm to build the JavaScript now, the first thing we need to do is build an HTML container for our program. This container links out to a CSS file (which we’ll create later), the JS file (which comes from the Elm compiler), and links the Elm program to the document using the `Elm.fullscreen` method.

{% highlight html %}
<!DOCTYPE html>
<html>
    <head>
        <title>Flickr Image Search</title>
        <link rel="stylesheet" type="text/css" href="flickr.css" />
    </head>
 
    <body>
        <script type="text/javascript" src="flickr.js"></script>
        <script type="text/javascript">
            Elm.fullscreen(Elm.Flickr);
        </script>
    </body>
</html>
{% endhighlight %}

Next, let’s rebuild the markup in elm-html. When we import the elm-html library, we need to be careful to disambiguate `Graphics.Input.input` from `Html.Tags.input`. I prefer to import `Html.Tags` fully, and qualify `Graphics.Input`, but ultimately the choice is up to you.

{% highlight haskell %}
-- Add this at the beginning, so the Elm.fullscreen can find what to embed
module Flickr where
 
-- import Graphics.Input (Input, input)
-- import Graphics.Input.Field as Field
 
import Graphics.Input as Input
 
import Html (..)
import Html.Attributes (..)
import Html.Events (..)
import Html.Tags (..)
import Html.Optimize.RefEq as Ref
{% endhighlight %}

Get rid of the old `scene` and `main` functions. Instead we’ll build our HTML markup as modular *views* (cf. [Architecture in Elm](https://gist.github.com/evancz/2b2ba366cae1887fe621)) which is also the technique used in the Elm [TodoMVC example](https://github.com/evancz/elm-todomvc/blob/master/Todo.elm).

{% highlight haskell %}
tag : Input.Input String
tag = Input.input ""
 
searchView : Html
searchView =
    div
        [ class "searchbar" ]
        [ input [ type' "text", placeholder "Flickr Instant Search", on "input" getValue tag.handle identity ] [] ]
 
resultView : Maybe String -> Html
resultView imgSrc =
    case imgSrc of
        Just source -> div [ class "result" ] [ img [ src source ] [] ]
        Nothing -> div [ class "result" ] []
 
view : Maybe String -> Html
view imgSrc =
    div
        [ class "container" ]
        [ searchView
        , resultView imgSrc
        ]
 
scene : (Int, Int) -> Maybe String -> Element
scene (w, h) imgSrc = toElement w h (view imgSrc)
 
main : Signal Element
main = scene <~ Window.dimensions
              ~ getSources (dropRepeats tag.signal)
{% endhighlight %}

In elm-html, HTML nodes are of type `Html`, and can be built using the node function, whose signature is

```
node : String -> [Attribute] -> [Html] -> Html
```

For example:

{% highlight haskell %}
node "ul" [ class "container" ] [ node "li" [] [ text "Item 1" ], node "li" [] [ text "Item 2" ] ]
 
-- Generates:
-- <ul class="container">
--   <li>Item 1</li>
--   <li>Item 2</li>
-- </ul>
{% endhighlight %}

Furthermore, `Html.Tags` contains aliases such as `div = node "div"` to make the Elm markup more seamless. In the code given, I’ve divided the views into the upper search bar portion, and the lower results portion, both of which are container in `div.container`. The scene function then just converts the Html into an Element ready to embed.

Note also the change in the tag signal. As we’re no longer using `Graphics.Input.Field`, I’ve converted the signal into a plain String signal. Linking this signal to our searchbar requires that we use the on function, which you can read more about [in the documentation](http://library.elm-lang.org/catalog/evancz-elm-html/0.3/Html). A few more minor changes to accommodate our signal change, and the conversion is complete. Compile using the following:

```
elm --make --only-js --bundle-runtime --build-dir=. flickr.elm
```

Open up `flickr.html` in your browser and you should see the exact same functionality as the original demo, only now built with elm-html! As a bonus, you can also now style the HTML markup using regular CSS in `flickr.css`.

Full code download here.