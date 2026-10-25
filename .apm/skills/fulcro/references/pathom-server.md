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

Mutations on the server use `(pco/defmutation ...)`. Symbols must match the client side so EQL flows symmetrically: a client `defmutation` in `app.model.session/login` sends a transaction whose key is the fully-qualified symbol `app.model.session/login`; the server resolves that symbol to a Pathom mutation of the same name. `pco/defmutation` names the mutation after the namespace it is defined in, so a server mutation in `app.server.resolvers.session` sets the client symbol explicitly:

```clojure
(pco/defmutation login [env {:keys [email password]}]
  {::pco/op-name 'app.model.session/login}
  {:user (authenticate env email password)})
```

The Pathom 3 documentation covers `::pco/op-name` under "Changing mutation symbol."

Connect the parser to a transit endpoint (`/api`) with Ring middleware. Fulcro's HTTP remote sends a transit-encoded EQL request and expects a transit-encoded result. `references/project-workflows.md` shows one working stack, and `com.fulcrologic.fulcro.server.api-middleware` provides `wrap-api`, `wrap-transit-params`, and `wrap-transit-response` for the same job.

### Tempids and mutation joins

A Pathom 3 mutation join returns only the attributes that the join query asks for. When a client mutation uses `m/returning`, the server mutation's `:tempids` map is missing from the response, and the client keeps the tempid in its database. A plain mutation without a join returns the whole result, `:tempids` included. Add an optional `:tempids` to every mutation join with the app's `:global-eql-transform`, the same idea as Fulcro RAD's `global-eql-transform`:

```clojure
(ns app.application
  (:require
    [com.fulcrologic.fulcro.application :as app]
    [com.fulcrologic.fulcro.networking.http-remote :as http]
    [edn-query-language.core :as eql]))

(def optional-tempids
  ;; The form that (pco/? :tempids) produces, written without requiring Pathom on the client.
  (eql/expr->ast '(:tempids {:com.wsscode.pathom3.connect.operation/optional? true})))

(defn global-eql-transform [ast]
  (let [ast (app/default-global-eql-transform ast)]
    (cond-> ast
      (and (symbol? (:dispatch-key ast)) (seq (:children ast)))
      (update :children conj optional-tempids))))

(defonce SPA
  (app/fulcro-app {:global-eql-transform global-eql-transform
                   :remotes              {:remote (http/fulcro-http-remote {:url "/api"})}}))
```

Keep `:tempids` optional. In Pathom 3's default strict mode, a required `:tempids` in the join fails every mutation whose result has no `:tempids` with "Attribute :tempids is unknown". The server mutation returns `{:tempids {tempid real-id} ...}` as usual.

## Securing the API (Iron Law FUL-5)

The `/api` endpoint accepts whatever EQL a client sends, so a resolver or mutation runs for any request that names it, whether or not the UI offers that action. The Fulcro Developer's Guide puts it directly in "Full-Stack Error Handling": "the server has to be able to validate a request for security reasons. There is no getting around this. You cannot trust a client." Its "Security" chapter calls the network API "the most critical part of your application to secure." The Pathom 3 documentation recommends setting the current user in the environment at the boundary because "we don't want to allow users to override it (for security reasons)."

- Authenticate the request in Ring middleware, and put the session user into the Pathom environment.
- Check authorization in each resolver and mutation that reads or changes protected data, using the user from the environment, not an id from the query parameters alone.
- Treat a failed check as an attack, as the guide suggests: throw on the server rather than returning a friendly error for the UI to handle.
- The guide's chapter also covers HTTPS, secure cookies, CSRF tokens, and origin checks. It notes that the chapter has not been updated for Fulcro 3, so check its middleware examples against current libraries.

