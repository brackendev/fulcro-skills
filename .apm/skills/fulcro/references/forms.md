# Forms

Part of the `fulcro` skill, for Fulcro 3 applications.

## Forms

`com.fulcrologic.fulcro.algorithms.form-state` (alias `fs`).

```clojure
(ns app.ui.profile
  (:require
   [com.fulcrologic.fulcro.components :as comp :refer [defsc]]
   [com.fulcrologic.fulcro.algorithms.form-state :as fs]
   [com.fulcrologic.fulcro.mutations :as m :refer [defmutation]]))

(defsc ProfileForm [this {:profile/keys [name email] :as props}]
  {:query         [:profile/id :profile/name :profile/email fs/form-config-join]
   :ident         :profile/id
   :form-fields   #{:profile/name :profile/email}
   :initial-state (fn [params] (fs/add-form-config ProfileForm params))})

(defmutation save-profile [_]
  (action [{:keys [state ref]}]
    (swap! state fs/entity->pristine* ref))
  (remote [_] true))
```

Key helpers:

| Helper | Purpose |
|--------|---------|
| `fs/add-form-config` / `fs/add-form-config*` | Add form metadata to an entity |
| `fs/dirty?` | Has any field changed since pristine? |
| `fs/validity` | `:valid`, `:invalid`, or `:unchecked` for a form or field |
| `fs/mark-complete*` | Mark a field as "user has interacted" so validation can run |
| `fs/entity->pristine*` | Snapshot the current state as the new pristine baseline |
| `fs/pristine->entity*` | Revert to the pristine snapshot |

Validation runs through a predicate per field (`(defmethod fs/form-field-valid? :profile/email ...)`), with custom error rendering driven by `(fs/validity profile :profile/email)` in the component body.

Fulcro RAD layers `fo/*` (`com.fulcrologic.rad.form-options`) on top of `fs/*` for attribute-driven form generation. When working in a RAD project, prefer RAD's form rendering; outside RAD, build the form body in component render and reach for `fs/*` directly.
