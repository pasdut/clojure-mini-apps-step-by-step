+++
title = "SQL Inspector : Clojure + JDBC + JavaFX Desktop application - Part 3"
author = ["Pascal Dutilleul"]
draft = false
+++

## Step 09 - Show the tables {#step-09-show-the-tables}

Now it is time to visualize the tables.

For this we extend the state with a new key _:tables_.  And we update _(-main)_ to actually retrieve the tables.

```clojure
(def *state
  (atom {:table-filter ""
         :selected-table ""
         :tables []}))

(defn -main
  [& args]
  (swap! *state assoc :tables (retrieve-all-tables))
  (initialize-cljfx))
```

Do not forget to re-execute _(-main)_ to be sure the tables are actually loaded in _state_ for your current REPL. And in some cases you must completely reload the file too. This is because the event _:update-table-filter_ messes with the state and the _:tables_ are lost if you don't reload the full file after modifying the _state_.

Just to be sure, let's check whether the _state_ now contains the tables:

```clojure
sqlinspector.core> *state
;; => #<Atom@1e3168e8:
  {:table-filter "",
   :selected-table "",
   :tables
   [{:table_name "t_address_type",
     :create_date #inst "2022-03-25T21:02:17.320000000-00:00",
     :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
    {:table_name "t_customer",
     :create_date #inst "2022-03-25T21:02:13.880000000-00:00",
     :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
    {:table_name "t_customer_address",
     :create_date #inst "2022-03-25T21:16:14.677000000-00:00",
     :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}]}>
```

(Remark: I also fixed a typo in _retrieve-all-tables_.  The field _modify_date_ was incorrectly aliased to _table_name_)

Let's display a _table view_ that contains the table names.

We could have used a _list view_ because we only have one column.  However we might add additional columns in the future like creation date or number of records in the table. Therefore we'll stick with _table view_.

