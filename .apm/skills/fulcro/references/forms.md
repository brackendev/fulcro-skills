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
| `fs/add-form-config` / `fs/add-form-config*` | Add form metadata to a tree entity (initial state, `merge-component!`) or, with `*`, to a normalized entity inside a mutation |
| `fs/dirty?` | Has any field changed since pristine? |
| `fs/make-validator` | Build a validator from a `(fn [form field] boolean)` checker. The validator returns `:valid`, `:invalid`, or `:unchecked` for a form or a single field |
| `fs/get-spec-validity` | The same three values, using the `clojure.spec` specs registered for the field keys |
| `fs/valid-spec?` / `fs/invalid-spec?` / `fs/checked?` | Boolean shortcuts over spec validity and completion |
| `fs/mark-complete*` | Mark a field as "user has interacted" so validation can run |
| `fs/entity->pristine*` | Snapshot the current state as the new pristine baseline |
| `fs/pristine->entity*` | Revert to the pristine snapshot |

A field reports `:unchecked` until `fs/mark-complete*` marks it, so mark fields complete on blur or before submit. Define one validator per form and drive error rendering from it in the component body:

```clojure
(def profile-validator
  (fs/make-validator
    (fn [form field]
      (case field
        :profile/email (boolean (re-find #"@" (str (get form field))))
        true))))

;; in render
(when (= :invalid (profile-validator props :profile/email))
  (dom/span "Enter a valid email address."))
```

Query `fs/form-config-join` in the form component, or the form state never reaches the props and `fs/dirty?` and the validators have nothing to read.

Fulcro RAD layers `fo/*` (`com.fulcrologic.rad.form-options`) on top of `fs/*` for attribute-driven forms. Its `defsc-form` macro generates the component and manages loading, saving, dirty checking, and validation. A render plugin draws the body when the form has none, and a hand-written body takes over the rendering while RAD keeps managing the data. Outside RAD, build the form body in component render and reach for `fs/*` directly.
