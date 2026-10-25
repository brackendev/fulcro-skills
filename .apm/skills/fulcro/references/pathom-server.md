# Pathom Server

Part of the `fulcro` skill, for Fulcro 3 applications.

## Pathom Server

The current server-side EQL implementation is [Pathom 3](https://pathom3.wsscode.com/). The conventional alias is `pco` for `com.wsscode.pathom3.connect.operation`.

```clojure
(ns app.server.resolvers
  (:require
   [com.wsscode.pathom3.connect.operation :as pco]
   [com.wsscode.pathom3.connect.indexes :as pci]
   [com.wsscode.pathom3.interface.eql :as p.eql]
   [app.server.db :as db]))

(pco/defresolver person-by-id [{:person/keys [id]}]
  {::pco/output [:person/id :person/name :person/email]}
  (db/get-person id))

(pco/defresolver all-people [_]
  {::pco/output [{:all-people [:person/id]}]}
  {:all-people (db/list-people)})

(def env (pci/register [person-by-id all-people]))

(defn process [eql] (p.eql/process env eql))
```

Mutations on the server use `(pco/defmutation ...)`. Symbols must match the client side so EQL flows symmetrically: a client `defmutation` in `app.model.session/login` sends a transaction whose key is the fully-qualified symbol `app.model.session/login`; the server resolves that symbol to a Pathom mutation of the same name.

Pathom 2 (`com.wsscode.pathom.connect`) is still in widespread use. New projects should adopt Pathom 3; existing Pathom 2 codebases should consult the [Pathom 3 migration guide](https://pathom3.wsscode.com/docs/migrate-from-pathom2/) before rewriting incrementally.

Wire the parser into a transit endpoint (`/api`) using Ring middleware. Fulcro's HTTP remote sends a transit-encoded EQL request and expects a transit-encoded result. The exact Ring stack varies; the developer's guide chapter on servers walks through one canonical wiring.
