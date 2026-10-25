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

## Securing the API (Iron Law FUL-5)

The `/api` endpoint accepts whatever EQL a client sends, so a resolver or mutation runs for any request that names it, whether or not the UI offers that action. The Fulcro Developer's Guide puts it directly in "Full-Stack Error Handling": "the server has to be able to validate a request for security reasons. There is no getting around this. You cannot trust a client." Its "Security" chapter calls the network API "the most critical part of your application to secure." The Pathom 3 documentation recommends setting the current user in the environment at the boundary because "we don't want to allow users to override it (for security reasons)."

- Authenticate the request in Ring middleware, and put the session user into the Pathom environment.
- Check authorization in each resolver and mutation that reads or changes protected data, using the user from the environment, not an id from the query parameters alone.
- Treat a failed check as an attack, as the guide suggests: throw on the server rather than returning a friendly error for the UI to handle.
- The guide's chapter also covers HTTPS, secure cookies, CSRF tokens, and origin checks. It notes that the chapter has not been updated for Fulcro 3, so check its middleware examples against current libraries.

