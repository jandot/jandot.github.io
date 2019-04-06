---
layout: post
title:  "Clojurescript, re-frame and modular data visualisation"
date:   2019-04-06
categories: clojure datavis
---
One of the issues we still have is how to combine our custom data visualisations in multi-view dashboards with brushing/linking. We'll have a programmer work on this for the next two years, but as clojure is a bit of a hobby I'd like to try that out as well. It appears that clojurescript [re-frame](https://github.com/Day8/re-frame) fits the bill.

Get set up:
```bash
lein new re-frame my-project
cd my-project
lein figwheel dev
```

`my-project/resources/public/index.html` is the landing page for the application. It contains a `div` with id `app` that will hold the actual html and svg that we'll create in `views.cljs`.

`index.html`
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset='utf-8'>


  </head>
  <body>
    <div id="app"></div>
    <script src="js/compiled/app.js"></script>
    <script>my_re_frame.core.init();</script>
  </body>
</html>
```

The `my-project/src/cljs/my_project` directory hosts the most important files.

* `core.cljs` defines which div in `index.html` is used to mount the application.
* `db.cljs` holds the application state.
* `events.cljs`: no idea yet what this does...
* `subs.cljs` contains subscriptions to different parts of the application state
* `views.cljs` holds the actual application code

I still need to read the '6-domino cascade' part at the re-frame website, so it's not all clear yet. Will update here where I get there...

`core.cljs`: didn't touch this
```clojure
(ns my-re-frame.core
  (:require
   [reagent.core :as reagent]
   [re-frame.core :as re-frame]
   [my-re-frame.events :as events]
   [my-re-frame.views :as views]
   [my-re-frame.config :as config]
   ))

(defn dev-setup []
  (when config/debug?
    (enable-console-print!)
    (println "dev mode")))

(defn mount-root []
  (re-frame/clear-subscription-cache!)
  (reagent/render [views/main-panel]
                  (.getElementById js/document "app")))

(defn ^:export init []
  (re-frame/dispatch-sync [::events/initialize-db])
  (dev-setup)
  (mount-root))
```

`db.cljs` holds the app state
```clojure
(ns my-re-frame.db)

(def default-db
  {:name "re-frame"
   :data [1 2 3 5 8 13 21 34 55 89]})
```

`events.cljs`
```clojure
(ns my-re-frame.events
  (:require
   [re-frame.core :as re-frame]
   [my-re-frame.db :as db]
   ))

(re-frame/reg-event-db
 ::initialize-db
 (fn [_ _]
   db/default-db))
```

`subs.cljs`
```clojure
(ns my-re-frame.subs
  (:require
   [re-frame.core :as re-frame]))

(re-frame/reg-sub
 ::name
 (fn [db]
   (:name db)))

(re-frame/reg-sub
  ::data
  (fn [db]
    (:data db)))
```

`views.cljs` is where I define the main panel, as well as the functions to draw circle and rectangle. These latter two should later be moved to separate files so as to make this truly modular.
Note the `:on-mouse-over` and `:on-click` (and keep a look at your browser console); we'll probably use that later for brushing/linking.
```clojure
(ns my-re-frame.views
  (:require
   [re-frame.core :as re-frame]
   [my-re-frame.subs :as subs]))

(defn rect
  [pos]
  [:rect {
    :style {
      :fill "rgba(0,0,128,0.3)"
      :stroke-width 0}
    :key (str "r" pos)
    :x (+ 50 pos)
    :y (+ 50 pos)
    :width 10
    :height 10
    :on-mouse-over #(println (str "rect" pos))
  }])

(defn circle
  [pos]
  [:ellipse {
    :style {
      :fill "rgba(128,0,0,0.3)"
      :stroke-width 0}
    :key (str "c" pos)
    :cx (+ (+ pos 50) 50)
    :cy (+ pos 50)
    :rx 10
    :ry 10
    :on-mouse-over #(println (str "circle" pos))
  }])

(defn main-panel []
  (let [name (re-frame/subscribe [::subs/name])
        data (re-frame/subscribe [::subs/data])]
    [:div
      [:h1 "Hello from " @name]
      [:p "Before"]
      [:br]
      [:svg {:x 0 :y 0 :width 400 :height 200 :on-click #(println "Clicked")}
        (map rect @data)
        (map circle @data)
      ]
      [:br]
      [:p "After"]

    ]))
```

This is ridiculously simple. Here's the result:<br/>
<img src="{{site.base}}/assets/re-frame_screenshot.png" style="width: 250px;"/>

