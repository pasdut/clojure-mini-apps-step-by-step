+++
title = "SQL Inspector : Clojure + JDBC + JavaFX Desktop application - Part 2"
author = ["Pascal Dutilleul"]
draft = false
+++

## Intro {#intro}

In part 2 we will develop the user interface.  We will ignore styling and just focus on the mechanics.  The result will be something usable (but with an awful look 'n feel).

I'll also want to demonstrate how the REPL development works step by step. Normally the functions are modified on the fly.  But as an educational tool, I will keep all subsequent versions in the _(comment ... )_ section.


## JavaFX {#javafx}

Now we can start with the actual User interface.

The GUI will be created by [JavaFX](https://openjfx.io/). Go ahead and download and install the binaries [from Gluon](https://gluonhq.com/products/javafx/).

Here is a nice [JavaFX Overview](https://jenkov.com/tutorials/javafx/overview.html).

On top of JavaFX we will use the clojure wrapper [cljfx](https://github.com/cljfx/cljfx).


## Step 04 - Let's show something {#step-04-let-s-show-something}


### 4.1 Add cljfx dependency {#4-dot-1-add-cljfx-dependency}

First add cljfx to the dependencies in _project.clj_.

```clojure
;; project.clj
:dependencies [[org.clojure/clojure "1.10.1"]
               [com.github.seancorfield/next.jdbc "1.2.772"]
               [com.microsoft.sqlserver/mssql-jdbc "10.2.0.jre11"]
               [cljfx "1.7.16"]
               ]
```

We can't use cljfx if we don't _require_ it. Thus add it in core.clj too.

```clojure
;;core.clj
(ns sqlinspector.core
  (:gen-class)
  (:require [next.jdbc :as jdbc]
            [cljfx.api :as fx])
  )
```

Now instruct _lein_ to load the dependencies. And to see if everything works as it should we start de REPL from _lein_ too.

```bash
~/sqlinspector$ lein deps
~/sqlinspector$ lein repl
Error in glXCreateNewContext, remote GLX is likely disabled
nREPL server started on port 37031 on host 127.0.0.1 - nrepl://127.0.0.1:37031
REPL-y 0.4.4, nREPL 0.7.0
Clojure 1.10.1
OpenJDK 64-Bit Server VM 11.0.11+9-Ubuntu-0ubuntu2.20.10
    Docs: (doc function-name-here)
          (find-doc "part-of-name-here")
  Source: (source function-name-here)
 Javadoc: (javadoc java-object-or-class-here)
    Exit: Control+D or (exit) or (quit)
 Results: Stored in vars *1, *2, *3, an exception in *e

sqlinspector.core=> (quit)
Bye for now!
```

(There is an _Error in glxCreateNewContext_ but I guess it might be because I'm running in WSL on Windows, so just ignore it...)

If you have a REPL open in your editor, don't forget to reload that REPL too.


### 4.2 Hello there {#4-dot-2-hello-there}

To get our feet wet we start with the [cljfx Hello World](https://github.com/cljfx/cljfx#hello-world) example.

```clojure
(fx/on-fx-thread
   (fx/create-component
    {:fx/type :stage
     :showing true
     :title "Cljfx example"
     :width 300
     :height 100
     :scene {:fx/type :scene
             :root {:fx/type :v-box
                    :alignment :center
                    :children [{:fx/type :label
                                :text "Hello from SQL Inspector"}]}}}))
```

If all went well you should get the following window:
![](/ox-hugo/si-step04-01.png)


### 4.3 Let's program interactively {#4-dot-3-let-s-program-interactively}

We want to develop the GUI interactively from within the REPL. The following is heavily based on the example [e12_interactive_development.clj](https://github.com/cljfx/cljfx/blob/master/examples/e12_interactive_development.clj) from the cljfx [examples](https://github.com/cljfx/cljfx/tree/master/examples).

The idea is whenever you changed a function, just call _(renderer)_ to redisplay the GUI.

You are supposed to execute the code that follows expression by expression, without the need to reload or recompile the complete application. Power to the REPL...

Let's see if we can display a label and a text-box.

```clojure
  ;; A state with some dummy data that we will display
  (def *state
    (atom {:table-filter "some filter data"}))


  (defn root-view [{{:keys [table-filter]} :state}]
    {:fx/type :stage
     :showing true
     :title "SQL inspector"
     :width 500
     :height 300
     :scene {:fx/type :scene
             :root {:fx/type :v-box
                    :children [{:fx/type :label
                                :text "Table filter:"}
                               {:fx/type :text-field
                                :text table-filter}]}}})

  (def renderer
    (fx/create-renderer
     :middleware (fx/wrap-map-desc (fn [state]
                                     {:fx/type root-view
                                      :state state}))))

  ;; Start watching for state changes and call the renderer.
  ;; This will show the window.
  (fx/mount-renderer *state renderer)

  ;; Whenever we update the state, the text-field should update too.
  (reset! *state {:table-filter "filter changed"})
```

Did you noticed the text-field showed the new value after you updated the state?

This is the resulting window:
![](/ox-hugo/si-step04-3-20.png)


### 4.4 Get to the bones {#4-dot-4-get-to-the-bones}

Now we are ready to build the skeleton of our application.

My motto is to let the user be as much as possible in control over the layout. Thus we immediately add a splitter with the [JavaFX SplitPane](https://jenkov.com/tutorials/javafx/splitpane.html), to let him decide how big he wants the tables part versus the columns part.

Here is a schema of what we will build in this step :
![](/ox-hugo/si-step04-04-10.png)

```clojure
  ;; The skeleton of the application
  (defn root-view [{{:keys [table-filter selected-table]} :state}]
    {:fx/type :stage
     :showing true
     :title "SQL inspector"
     :width 500
     :height 300
     :scene {:fx/type :scene
             :root {:fx/type :v-box
                    :children [{:fx/type :split-pane
                                :items [{:fx/type :v-box
                                         :children [{:fx/type :label
                                                     :text "Table filter:"}
                                                    {:fx/type :text-field
                                                     :text table-filter}
                                                    {:fx/type :label
                                                     :text "Tables:"}]}
                                        {:fx/type :v-box
                                         :children [{:fx/type :label
                                                     :text (str "Columns for table:"
                                                                selected-table)}]}]}]}}})
  ;; We updated the view, thus must re-render.
  (renderer)

  ;; The new root-view expects the key :selected-table in the state,
  ;; so add it to the state.
  ;; Because we renderer is watching *state (see step 4.3) the screen
  ;; is immediately updated.
  (reset! *state {:table-filter "some filter data xx"
                  :selected-table "t_my_table"})
```

And this is the window so far:

{{< figure src="/ox-hugo/si-step04-04-20.png" >}}

You might have noticed the splitter doesn't fill the complete window. Don't worry. For now, we are just working on the mechanics, not on the visuals.


### 4.5 Code cleanup {#4-dot-5-code-cleanup}

If you take a look at the _root-view_ function, there is a lot of nesting going on.  And I foresee this will gain some more complexity.

Therefore I decided to refactor this a little bit.  Let's move the rendering of the tables — the left side of the split-pane — and the rendering of the columns — the right side of the split-pane — in their own functions.

```clojure
  ;; Move tables and columns in their own function
  (defn tables-view [{:keys [table-filter]}]
    {:fx/type :v-box
     :children [{:fx/type :label
                 :text "Table filter:"}
                {:fx/type :text-field
                 :text table-filter}
                {:fx/type :label
                 :text "Tables:"}]})

  (defn columns-view [{:keys [selected-table]}]
    {:fx/type :v-box
     :children [{:fx/type :label
                 :text (str "Columns for table: " selected-table)}]})

  (defn root-view [{{:keys [table-filter selected-table]} :state}]
    {:fx/type :stage
     :showing true
     :title "SQL inspector"
     :width 500
     :height 300
     :scene {:fx/type :scene
             :root {:fx/type :v-box
                    :children [{:fx/type :split-pane
                                :items [{:fx/type tables-view
                                         :table-filter table-filter }
                                        {:fx/type columns-view
                                         :selected-table selected-table }]}]}}})
  ;; We updated the view, thus must re-render.
  (renderer)
```


## Step 05 - Run something {#step-05-run-something}

In the previous step 04, we've tried in the REPL to build our first version of the GUI.  In this step we'll use the stuff we learned.  Now we will update our application in such a way that _lein run_ will actually show our basic window.

First of all we'll delete the functions _tables-view, columns-view, root-view_ and _renderer_ from the _(comment ...)_ block and create them into the source code above the _(comment ...)_.

Second, we create a new function _initialize-cljfx_ to mount the state and the renderer.

And finally the  _-main_ is changed to call the initialization function. We can then run the application on the commandline via _lein run_. Or we could launch the application via the REPL by evaluating _(-main)_

```clojure
(defn initialize-cljfx []
  (fx/mount-renderer *state renderer))

(defn -main
  [& args]
  (initialize-cljfx))

(comment

  ;; run the application from the REPL
  (-main)
)
```

This is the last time you can see the different steps that I executed in the Repl. You can see these in the _(comment ...)_ block in Step 04.

Now, please forgive me, but keeping this backlog of code I executed in the REPL involves a lot of copy paste work from my part. Besides, I assume you now get the idea how REPL development actually works. And thus, for all subsequent steps, I'll only commit the final code.


## Step 06 - Handle something {#step-06-handle-something}

Up to our first event handler.

The goal for this step is to update the field _:table-filter_ in the _\*state_ whenever the user enters text in the text field.

Remember from the previous steps that cljfx is monitoring the state. And when something from the state has changed cljfx will update the window. To see this in action we'll temporary add a label to the _tables-view_ that shows the contents of the _\*state_  _:table-filter_.

```clojure
;; change this immediately in the code and evaluate that in the REPL
(defn tables-view [{:keys [table-filter]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text table-filter}
              {:fx/type :label
               :text "Tables:"}
              ;; temporary added
              {:fx/type :label
               :text (str ":table-filter contains " table-filter)}]})
```

And as we already know, after changing a view function we must call _renderer_.

```clojure
  ;; Whenever we changed the user interface we must rerender
  ;; This is something we will often do, so keep this in the comment
  (renderer)
```

_\*state_ is an atom, so we can give it a new value. Let's update :table-filter and see if cljfx picks it up.

```clojure
  (swap! *state assoc :table-filter "Blah blah")
```

Did you notice — after you evaluated the previous expression — that both the textbox and the label were updated and they both show _Blah blah_?

Of course we won't manually update the state when we want to use another filter.  Instead we'll use the  _on-text-changed_ handler of the text field to update the filter. In _cljfx_ we just need to add the keyword _:on-text-changed_ with as value a (anonymous) function.

```clojure
(defn tables-view [{:keys [table-filter]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text table-filter
               ;; handler
               :on-text-changed #(swap! *state assoc :table-filter %)}
              {:fx/type :label
               :text "Tables:"}
              ;; temporary added
              {:fx/type :label
               :text (str ":table-filter contains " table-filter)}]})
```

Now we are getting somewhere.  When you enter text in the textbox, the _:table-filter_ of the _\*state_ gets updated.  Which in turn resulted in an update of the label.

There is however a problem. The view function _tables-view_ now has to know the structure of _\*state_. In other words, the view function is coupled to the state.

Fortunately, cljfx let us also define an event handler as an arbitrary map. See the next step...


## Step 07 - Pure events in the view {#step-07-pure-events-in-the-view}

Let's use a map for the :on-text-changed event:

```clojure
(defn tables-view [{:keys [table-filter]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text table-filter
               ;; use a map
               :on-text-changed {:event/type :update-table-filter}}
              {:fx/type :label
               :text "Tables:"}
              {:fx/type :label
               :text (str ":table-filter contains " table-filter)}]})
```

The _tables-view_ function is now again a pure function without side effects.

These kind of events are handled asynchronous by _cljfx_.  We also need to provide a function to the renderer so it knows how to actually handles these map-events.

```clojure
(defn map-event-handler [event]
  ;; just print the event we get
  (println "Event Received : " event ))
```

And we must tell the renderer to use our mapping function:

```clojure
(def renderer
  (fx/create-renderer
   :middleware (fx/wrap-map-desc (fn [state]
                                   {:fx/type root-view
                                    :state state}))
   :opts {:fx.opt/map-event-handler map-event-handler}))
```

To see these changes, you need to execute _(-main)_ in the REPL.  If this doesn't work then try to refresh the REPL and then execute _(-main)_ followed by  _(renderer)_. If you had an existing _Sql Inspector_ window open, then first close it before calling the _(renderer)_

If everything went well, and you typed _abc_ in the text edit you will see the following output:

```clojure
Event Received :  {:event/type :update-table-filter, :fx/event a}
Event Received :  {:event/type :update-table-filter, :fx/event ab}
Event Received :  {:event/type :update-table-filter, :fx/event abc}
```

We found out the structure of the _event_ parameter we receive in _map-event-handler_.  Now use it.

I found out it is hard to get this reloaded in the REPL.  So I changed also the _println_ from outputting _"Event Received"_ to just _"Event:"_.  So we can see in the output that the latest handler is loaded.

```clojure
(defn map-event-handler [event]
  (println "Event : " event )
  (case (:event/type event)
    :update-table-filter  (swap! *state assoc :table-filter (:fx/event event))))
```

So just like before, close all open _Sql Inspector_ windows, re-evaluate _(-main)_ and _(renderer)_. And maybe restart the complete REPL when needed.

The end result now is the label is automatically updated whenever you type something in the filter text edit.

To recap how this event handling works :

-   The user types a character in the text edit.
-   That character is immediately shown in the text edit because of the OS Widget implementation.
-   That widget generates an event _:on-text-changed_.
-   We defined that the _:on-text-changed_ event should generate a _cljfx_ event  _:update-table-filter_.
-   _cljfx_ will then handle that event asynchronously by calling our _map-event-handler_ function.
-   That function will then finally update the  _:table-filter_ value of our _\*state_.
-   Which in turn will then update the _:text_ of both the text-edit and the temporary label.

There is one issue with this in the rare case when the user types extremely fast. I mean, when he types sooooo fast that _cljfx_ events are piled up.

Imagine the user types _"abc"_ really fast.  And _cljfx_ only starts to process the _:update-table-filter_ event for the _"a"_ after the user finished typing the _"abc"_.

What will happen is that the text edit first will contain _"abc"_ because the OS Widget shows immediately what he types.  And then a moment later this is changed to _"a"_ because of the processing of the _:update-table-filter_ event. In the best case the user will experience some lagging, in the worst case he will miss some characters. (_cljfx_ got some ideas from _re-frame_ and here is the same issue explained : [Why is my input field laggy?](https://day8.github.io/re-frame/FAQs/laggy-input/))

Fortunately the solution is simple: inform _cljfx_ to handle this evens synchronously.  Just notice this is only needed for text inputs.

```clojure
(defn tables-view [{:keys [table-filter]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text table-filter
               :on-text-changed {:event/type :update-table-filter
                                 ;; handle this synchronously
                                 :fx/sync true}}
              {:fx/type :label
               :text "Tables:"}
              {:fx/type :label
               :text (str ":table-filter contains " table-filter)}]})
```


## Step 08 - Pure event handlers {#step-08-pure-event-handlers}

The event handler _map-event-handler_ that processes the event _:update-table-filter_ is still coupled to (and updates) the _\*state_ atom. This makes testing hard because we must mock the state in our tests.

It would be handy if the event handler could be just data in, data out. See the cljfx documentation [Event handling on steroids](https://github.com/cljfx/cljfx#event-handling-on-steroids).

We have 2 things to do.

First we need to get the _state_ as input to the event handler. The _cljfx_ function _wrap-co-effects_ does this for us and will pass the dereferenced _state_ to our handler.  The state will be inserted in the _event_ map.

```clojure
(defn event-handler [event]
  (println "event-handler:" event))

;; Notice this is "def" and not "defn" as wrap-co-effects
;; returns a function.
(def map-event-handler
    (-> event-handler
        (fx/wrap-co-effects
         {:state (fx/make-deref-co-effect *state)})
        ))
```

This is again a modification of the cljfx initialization stuff. If you're still in the REPL then first close existing SQL Inspector windows. Then evaluate the complete source and execute _(-main)_.

If we now type _"abc"_ in the search filter we get the following output:

```shell
event-handler: {:event/type :update-table-filter, :fx/sync true, :fx/event a, :state {:table-filter , :selected-table }}
event-handler: {:event/type :update-table-filter, :fx/sync true, :fx/event ab, :state {:table-filter , :selected-table }}
event-handler: {:event/type :update-table-filter, :fx/sync true, :fx/event abc, :state {:table-filter , :selected-table }}
```

Notice we now have a key _:state_, so we no longer have to dereference the actual state.  It is just data _in_.

Next step is to return the updated state.

```clojure
(defn event-handler [e]
  (println "event-handler:" e)
  (let [{:keys [event/type fx/event state]} e]
    (case type
      :update-table-filter {:state (assoc state :table-filter event)})))
```

This is the data _out_. And it is easy to verify if the state (not the real state, but the :state in the returning map) is updated as expected.

```clojure
(event-handler
 {:event/type :update-table-filter
  :fx/event "xyz"
  :state {:table-filter "abc"}})
=>{:state {:table-filter "xyz"}}
```

The last thing we have to do now is to use the value returned from the event to update the actual _State_ atom.  For this we use _wrap-effects_.

```clojure
;; Notice this is "def" and not "defn" as wrap-co-effects and wrap-effects
;; return a function.
(def map-event-handler
    (-> event-handler
        (fx/wrap-co-effects
         {:state (fx/make-deref-co-effect *state)})
        (fx/wrap-effects
         {:state (fn [state _] (reset! *state state))})))
```

When we now type something in the text box, the actual state will be updated.  And this causes the label to show also show what we have typed.


## End of Part 2 {#end-of-part-2}

This concludes the second part in the series.

We created a basic window, learned  and discovered how we can react to events.  With these building blocks in our toolbox we are ready


## <span class="org-todo todo TODO">TODO</span> : {#853ae9}

Subscriptions and context[GitHub - cljfx/cljfx: Declarative, functional and extensible wrapper of JavaF...](https://github.com/cljfx/cljfx#subscriptions-and-contexts)
