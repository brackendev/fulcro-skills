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

Navigate with `(dr/change-route! this ["person" id])`. Routes can defer rendering (`dr/route-deferred`) until a load completes, then call `dr/target-ready` to signal completion.

For HTML5 history integration, wire the [`kibu/pushy`](https://github.com/kibu-australia/pushy) library to bridge browser navigation events to `dr/change-route!`. The pattern is documented in the developer's guide.
