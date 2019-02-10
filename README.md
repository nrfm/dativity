# dativity

Get 

Dativity is a stateless, data driven process engine library for Clojure and ClojureScript.

It is inspired by the [Artifact centric business process model.](https://en.wikipedia.org/wiki/Artifact-centric_business_process_model#cite_note-VAN2005-6)

## Motivation 

Conventional process engines (like [Activiti](https://www.activiti.org/)) are centered around activities and the sequence in which they should be performed.

The key concept of Dativity is not to say in what sequence actions in a process _should_ be performed. 
But rather what actions _can_ be performed given how business activities depend on collected information. 

For example, you cannot review an insurance claim before the claim has been submitted. However, it can be reviewed precisely after it has been submitted.

Process engines should be concerned with process models,

## 

Dativity models a process into three different entities:
* Actions
* Data
* Roles

The above entites relate in the following ways:
* Data (green) can be _required_ by an action
* An action (purple) _produces_ data
* R role (yellow) _performs_ an action


![](dativity.png)
_a simple credit application process_

In the above example, the action 'create case' produces the data 'case id' and 'customer-id'. When those pieces of information have been added to the case, 'enter-loan-details' can be performed because it only depends on 'case-id' to be present.
## Abilities
Given a process definition and a set of collected data, Dativity can answer questions like:
* What actions can be performed next?
* What actions can be performed by role X (user, system, officer...)
* What actions have been performed?
* Is action X allowed?
* What data is required for action X?

Sometimes a user goes back to a previous step and change data. 
Then all subsequent (in the sense that the changed data is required by other actions) actions need to be invalidated.
Dativity offers an invalidation feature, which rewinds the process to the action that was re-done. Previously entered data is kept, but the depending actions need to be performed again.

## Examples  

The case data is just a map
```clojure
(def case {})
```

Defining an empty case model
```clojure
(def case-model (dativity.define/empty-case-model))
```

Add action, data and role entities to the model
```clojure
(def case-model
  (-> case-model
      ; Actions
      (dativity.define/add-entity-to-model (dativity.define/action :create-case))
      (dativity.define/add-entity-to-model (dativity.define/action :enter-loan-details))
      (dativity.define/add-entity-to-model (dativity.define/action :produce-credit-application-document))
      (dativity.define/add-entity-to-model (dativity.define/action :sign-credit-application-document))
      (dativity.define/add-entity-to-model (dativity.define/action :payout-loan))
      ; Data entities
      (dativity.define/add-entity-to-model (dativity.define/data :case-id))
      (dativity.define/add-entity-to-model (dativity.define/data :customer-id))
      (dativity.define/add-entity-to-model (dativity.define/data :loan-details))
      (dativity.define/add-entity-to-model (dativity.define/data :credit-application-document))
      (dativity.define/add-entity-to-model (dativity.define/data :applicant-signature))
      (dativity.define/add-entity-to-model (dativity.define/data :officer-signature))
      (dativity.define/add-entity-to-model (dativity.define/data :loan-number))
      ; Roles
      (dativity.define/add-entity-to-model (dativity.define/role :applicant))
      (dativity.define/add-entity-to-model (dativity.define/role :system))
      (dativity.define/add-entity-to-model (dativity.define/role :officer))
      ; Production edges
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :create-case :customer-id))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :create-case :case-id))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :enter-loan-details :loan-details))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :produce-credit-application-document :credit-application-document))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :sign-credit-application-document :applicant-signature))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :sign-credit-application-document :officer-signature))
      (dativity.define/add-relationship-to-model (dativity.define/action-produces :payout-loan :loan-number))
      ; Prerequisite edges
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :enter-loan-details :case-id))
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :produce-credit-application-document :loan-details))
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :produce-credit-application-document :customer-id))
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :sign-credit-application-document :credit-application-document))
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :payout-loan :applicant-signature))
      (dativity.define/add-relationship-to-model (dativity.define/action-requires :payout-loan :officer-signature))
      ; Role-action edges
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :applicant :create-case))
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :applicant :enter-loan-details))
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :applicant :sign-credit-application-document))
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :officer :sign-credit-application-document))
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :system :payout-loan))
      (dativity.define/add-relationship-to-model (dativity.define/role-performs :system :produce-credit-application-document))))
```

Generate a picture of the process definition (requires graphviz, clj only).
```clojure
(dativity.visualize/generate-png case-model)
```

What actions are possible?
```clojure
(dativity.core/next-actions case-model case)
=> #{:create-case}
```

What can the roles do?
```clojure
(dativity.core/next-actions case-model case :applicant)
=> #{:create-case}
(dativity.core/next-actions case-model case :officer)
=> #{}
```

What data is produced by ':create-case'?
```clojure
(dativity.core/data-produced-by-action case-model :create-case)
=> #{:customer-id :case-id}
```

Add some data to the case to simulate a few actions
```clojure
(def case
  (-> case
    (dativity.core/add-data :case-id "542967")
    (dativity.core/add-data :customer-id "199209049345")
    (dativity.core/add-data :loan-details {:amount 100000 :purpose "home"})))
```

What actions were performed (simulated) so far?
```clojure
(dativity.core/actions-performed case-definition case)
=> #{:enter-loan-details :create-case}
```

Who can do what?
```clojure
(dativity.core/next-actions case-model case :applicant)
=> #{}
(dativity.core/next-actions case-model case :system)
=> #{:produce-credit-application-document}
(dativity.core/next-actions case-model case :officer)
=> #{}
```
## Usage

To generate graph pictures, install [graphviz](https://graphviz.gitlab.io/download/):

`brew install graphviz`

## Dependencies
The core functionality of Dativity only depends on Clojure.

[Ubergraph](https://github.com/Engelberg/ubergraph) is used as an adapter to graphviz for vizualisation and is only used by a utility namespace.

## License

MIT License

Copyright (c) 2018 Morgan Bentell