In the cljfx [examples](https://github.com/cljfx/cljfx/tree/master/examples) folder, the example [e27_selection_models](https://github.com/cljfx/cljfx/blob/master/examples/e27_selection_models.clj) teaches us how we can display a _table view_.

We need an event to know when the selection is changed. That's why we use _cljfx.ext.list-view/with-selection-props_, just like in the example.

First of all, we must add a new _require_:

```clojure
(ns sqlinspector.core
  (:gen-class)
  (:require [next.jdbc :as jdbc]
            [cljfx.api :as fx]
            [cljfx.ext.table-view :as fx.ext.table-view])
  )
```

Next, we update the _tables-view_ function to actually create the table-view. It is yet unclear to me what data I get.  Therefore I add a _println_ and return some dummy text.

```clojure
(defn tables-view [{:keys [table-filter tables]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text table-filter
               ;; use a map
               :on-text-changed {:event/type :update-table-filter
                                 :fx/sync true}}
              {:fx/type :label
               :text "Tables:"}
              ;; temporary added
              {:fx/type :label
               :text (str ":table-filter contains " table-filter)}
              {:fx/type fx.ext.table-view/with-selection-props
               :props {:selection-mode :single}
               :desc {:fx/type :table-view
                      :columns [{:fx/type :table-column
                                 :text "Tablename"
                                 :cell-value-factory identity
                                 :cell-factory {:fx/cell-type :table-cell
                                                :describe (fn [x]
                                                            (println "Data for the cell Tablename is:" x)
                                                            {:text "DUMMY TEXT"})}}]
                      :items tables }}]})
```

We now have an extra parameter _tables_, se we must pass that from the _root-view_ to _tables-view_:

```clojure
(defn root-view [{{:keys [table-filter selected-table tables]} :state}]
  {:fx/type :stage
   :showing true
   :title "SQL inspector"
   :width 500
   :height 300
   :scene {:fx/type :scene
           :root {:fx/type :v-box
                  :children [{:fx/type :split-pane
                              :items [{:fx/type tables-view
                                       :table-filter table-filter
                                       :tables tables}
                                      {:fx/type columns-view
                                       :selected-table selected-table}]}]}}})
```

After calling _(renderer)_ we see the result of the _print_ statements:

```clojure
Data for the cell Tablename is: {:table_name t_address_type, :create_date #inst "2022-03-25T21:02:17.320000000-00:00", :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
Data for the cell Tablename is: {:table_name t_customer, :create_date #inst "2022-03-25T21:02:13.880000000-00:00", :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
Data for the cell Tablename is: {:table_name t_customer_address, :create_date #inst "2022-03-25T21:16:14.677000000-00:00", :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
```

It seems we get the table data as parameter to the _describe_ function of the _cell-factory_. Let's change parameter _x_ to something meaningful, like _table-data_. For now we just visualize the _:table_name_. And lets comment out the _println_ statement:

```clojure
:cell-factory {:fx/cell-type :table-cell
               :describe (fn [table-data]
                           #_(println "Data for the cell Tablename is:" table-data)
                           {:text (:table_name table-data) })}
```

After _(renderer)_ we get the following result:

{{< figure src="/ox-hugo/si-step09-10.png" >}}


## Step 10 - Context {#step-10-context}

For now, the tables in the list are not yet filtered.

However, before we fix this, let's first introduce the concept of _context_.  See explanation in the official cljfx documentation : [Subscriptions and Context](https://github.com/cljfx/cljfx#subscriptions-and-contexts).

The idea is that the views shouldn't know the full structure of the state. Instead a view should subscribe to a piece of data. Additionally, values can be cached so a view will only render when the actual value it depends on is changed. An example: the user keeps adding characters to the filter. If the list of filtered tables doesn't change after updating the filter text, the view won't be rendered again.


### 10.1 Add _core.cache_ {#10-dot-1-add-core-dot-cache}

Whenever we use _cljfx contexts_ we must add the _core.cache_ dependency.

Update _project.clj_

```clojure
;; project.clj
:dependencies [[org.clojure/clojure "1.10.1"]
                 [com.github.seancorfield/next.jdbc "1.2.772"]
                 [com.microsoft.sqlserver/mssql-jdbc "10.2.0.jre11"]
                 [cljfx "1.7.16"]
                 [org.clojure/core.cache "0.7.1"]  ;;--- ADDED ---
                 ]
```

Add to _:require_ in _core.clj_:

```clojure
;; core.clj
(ns sqlinspector.core
  (:gen-class)
  (:require [next.jdbc :as jdbc]
            [cljfx.api :as fx]
            [cljfx.ext.table-view :as fx.ext.table-view]
            [clojure.core.cache :as cache])  ;;--- ADDED ---
  )
```

Don't forget to restart your REPL to load the new module and then also reload the complete _core.clj_ file in your REPL.


### 10.2 Start using Context {#10-dot-2-start-using-context}

Update the source to use _context_.

First of all, the _state_ is now a _context_:

```clojure
(def *state
  (atom (fx/create-context  ;;--- ADDED ---
         {:table-filter ""
          :selected-table ""
          :tables []}
         cache/lru-cache-factory)))  ;;--- ADDED ---
```

Add some middle ware to pass the _context_ through the _option_ map.

```clojure
(def renderer
  (fx/create-renderer
   :middleware (comp ;;--- ADDED --- middleware is now a composition of multiple functions
                ;; Pass context to every lifecycle as part of option map
                fx/wrap-context-desc ;;--- ADDED ---
                (fx/wrap-map-desc (fn [_]{:fx/type root-view})))
   :opts {:fx.opt/map-event-handler map-event-handler
          :fx.opt/type->lifecycle #(or (fx/keyword->lifecycle %);;--- ADDED ---
                                       ;; For functions in ':fx/type' values, pass
                                       ;; context from option map to these functions
                                       (fx/fn->lifecycle-with-context %))}));;--- ADDED ---
```

The _root-view_ no longer needs to pass data to _tables-view_ and _columns-view_. Heck, _root-view_ can just ignore its own parameters

```clojure
(defn root-view [_] ;;--- IGNORE PARAMETER ---
  {:fx/type :stage
   :showing true
   :title "SQL inspector"
   :width 500
   :height 300
   :scene {:fx/type :scene
           :root {:fx/type :v-box
                  :children [{:fx/type :split-pane
                              :items [{:fx/type tables-view} ;;--- NO MORE PARAMETER PASSING ---
                                      {:fx/type columns-view}]}]}}});;--- NO MORE PARAMETER PASSING ---
```

The function _columns-view_ no longer receives the _selected-table_ as parameter. Instead a context is received.  The  _columns-view_ should now extract the _selected-table_ value from the _context_ via the _fx/sub-val_ function.

```clojure
(defn columns-view [{:keys [fx/context]}] ;;--- PARAMETER NOW CONTAINS CONTEXT ---
  {:fx/type :v-box
   :children [{:fx/type :label
               :text (str
                      "Columns for table: "
                      (fx/sub-val context :selected-table))}]}) ;;--- USE "sub-val" ---
```

The _tables-view_ function must also be converted to using context.

```clojure
(defn tables-view [{:keys [fx/context]}];;--- PARAMETER NOW CONTAINS CONTEXT ---
  {:fx/type :v-box
   :children [{:fx/type :label
               :text "Table filter:"}
              {:fx/type :text-field
               :text (fx/sub-val context :table-filter);;--- USE "sub-val" ---
               ;; use a map
               :on-text-changed {:event/type :update-table-filter
                                 :fx/sync true}}
              {:fx/type :label
               :text "Tables:"}
              ;; temporary added
              {:fx/type :label
               :text (str
                      ":table-filter contains "
                      (fx/sub-val context :table-filter))};;--- USE "sub-val" ---
              {:fx/type fx.ext.table-view/with-selection-props
               :props {:selection-mode :single}
               :desc {:fx/type :table-view
                      :columns [{:fx/type :table-column
                                 :text "Tablename"
                                 :cell-value-factory identity
                                 :cell-factory
                                 {:fx/cell-type :table-cell
                                  :describe (fn [table-data]
                                              #_(println "Data for the cell Tablename is:" table-data)
                                              {:text (:table_name table-data) })}}]
                      :items (fx/sub-val context :tables)}}]});;--- USE "sub-val" ---
```

The function _(-main)_ loads all the table names in the _state_.  This _state_ is now a _context_ so we must use _fx/swap-context_ to load all tables.

```clojure
(defn -main
  [& args]
  (reset! *state (fx/swap-context @*state assoc :tables (retrieve-all-tables)))
  (initialize-cljfx))
```

At this point of the refactoring to use _context_ we can already execute _(-main)_ and see if the changes work. Normally you should see all table names, the same as the screenshot from the previous step 09.

Remember, we didn't update the event handling yet. So you'll probably get an error as soon as you type something in the text field!

Let's manually set the filter with _fx/swap-context_ and see if this works...

```clojure
(reset! *state (fx/swap-context @*state assoc :table-filter "Blah blah"))
```

Executing the previous line in the REPL should update the search filter in the application.

The _event-handler_ function should also update the context via _swap-context_

```clojure
(defn event-handler [e]
  (println "event-handler:" e)
  (let [{:keys [event/type fx/event fx/context]} e] ;;--- use fx/context ---
    (case type
      :update-table-filter {:context (fx/swap-context context assoc :table-filter event)})));;--- USE "swap-context" ---
```

Now we need to pass the context to the event handler as a co-effect.
And, based on [examples/e18_pure_event_handling.clj](https://github.com/cljfx/cljfx/blob/master/examples/e18_pure_event_handling.clj), we also define 2 effects _:context_ and _:dispatch_. If these keys are returned by the event handler, the corresponding effect will be executed. See also : [cljfx doc : Event handling on steroids](https://github.com/cljfx/cljfx#event-handling-on-steroids)

```clojure
(def map-event-handler
    (-> event-handler
        (fx/wrap-co-effects
         ;;--- Pass the deref state as :fx/context to the event-handler  ---
         {:fx/context (fx/make-deref-co-effect *state)})
        (fx/wrap-effects
         ;;--- what to do if event handler returns :context or :dispatch---
         {:context (fx/make-reset-effect *state)
          :dispatch fx/dispatch-effect})))
```

Remark that we changed the event-handler, which is a crucial part of the application.  These changes are not easily picked up by the REPL.  It is important to close the running REPL, restart it and reload the source.  Then you can execute the _(-main)_ function to start the application.

The end result is we simplified the parameters to the view functions by passing only the _context_.  With the additional bonus that the values are cached and a view is only regenerated when the value is actually changed.


## Step 11 - Apply filtering {#step-11-apply-filtering}

Now we can begin to filter the tables list.

We could do the filtering directly in _tables-view_. However, I'd rather keep data manipulation out of the view logic.

Let's try if we get the filtering to work, without using cljfx specifics. We opt for a case insensitive search. As a matter of test we try to get all tables that matches the search text _"CUST"_. Execute the following in the REPL to see if we get the expected result.

```clojure
(let [tables (retrieve-all-tables)
        table-filter "CUST"
        table-filter-pattern (re-pattern (str "(?i).*" table-filter ".*"))]
    (filter #(re-matches table-filter-pattern (:table_name %)) tables))
;; => ({:table_name "t_customer",
;; =>   :create_date #inst "2022-03-25T21:02:13.880000000-00:00",
;; =>   :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"}
;; =>  {:table_name "t_customer_address",
;; =>   :create_date #inst "2022-03-25T21:16:14.677000000-00:00",
;; =>   :modify_date #inst "2022-03-25T21:16:14.677000000-00:00"})
```

It seems to be working.

Now we can create a subscription function. (See explanation in the official cljfx documentation : [Subscriptions and Context](https://github.com/cljfx/cljfx#subscriptions-and-contexts))

```clojure
(defn subs-filtered-tables
  "Returns all tables that satisfy to the search text."
  [context]
  (let [tables (fx/sub-val context :tables)
        table-filter (fx/sub-val context :table-filter)
        table-filter-pattern (re-pattern (str "(?i).*" table-filter ".*"))]
    (filter #(re-matches table-filter-pattern (:table_name %)) tables)))
```

Rest us to modify the _tables-view_ function to use that subscription via _fx/sub-ctx_

```clojure
(defn tables-view [{:keys [fx/context]}]
  {:fx/type :v-box
   :children [;; ...
              {:fx/type fx.ext.table-view/with-selection-props
               :props {:selection-mode :single}
               :desc {:fx/type :table-view
                      :columns [
                                 ;; ...
                                ]
                      :items (fx/sub-ctx context subs-filtered-tables)}}]}) ;;--- USE SUBSCRIPTION ---
```

If you now evaluate _(renderer)_ in the REPL you'll see the filtering is now working.


## Step 12 - Table selection {#step-12-table-selection}


### 12.1 Select a table from the list {#12-dot-1-select-a-table-from-the-list}

Before we can show the columns, we must know what table is clicked on. We can do this by responding to the _:on-selected-item-changed_ event. Let's fire the event _:select-table_ as a responce.

```clojure
(defn tables-view [{:keys [fx/context]}]
  {:fx/type :v-box
   :children [;; ...
              {:fx/type fx.ext.table-view/with-selection-props
               :props {:selection-mode :single
                       :on-selected-item-changed {:event/type :select-table}} ;; NEW EVENT
               ;; ...
               }]})
```

And handle that event appropriately.

```clojure
(defn event-handler [e]
  (println "event-handler:" e)
  (let [{:keys [event/type fx/event state]} e]
    (case type
      :update-table-filter {:state (fx/swap-context state assoc :table-filter event)}
      ;; NEW EVENT
      :select-table {:state (fx/swap-context state assoc :selected-table (:table_name event))})))
```

Because we updated the event-handler we must unfortunately quit and restart the REPL to have it in effect.
But once the app is up and running again, whenever we select a table that table name appears on the  columns header.
![](/ox-hugo/si-step12-10.png)


### 12.2 Multi method event handling {#12-dot-2-multi-method-event-handling}

The function _event-handler_ is on its way to become a huge monster _case_ statement. (Yes I hear you, 2 clauses is not really too big, I know)

Let's refactor it via a multi methods.

```clojure
(defmulti event-handler :event/type)

(defmethod event-handler :default [event]
  (println "Unhandled event")
  (prn event)
  )

(defmethod event-handler :update-table-filter [{:keys [state fx/event]}]
  (println ":update-table-filter --->" event)
  {:state (fx/swap-context state assoc :table-filter event)})

(defmethod event-handler :select-table [{:keys [state fx/event]}]
  (let [table-name (:table_name event)]
    (println ":select-table --> " table-name)
    {:state (fx/swap-context state assoc :selected-table table-name)}))
```

Once again, we must restart the REPL and reload the source.  However it is the last time we need a full restart of the REPL. For some reason, updating a branch on a multimethod gets immediately picked up by the REPL. You can see this if you change the text in the _println_, re-evaluate that _defmethod_, and bam, it has effect.


### 12.3 Auto table selection {#12-dot-3-auto-table-selection}

It would be nice if the first table of the list will be selected automatically whenever the filter is changed.

Say we have an empty filter and we clicked the first table to select it. On the right side we see _Columns for table: t_address_type_.
![](/ox-hugo/si-step12-30.png)

If we now type _"cus"_ in the _table filter_, the table _t_address_type_ disappears from the table view. However the current selected table (see green) still remains _t_address_type_.

{{< figure src="/ox-hugo/si-step12-31.png" >}}

What we actually want is to select automatically the first table whenever the table list is changed.

Adhering to the single responsibility principle we create a new event _select-visible-table_. That event will be fired by the _update-table-filter_. This is achieved because we return the key _:dispatch_.

```clojure
;;--- NEW EVENT ---
(defmethod event-handler :select-visible-table [{:keys [fx/context fx/event]}]
  (println ":select-visible-table "))

(defmethod event-handler :update-table-filter [{:keys [fx/context fx/event]}]
  (println ":update-table-filter --->" event)
  {:context (fx/swap-context context assoc :table-filter event)
   :dispatch {:event/type :select-visible-table}}) ;;--- EMIT THE NEW EVENT ---
```

If we now change the table filter (by typing in the textbox), the text _":select-visible-table"_ is printed on the output.

The next step is to create a subscription function that actually returns the first table.

```clojure
(defn subs-visible-table-to-select
  "Return the table from the visible table list that must be selected"
  [context]
  (let [tables (fx/sub-ctx context subs-filtered-tables)]
    (first tables)))
```

When the filter is empty, the first table should be _t_address_type_. Let's check this by executing the following code in the repl:

```clojure
  (reset! *state (fx/swap-context @*state assoc :table-filter ""))
  (println (fx/sub-ctx @*state subs-visible-table-to-select))
  ;;==> {:table_name t_address_type, :create_date ...}
```

And when the filter is _cus_, the first table should be _t_customer_:

```clojure
  (reset! *state (fx/swap-context @*state assoc :table-filter "cus"))
  (println (fx/sub-ctx @*state subs-visible-table-to-select))
  ;;==> {:table_name t_customer, :create_date ...}
```

Woo-hoo, it works!  Let's incorporate this in the _select-visible-table_ event.

```clojure
(defmethod event-handler :select-visible-table [{:keys [fx/context fx/event]}]
  (let [table-name (:table_name (fx/sub-ctx context subs-visible-table-to-select))]
    (println ":select-visible-table --> table = " table-name)
    {:context (fx/swap-context context assoc :selected-table table-name)}))
```

If we now play with the table-filter the first table is selected in the state. This means whatever filter you type, the right side _Columns for table: xxx_ always shows the first table of the visible tables.


### 12.4 Show the selected table {#12-dot-4-show-the-selected-table}

We have a slightly cosmetic problem.  If we use the filter _cus_ the first matching table is selected.  In this case it is _t_customer_, which is visible in the header on the right. However, this is not visible in the table view on the left.

{{< figure src="/ox-hugo/si-step12-40.png" >}}

From the example [examples/e27_selection_models.clj](https://github.com/cljfx/cljfx/blob/master/examples/e27_selection_models.clj) it seems we should set the property _:selected-item_. The documentation in [cljfx source ext/table_view.clj](https://github.com/cljfx/cljfx/blob/master/src/cljfx/ext/table_view.clj), says that we should pass the same value as from table-view's items.

Now we have an issue: we need the complete table data to select a row in the table-view.  But we store only the table_name in _:selected-table_ in our state.

We could write a new subscription that returns the full table data for the selected table name. However, I love the KISS principle.  And I see no particular reason to save only the table_name in _:selected-table_.  From now on, we'll store the complete table data.

Thus we need to change the handler for _:select-table_

```clojure
(defmethod event-handler :select-table [{:keys [fx/context fx/event]}]
  (println ":select-table --> " (:table-name event))
  {:context (fx/swap-context context assoc :selected-table event)});;--- SAVE FULL EVENT(=TABLE DATA) ---
```

And also the _:select-visible-table_ event.

```clojure
(defmethod event-handler :select-visible-table [{:keys [fx/context fx/event]}]
  (let [table (fx/sub-ctx context subs-visible-table-to-select)]
    (println ":select-visible-table --> table = " (:table_name table))
    {:context (fx/swap-context context assoc :selected-table table)}));;--- SAVE FULL TABLE DATA ---
```

For the _columns-view_ we need to extract just the table name.

```clojure
(defn columns-view [{:keys [fx/context]}]
  {:fx/type :v-box
   :children [{:fx/type :label
               :text (str "Columns for table: "
                          (:table_name (fx/sub-val context :selected-table)))}]});;--- ONLY TABLENAME ---
```

(Remember: we changed a view procedure so you must execute _(renderer)_ in the REPL to see the changes)

Now we can use the _:selected-item_ property on the _table-view_ to automatically select the table in the table-view based on the value in the state's _:selected-table_

```clojure
(defn tables-view [{:keys [fx/context]}]
  {:fx/type :v-box
   :children [;; ...
              {:fx/type fx.ext.table-view/with-selection-props
               :props {:selection-mode :single
                       :on-selected-item-changed {:event/type :select-table}
                       :selected-item (fx/sub-val context :selected-table)};;--- SHOW SELECTED TABLE ---
               :desc {:fx/type :table-view
                      ;; ...
                      }}]})
```


### 12.5 Improve User Experience {#12-dot-5-improve-user-experience}

There is still an inconvenience.

Suppose we have an empty filter.  And we select the table _t_customer_address_.
![](/ox-hugo/si-step12-50.png)

If we now type _cus_ in the filter text edit, the first visible table _t_customer_ gets selected.

{{< figure src="/ox-hugo/si-step12-51.png" >}}

This is a bit annoying because we previously selected _t_customer_address_. Because that selected table also occurs after filtering I expect that table to remain selected.

All we have to do is change _subs-visible-table-to-select_ to return the current selected table if that is still present in the new filtered list. Only if the current selected table is not in the new filtered list we return the first table from the list.

```clojure
(defn subs-visible-table-to-select
  "Return the table from the visible table list that must be selected"
  [context]
  (let [tables (fx/sub-ctx context subs-filtered-tables)
        first-table (first tables)
        selected-table (fx/sub-val context :selected-table)
        contains-selected-table (some? (some #(= (:table_name selected-table) (:table_name %)) tables)) ]
    (println "subs-visible-table-to-select (contains-selected-table = " contains-selected-table ")")
    (if contains-selected-table
                             selected-table
                             first-table)))
```


## Step 13 - Show the columns {#step-13-show-the-columns}
