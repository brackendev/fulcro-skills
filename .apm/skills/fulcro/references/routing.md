# Routing

Part of the `fulcro` skill, for Fulcro 3 applications.

## Routing

Dynamic routing from `com.fulcrologic.fulcro.routing.dynamic-routing` (alias `dr`).

```clojure
(ns app.ui.root
  (:require
    [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
    [com.fulcrologic.fulcro.routing.dynamic-routing :as dr :refer [defrouter]]
    [app.ui.home :refer [Home]]
    [app.ui.people :refer [PeoplePage]]
    [app.ui.person :refer [PersonDetail]]))

(defrouter RootRouter [this {:keys [current-state]}]
  {:router-targets [Home PeoplePage PersonDetail]})

(defsc Root [this {:keys [router]}]
  {:query         [{:router (comp/get-query RootRouter)}]
   :initial-state {:router {}}}
  (ui-root-router router))
```

Each router target component declares a `:route-segment`:

```clojure
(defsc PersonDetail [this {:person/keys [id name]}]
  {:query         [:person/id :person/name]
   :ident         :person/id
   :route-segment ["person" :person-id]
   :will-enter    (fn [app {:keys [person-id]}]
                    (dr/route-deferred
                      [:person/id (uuid person-id)]
                      #(df/load! app [:person/id (uuid person-id)] PersonDetail
                         {:post-mutation        `dr/target-ready
                          :post-mutation-params {:target [:person/id (uuid person-id)]}})))}
  (dom/div name))
```

Navigate with `(dr/change-route! this ["person" id])`. Keyword elements of a `:route-segment` receive the path element as a string, which is why `:will-enter` above calls `uuid` on it. Routes can defer rendering (`dr/route-deferred`) until a load completes, then call `dr/target-ready` to signal completion.

Route lifecycle rules from the `dynamic-routing` docstrings and the Developer's Guide:

- **Route at startup.** The app stays unrouted until something calls `dr/change-route!`. The first router target only supplies initial state. A typical client `init` calls `(app/set-root! SPA Root {:initialize-state? true})`, `(dr/initialize! SPA)`, and `(dr/change-route! SPA ["home"])` before `(app/mount! SPA Root "app" {:initialize-state? false})`. Route to a leaf target the first time a nested router is shown.
- **Keep `:will-enter` free of side effects.** The router can call it several times while it resolves one route. Put loads and other effects in the function passed to `dr/route-deferred`, as above, and return `dr/route-immediate` or `dr/route-deferred` with the target's ident.
- **Guard leaving a route with `:allow-route-change?`.** It receives the component instance and returns `false` to refuse the change. It can also be called more than once, so it must not have side effects either. `:route-denied` on the same target runs when it refused, and can ask the user and then call `dr/retry-route!` with the arguments it received.

The dynamic router does not read or write the browser URL. To sync with HTML5 history, split the URL path on `/`, pass the resulting vector to `dr/change-route!`, and update the URL after routing with the History API or a library such as [`kibu/pushy`](https://github.com/kibu-australia/pushy).
