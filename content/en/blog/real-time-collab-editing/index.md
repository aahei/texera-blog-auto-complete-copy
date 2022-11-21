---
title: "How We Built a Real-time Collaborative Workflow Editor in Texera"
description: ""
lead: "In this blog, we share how we built a real-time collaborative workflow editor."
date: 2022-11-17T14:27:42+01:00
lastmod: 2022-11-17T14:27:42+01:00
draft: false
weight: 50
# images: ["demo_video.jpg"]
contributors: ["Xiaozhen Liu", "Zuozhi Wang"]
aliases:
  - /blog/how-we-built-a-real-time-collabortive-workflow-editor-in-texera/
---

Real-time collaborative editing has been so popularized in recent years to the point that it has become almost a hidden standard for major document editing providers. [Google Docs Editors](https://support.google.com/docs/?hl=en#topic=1382883), which includes Google Docs, Google Slides, etc., was a pioneer in real-time collaborative editing. Microsoft Office did not have real-time co-authoring capability until [2013](https://), 7 years after the initial release of Google Docs Editors. Then came Overleaf, which has become the go-to place for co-editing LaTeX documents. In WWDC this year, collaboration has also been a focus for Apple, with the introduction of FreeForm, a tool that allows users to collaborate on a single board with drawings, text, videos, etc. This is to add to Apple's trend of making most of its tools real-time collaborative, including iWorks, Apple Notes, and even Safari tabs.

In 2022, instead of just a fancy add-on,  real-time collaboration (RTC) has become a must for most document editing apps. However, this capability is not limited to document editing, where the collaboration happens on static, no-executable contents, and primarily texts. For data science tools, RTC has also gradually become a new norm. JupyterLab, the native web-based IDE for Jupyter Notebooks, introduced RTC in v3.1 (See *Figure 1*). This addition allows a user to share a notebook to others to collaboratively edit a notebook, but the notebook environment is still hosted locally. Google Colab and DeepNote made Jupyter Notebook available as cloud services, while DeepNote stood out recently for its better RTC, environment management, and version control.

<figure>
<a href="https://jupyterlab.readthedocs.io/en/stable/user/rtc.html">
<img src="jupyter.png" alt="Fig 1" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 1: Collaborative Editing in Jupyter Lab</i></figcaption>
</figure>

But Data Science tools are not just about Notebooks, and collaboration more often than not involves knowledge from multiple domains, in which case not everyone knows Python or Jupyter Notebook. For this kind of collaboration, workflow-based tools with which one can simply use GUI to formulate a [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (DAG) of operators become the natural platforms.

Texera is such a cloud service for GUI-and-workflow-based collaborative data analytics at scale where users with different levels of technical expertise can work together at the same time. Before, the collaboration in Texera only allowed sharing a workflow, but two users could not co-edit the workflow in real time. With the introduction of RTC, using Texera to collaborate on Data Science tasks becomes even more natural, seamless, and powerful. More excitingly, the collaboration happens both at the workflow construction and execution stages, as demonstrated in the following video (See *Figure 2*):

<figure>
<a href="https://www.youtube.com/watch?v=2gfPUZNsoBs">
<img src="demo_video.jpg" alt="Fig 2" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 2: Texera's Demo Video for RTC and Collaborative Data Analytics</i></figcaption>
</figure>


In the following sections, we are going to unveil how we made real-time collaborative editing for a GUI-based workflow possible in Texera.

## Conflict Resolution and CRDT

In our [initial attempt](https://github.com/Texera/texera/pull/1392) at RTC for workflow editor, we did not use any existing shared editing library but implemented a version based solely on WebSocket. A demo can be seen in *Figure 3*. The core implementation logic is to simply propagate editing actions from one client to all other connected clients for a workflow and replay the actions on these clients. This was not actually shared editing because we enforced an editing lock so that only one user can edit at a time to prevent any possible conflicts. This implementation was simple for our architecture and served a limited set of scenarios where only letting a user follow another user's editing session was enough.

<figure>
<a href="https://github.com/Texera/texera/pull/1460">
<img src="initial_imp.gif" alt="Fig 3" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 3: Texera's Initial Attempt at RTC with Lock</i></figcaption>
</figure>


However, we soon realized this lock-based implementation was not enough. To make our workflow shared editing truly achieve the goal of enabling collaborative data analytics, we had to get rid of the lock.

From day one we knew that conflict resolution is the key to real-time shared editing. But to do it ourselves would be not only complicated but also unnecessary, because the current technology is ripe enough that there are plenty of existing libraries. The algorithms used by most libraries are either [CRDT](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (Conflict-Free Replicated Data Type) or [OT](https://en.wikipedia.org/wiki/Operational_transformation) (Operational Transformation), two major approaches to distributed data consistency and real-time conflict resolution.

There is a plethora of resources introducing and [comparing](https://news.ycombinator.com/item?id=18191867) [CRDT](https://crdt.tech) and [OT](https://www3.ntu.edu.sg/scse/staff/czsun/projects/otfaq/) which obviates the need for detailed dicsussions here, but to put it simply, OT relies on a centralized server and handles conflict resolution based on the transforming editing operations. We do not choose OT mainly because there are fewer libraries available (even though it has a few decades more history than CRDT), and all the existing libraries that do use OT are either [deprecated](https://support.google.com/answer/1083134?hl=en) or [no longer under maintenance](https://github.com/jsfiddle/togetherjs).

On the contrary, most shared-editing libraries today are CRDT-based, with Yjs and Automerge being two popular open-source libraries. A very intuitive introduction to what a CRDT looks like can be found [here](https://ably.com/blog/crdts-distributed-data-consistency-challenges), and [another very well-written blog series](https://lars.hupel.info/topics/crdt/01-intro/) gives a more detailed explanation of CRDT. To put it simply, A CRDT is a well-defined data structure with a strict total order. As a result, it can reach consistency regardless of the order of updates, as long as all the updates are eventually received by each client. So the conflict resolution with CRDT really depends on the data itself, and no transformation of operations is needed. That is probably a reason why CRDT is very popular for these ready-to-use libraries, since the core of the library would only need to be the structure.

*Table 1* presents a list of RTC libraries that we have compiled during our search.

<figcaption align = "center"><i>Table 1: RTC Libraries (by Nov. 2022)</i></figcaption>

| Name                                                                            | License | Algorithm | Age (yrs) | Status  | Repo                                                                                                                                                                                                     |
| ------------------------------------------------------------------------------- | ------- | --------- | --------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [](https://fluidframework.com/)[FluidFramework](https://fluidframework.com/)    | MIT     | CRDT-like | 5         | Active  | [](https://github.com/microsoft/FluidFramework)[https://github.com/microsoft/FluidFramework](https://github.com/microsoft/FluidFramework)                                                                |
| [](https://docs.yjs.dev/)[Yjs](https://docs.yjs.dev/)                           | MIT     | CRDT      | 7         | Active  | [](https://github.com/yjs/yjs)[https://github.com/yjs/yjs](https://github.com/yjs/yjs)                                                                                                                   |
| [](https://github.com/share/sharedb)[sharedb](https://github.com/share/sharedb) | MIT     | OT        | 9         | Active  | [](https://github.com/share/sharedb)[https://github.com/share/sharedb](https://github.com/share/sharedb)                                                                                                 |
| [](https://automerge.org/)[automerge](https://automerge.org/)                   | MIT     | CRDT      | 6         | Active  | [](https://github.com/automerge/automerge)[https://github.com/automerge/automerge](https://github.com/automerge/automerge)                                                                               |
| [](https://replicache.dev/)[Replicache](https://replicache.dev/)                | BSL     | None      | 2         | Active  | [](https://github.com/rocicorp/replicache)[https://github.com/rocicorp/replicache](https://github.com/rocicorp/replicache)                                                                               |
| [SyncedStore](http://syncedstore.org/) (Yjs-based)                              | MIT     | CRDT      | 1         | Active  | [](https://github.com/YousefED/SyncedStore)[https://github.com/YousefED/SyncedStore](https://github.com/YousefED/SyncedStore)                                                                            |
| [](https://convergence.io/)[Convergence](https://convergence.io/)               | LGPLv3  | Unknown   | 7         | Active  | [](https://github.com/convergencelabs/convergence-client-javascript)[https://github.com/convergencelabs/convergence-client-javascript](https://github.com/convergencelabs/convergence-client-javascript) |
| [](https://webstrates.net/)[Webstrates](https://webstrates.net/)                | Apache  | OT        | 9         | Active  | [](https://github.com/Webstrates/Webstrates)[https://github.com/Webstrates/Webstrates](https://github.com/Webstrates/Webstrates)                                                                         |
| [](https://togetherjs.com/)[TogetherJS](https://togetherjs.com/)                | MPL     | OT        | 10        | Stopped | [](https://github.com/jsfiddle/togetherjs)[https://github.com/jsfiddle/togetherjs](https://github.com/jsfiddle/togetherjs)                                                                               |
| [](https://swellrt.org/)[swellrt](https://swellrt.org/)                         | Apache  | OT        | 12        | Stopped | [](https://github.com/SwellRT/swellrt)[https://github.com/SwellRT/swellrt](https://github.com/SwellRT/swellrt)                                                                                           |

We eventually chose Yjs because it supports various data types as CRDTs, including text, array, map, xml, etc., and the ability to build custom data types based on them was critical. Most importantly, Yjs does not require us to rewrite our backend server and is compatible with our frontend framework, Angular. Another appealing aspect of Yjs is its ready-to-use bindings to several editors, e.g., Quill Editor for rich text and Monaco Editor for code, which is a big convenience for us since we do not need to build collaborative text editors ourselves. The theory behind Yjs, [YATA](https://www.researchgate.net/publication/310212186_Near_Real-Time_Peer-to-Peer_Shared_Editing_on_Extensible_Data_Types), is also an interesting CRDT.

## Challenges

Yjs is powerful as a CRDT library, but for Texera's frontend, it cannot be used as-is. Even though Yjs encapsulates several commonly used data structures as CRDTs, in particular, Texts, Maps and Arrays,  Texera's core workflow data is not that simple. There exist three challenges:

1. We are not building a collaborative editor from scratch. Instead, we are migrating from an existing architecture that was built using plain TypeScript objects and has a well-defined architecture.
2. The old implementation involved quite a lot of APIs that relied on the assumption of previous data structures for workflows.
3. Most collaborative editors have one or only a few core data structures that need to be shared, e.g., for text editors, there needs to be only one single shared data structure, which is the text. This is not the case for Texera's Workflow Editor because it involves a more complicated data model.

## Change of Architecture

To make it easier to understand, *Figure 4* gives an overview of the elements that support Texera's Workflow Editor before the introduction of RTC. Let us leave things like User Systems and all that stuff about management of different workflows out for now, and focus on the case of only one workflow.

<figure>
<a href="https://github.com/Texera/texera/pull/1674">
<img src="before.png" alt="Fig 4" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 4: Texera's Previous Architecture for Workflow Editor</i></figcaption>
</figure>


Essentially, the data model for a workflow editor in Texera stores a DAG, which can be broken down into operators and links. We represent them separately in two `Map` structures, with their respective IDs being map keys. This can be seen in the left box for ***Texera Graph*** in the diagram. There are two more maps in *Texera Graph*, which are for other features. In general, the maps in *Texera Graph* are all that we need to send to Texera's backend engine (Amber) to execute, so *Texera Graph* is like the backbone of the frontend.

But to show the DAG as nice UIs and to let a user create and edit the DAG requires not only the *Data Model*. We rely on a diagramming library to bridge the *Data Model* and user interface/interaction, i.e., the *View Model*. In Texera's case, we use the open-source library [*JointJS*](https://www.jointjs.com). Our view model is another graph called ***Joint Graph***, which also includes Operators, links, etc. The difference in what our data model stores and what JointGraph includes means they are not two-way bound, hence the need to sync the two models.

In the old architecture, we used the *[Command Pattern](https://en.wikipedia.org/wiki/Command_pattern)* to handle this synchronization. Every available action done by the user to modify a workflow is encapsulated inside an action in `WorkflowActionService`,  and this service is the gateway to both the data model and view model, meaning in each action both *Texera Graph* and *Joint Graph* will be updated at the same time.

For example, when the user drags and drops an operator from the operator library (a component that is not part of *Joint Graph*), the component calls `addOperator` in  `WorkflowActionService` , which creates a visual operator component and adds into *Joint Graph*,  and also adds the corresponding `OperatorPredicate` object with its necessary information like operator type and properties into `Texera Graph`.

Another complexity is that instead of directly executing the action, we wrap the action as a *Command*, which is basically a callback function that is not executed immediately at the start of  `addOperator` , but passed until the end along with its corresponding "reverse-action", so that when the command is actually executed, we make sure to also store its reverse action in `UndoRedoService`, in this case removing the operator from both models.

This old architecture solved three things at a time:
1. Synchronization of both models,
2. Enabling Undos/Redos when editing the workflow,
3. Realization of our first attempt at collaborative workflow editing, which basically was using WebSocket to broadcast these commands to other users so that the actions can be replayed. But this approach had a fundamental problem of not being able to do any conflict resolution, so a "lock" had to be enforced.

However, when we try to include synchronization with other clients into the scope, we see an apparent conflict in design patterns. Since it would be very complicated to try to alter the *Joint Graph* model into CRDTs, only *Texera Graph*  can be made shared-editable. The migration of *Texera Graph* is another challenge which we will show later, but a more pressing issue is the separation of data model and view model.

For example, for adding an operator, previously all we had to do was add the new `OperatorPredicate` into `OperatorIDMap` and draw a new Operator in *Joint Graph*. In a shared-editable *Texera Graph*,  `OperatorIDMap` is a CRDT which can be automatically synced across collaborators]. After adding a new `OperatorPredicate` into `OperatorIDMap`, if we also add the operator into *Joint Graph* in local frontend, since the view is not automatically synced, other collaborators will not be able to see it on their frontends.

Solving this would require our architecture to be *[Observer](https://en.wikipedia.org/wiki/Observer_pattern)*-based as opposed to *Command*-based. The good thing about using Yjs is that it has convenient observer APIs for its data structures: the `.observe` and `.observeDeep` methods on any [`Y.Map`](https://docs.yjs.dev/api/shared-types/y.map), [`Y.Array`](https://docs.yjs.dev/api/shared-types/y.array) and [`Y.Text`](https://docs.yjs.dev/api/shared-types/y.text), etc. Using these observers, we can listen to any changes on the data model, regardless of whether these changes come from "my frontend" or other people's actions.

A very commonly used observer API in our codebase is `Y.Map.observe`. We can use `.observe` to listen to the addition, deletion and modification of items in the map. Given the changes in the observers for the data model, we can reflect them in the view model, which also serves the purpose of data-view synchronization.

Based on this change in the synchronization logic, we came up with our new architecture for the workflow editor, as shown in *Figure 5*.

<figure>
<a href="https://github.com/Texera/texera/pull/1674">
<img src="after.png" alt="Fig 5" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 5: Texera's New Architecture for Workflow Editor</i></figcaption>
</figure>


The new architecture is what we call "*data-observer*" based, where the "single source of truth" is *Texera Graph*, in particular the CRDT-based `SharedModel`, and the view model, *Joint Graph*, secondarily depends on the data model, *Texera Graph*.

We also unify the event handling logic for local changes and remote changes, i.e., regardless of whether an action is done by the local user or a collaborator, we always update `SharedModel` first, and then in the corresponding observer(s), we update the view model.

For example, when adding an operator, `WorkflowActionService` would only call `add()` on `OperatorIDMap` and not do anything about *Joint Graph* at all. Yjs, along with *y-websocket*, the lightweight network provider for Yjs, would automatically sync the changes on `OperatorIDMap` across all the clients, and notify the observers associated with `OperatorIDMap` that a new item has been added to this map. On all clients, we have the same event handler in `SharedModelChangeHandler` class to add this new operator in the *Joint Graph* of each respective frontends.

There are views other than *Joint Graph*, such as the navigation bar and property editor, but their handling logic is similar since two-way binding is not possible for them. For some components, however, there are two-way bindings available, either natively or from existing shared editors, which saves us a lot of trouble.

## Construction of Shared Data Model

We have seen the problem of data-view synchronization can be solved by our change design patterns. Let us look at another complexity when implementing RTC for our workflow editor based on Yjs. Up till now we have treated our data model, *Texera Graph* and its underlying maps as a black box. But to make collaborative editing seamless we need more than just being able to sync the addition and deletion of operators. To sync the details of an operator, like changes in its name, its properties, its disabled/enabled status, etc, we need to be able to control the shared object in a finer-grained manner. For `OperatorIDMap`, its values are of `OperatorPredicate` type:

```
export interface OperatorPredicate
  extends Readonly<{
  operatorID: string;
    operatorType: string;
    operatorVersion: string;
    operatorProperties: Readonly<{ [key: string]: any }>;
    inputPorts: { portID: string; displayName?: string }[];
    outputPorts: { portID: string; displayName?: string }[];
    showAdvanced: boolean;
    isDisabled?: boolean;
    isCached?: boolean;
    customDisplayName?: string;
  }> {}
```

We see on this interface that there are richer contents in an `OperatorPredicate` than just the type and name. For the `OperatorProperties` it is even more complicated because itself is a map which can have any type of contents, but that complexity will not be covered here yet. To be able to know which specific change on a particular `OperatorPredicate` is, the `OperatorPredicate` itself needs to be a `Y.Map` because the parent `Y.Map` , `OperatorIDMap`, can only detect if its map values have changed, but cannot detect the content of the change unless the change is also on a Yjs structure (`Y.Map`, `Y.Array`, `Y.Text`, etc) via the `ObserveDeep` method. That is why we need to make `OperatorPredicate` type a `Y.Map`.

Generally, any JavaScript object that we want to have finer-grained control of shared editing can and needs to be made a `Y.Map`. If we only need it as a whole, it can be left as a plain JavaScript object. For example, for `OperatorLinkMap`, its original type is as follows:

```
export interface OperatorLink
  extends Readonly<{
  linkID: string;
    source: OperatorPort;
    target: OperatorPort;
  }> {}
```
Since we never need to modify an existing link's source or target, instead we always create a new link or delete an old link, we consider a link itself to be an atomic unit, and do NOT need a link to be made into a `Y.Map`.

Another problem is with engineering efforts, in particular API changes. Outside the scope of *Texera Graph*, all other existing code use the plain JavaScript object interface for `OperatorPredicate`, etc., so if we were to directly get rid of the old interface it would involve huge refactoring efforts. That is why we decided not to change the interfaces outside *Texera Graph*, and preserve the existing APIs for the actions in `WorkflowActionService` and *Texera Graph*. Take our running example, adding an operator, as an instance. Previously the corresponding method in *Texera Graph* had the following signature:

```
public addOperator(operator: OperatorPredicate): void
```

We do not change the API. Instead, we create a new Yjs object based on `OperatorPredicate`. For finer-grained actions like changing the enabled/disabled status of an operator, the API is like the following:

```
public enableOperator(operatorID: string): void
```

We also do not change such APIs. We modify the specific sub-attribute on the existing Yjs object.

In general, almost none of the APIs need to be changed. Instead, we change the internal implementations from modifying the plain JavaScript object to modifying Yjs-based objects.

For methods that expect to *read* plain JavaScript objects, it is even easier since all Yjs-based objects have a method `.ToJSON()` which returns itself as plain JavaScript objects.

In the end, the part of our workflow editor that is for shared editing is contained within `SharedModel` class, which is transparent to outside the scope of *Texera Graph*. To uniformly represent this shared-editable black box based on existing interfaces without any manual efforts, and to preserve the type-strictness of TypeScript, we came up with something like the reverse-action of `.toJSON()` and its expected type, which we call `createYTypeFromObject<T extends object>(obj: T): YType<T>`. Let's look at the expected return type of this method:

```
export type YTextify<T> = T extends string ? Y.Text : T;
export type YArrayify<T> = T extends Array<any> ? Y.Array<any> : T;
export type YType<T> = Omit<Y.AbstractType<any>, "get" | "set" | "has" | "toJSON"> & {
  get<TKey extends keyof T>(key: TKey): YArrayify<YTextify<T[TKey]>>;
  set<TKey extends keyof T>(key: TKey, value: YArrayify<YTextify<T[TKey]>>): void;
  has<TKey extends keyof T>(key: TKey): boolean;
  toJSON(): T;
};
```

What `YType<T>` does as a TypeScript type is that it converts any JavaScript object type `T`, including `Array`, `String` and custom types like `OperatorPredicate`, into a corresponding `Y.AbstractType`. It also replaces the signatures of `get`, `set`, `has` and `toJSON` methods from `Y.AbstractType` so that these methods have strong type-checking before being compiled into JavaScript codes. The resulting type is like a hybrid of the original type and Yjs's type.

Now that we know what the type is for, `createTypeFromObject()` is a template function that converts any JavaScript object into a Yjs object. We use the following conversion rule in *Table 2*:

<figcaption align = "center"><i>Table 2: Conversion Rule for <code>createYTYpeFromObject</code></i></figcaption>

| Original type   | Yjs Type      |
| --------------- | ------------- |
| `Object`          | `Y.Map`         |
| `Array`           | `Y.Array`       |
| `String`/`string`   | `Y.Text`        |
| Other primitive types | No conversion |
| Others          | Error         |

And the conversion is recursive, meaning all properties of a given `Object` or `Array` or will also be converted according to this rule, until a "leaf" is reached, which will either be a string or other primitive types that cannot be converted. A `string` (primitive type) or `String` (inherits `Object`) will be converted to `Y.Text` so that it can be directly connected to existing shared editor libraries for Yjs. Other primitive types do not need conversion since a `number` or `boolean` is an atomic type which can only be replaced, unlike `String` ,`Arrat` or `Object` which can be nested and the modification can happen on its children.

This suits Texera's custom types/interfaces well since most of our original data models are plain objects with only properties and no methods. Based on this, we were able to preserve most of the previous APIs. For this method:
```
public addOperator(operator: OperatorPredicate): void
```
There is no need to change the input type. We just need to create the shared object inside the method and add to the shared model:

```
const newOp = createYTypeFromObject(operator);
this.sharedModel.operatorIDMap.set(operator.operatorID, newOp);
```

And all of its sub-objects will also be Yjs-based, so that fine-grained control is available.

That concludes how we solved the 2nd and 3rd problems.

## Other features

It would be less interesting to enumerate all the details of this work, but some other features are worthy enough to be mentioned here.

### User Presence and "Shadowing" Mode

User presence in RTC editors typically means the capability of being able to see other co-editors' current editing states in real-time. In a text editor, it is typically the current text cursor of other users. In a drawing editor (like Diagrams.net), it is typically the mouse cursor or the currently selected object.

All of these have been implemented in Texera, using Yjs's [`awareness`](https://docs.yjs.dev/getting-started/adding-awareness) API, which saves each currently connected user's states. `Y.awareness` is also a shared map, with the key set being each connected user's clientIDs. A caveat of this API is that its values have to be a plain JavaScript map meant to keep each client's state information (a state dictionary), with the key being the name of the state and value being any type. This map is NOT a Y.Map so it cannot use any of its APIs, which means the state dictionary has no fine-grained updates. Because of this limitation quite a lot of engineering efforts are made to implement our custom User Presence features.

We also implemented the "Shadowing" Mode feature, which lets a user's frontend mirror whatever is going on with another currently editing user's frontend. In our implementation, this feature is part of the user presence features and shares the same logic.

### Shared Property Editor

The Property Editor lets a user edit the properties of an operator, which was of `Readonly<{ [key: string]: any }>` (map) type in our original API. Previously, when updating an operator's property, the API looks like this:

```
setOperatorProperty(operatorID: string, newProperty: object): void
```
This is because we hooked the (read-only) property object to *formly*, which only gives us the whole object every time an update in any field happens. Internally in this method, we just replace the whole property object. This worked fine in previous settings, and is mostly fine under the new architecture by using `createYTypeFromObject`. However, this does not allow fine-grained RTC control over the operator properties, especially if we want the textual fields to be shared-editable without collaborators replacing each other's texts.

*Figure 6* shows our Shared Property Editor with fine-grained control of textual fields.

<figure>
<a href="https://github.com/Texera/texera/pull/1712">
<img src="property_editor.gif" alt="Fig 6" style="width:100%">
</a>
<figcaption align = "center"><i>Figure 6: Texera's Collaborative Operator Property Editor</i></figcaption>
</figure>

To allow fine-grained control, one thing needs to be made sure: no existing shared object can be replaced with a new shared object, e.g., when a `Y.Text` has been created and added to the map, any modification on this `Y.Text` needs to be done in-place by modifying its children instead of creating a new `Y.Text` .

But since we do not know the specific changes on the property object, how would we be able to even locate which attribute is to be modified? For that, we wrote a `updateYTypeFromObject()` method:

```
export function updateYTypeFromObject<T extends object>(oldYObj: YType<T>, newObj: T)
```

Internally in this method, we compare the new object with the old shared object and try to do in-place updates as many as possible. A few assumptions are made based on our use cases, so this is not a true general-purpose function that can be used for any update. But at least it fits our purpose, and does not require API-changes.

On the more engineering-heavy side, since we use the *formly* library to generate the UI for the property object (given the schema), to make all textual fields shared-editable, we create a custom wrapper for all `string` fields, which locates the specific `Y.Text` and connects to a *Quill* shared text editor. The part about locating the shared text given only a field and its parent fields is also interesting, but for brevity let us not delve into such details.

## Summary

In this blog, we shared how we managed to implement real-time collaborative editing for Texera's Workflow Editor. Specifically, we introduced the conflict-resolution algorithm we chose and how we solved a few engineering challenges along the way. We hope the insights of this article can empower more open-source systems to include RTC more easily. In the future we might add more interesting details related to this work.

## Acknowledgement

We want to thank Prof. Chen Li for helping with this blog.
