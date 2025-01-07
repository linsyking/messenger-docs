---
sidebar_position: 3
---

# Messenger Model

The concept of the Messenger model is summarized in the following diagram:

![](/img/intro1.jpg)

Messenger provides two parts for users: the template _user code_ and the _core library code_. Users write code based on the template and may use any functions in the core library. In user code, users need to design the _logic_ of scenes, layers, and components. _Logic_ includes the data structure it uses, the `update` function that updates the data when events occur, and the `view` function that renders the object. Messenger core first transforms world events into user events, then sends those events to the scene with the current `globalData`. `globalData` is the data structure Messenger keeps between scenes, and users can read and write it. The user code updates its own data and generates some `SceneOutputMessage`. Messenger core ("core" for short) will handle all these messages and update `globalData`.

Messenger manages a game through three levels of objects (users can create more levels if they want), listed from parent to child:

1. **Scenes**: The core maintains *one scene* at a time. Example: a single level of the game
2. **Layers**: One scene may have multiple layers. Example: the map of the game and the character layer
3. **Components**: One layer may have multiple components. The small yellow circles in layers in the diagram are _components_. Example: bullets a character shoots

Parent levels can hold child levels, while child levels can send messages to parent levels. Messages can also be sent within a level between different objects.

## General Model

There are many similarities among scenes, layers, and components. Messenger abstracts those objects into one concept called the _general model_. Scenes are not general models as they are special, but they are similar to general models.

One of the most important features of Messenger is its _customizability_, i.e., users can define their own datatype for their general models. However, if the layers or components in one scene are not of the same type, how can the core update them?

The key is that although the data of general models differs, the _interface_ or actions on those objects are the same. For example, the `update` function is the same for all layers in one scene. As an analogy, an abstract class in OOP may define many _virtual functions_, and derived classes implement those functions with their own data structure and implementation details. It's possible to convert a derived class instance to a base class instance and only use the base class interface. The type of the base class is the same, while the derived classes may have different types.

In Messenger, the "derived class" is a "concrete object" that users implement, and they can use whatever datatype they want. The "base class" is an "abstract object" that "upper-casts" the concrete object. Therefore, users can store different types of objects together in a list by casting them to their abstract form.

However, unlike in OOP, it is impossible to downcast an abstract object to a concrete object. Therefore, users should only upper-cast at the last moment.

Layers and components are defined as an alias of `AbstractGeneralModel`. It is generated by a `ConcreteGeneralModel`, where users implement their logic. Its definition is:

```elm
type alias ConcreteGeneralModel data env event tar msg ren bdata sommsg =
    { init : env -> msg -> ( data, bdata )
    , update : env -> event -> data -> bdata -> ( ( data, bdata ), List (Msg tar msg sommsg), ( env, Bool ) )
    , updaterec : env -> msg -> data -> bdata -> ( ( data, bdata ), List (Msg tar msg sommsg), env )
    , view : env -> data -> bdata -> ren
    , matcher : data -> bdata -> tar -> Bool
    }
```

`init` is the function to initialize the object.
`update` is the function to update the object when an event occur. `updaterec` is the function to update when other object send you a message. `view` is the function to generate `Renderable`. `matcher` is the function to identifies itself.

Type `env` is the _environment type_. It contains global data and common data, if any. `event` is the event type, `data` is the user defined datatype. `bdata` is the base data used in components (see @component). `ren` is the rendering type.

Messenger CLI will use templates to help you create scenes, layer and components.

## Msg Model

The `Msg` type of Messenger is defined as below:

```elm
type Msg othertar msg sommsg
    = Parent (MsgBase msg sommsg)
    | Other (othertar, msg)
```
where `MsgBase` is defined as
```elm
type MsgBase othermsg sommsg
    = SOMMsg sommsg
    | OtherMsg othermsg
```

`SOMMsg`, or _Scene Output Message_, is a message that can directly interact with the core. For example, to play an audio, users can emit a `SOMPlayAudio` message, and the core will handle it.


![](/img/intro2.jpg)



`SOMMsg` is passed to the core from Component $\rightarrow$ Layer $\rightarrow$ Scene. It's possible to block `SOMMsg` from a higher level. See @sommsg to learn more about `SOMMsg`s.

Users may need to handle `Parent` messages from components in a layer. Messenger provides a handy function `handleComponentMsgs` which is defined in `Messenger.Layer.Layer`, to help users handle those messages. Users need to provide a `MsgBase` handler, for example:

```elm
handleComponentMsg : Handler Data SceneCommonData UserData Target LayerMsg SceneMsg ComponentMsg
handleComponentMsg env compmsg data =
    case compmsg of
        SOMMsg som ->
            ( data, [ Parent <| SOMMsg som ], env )

        OtherMsg _ ->
            ( data, [], env )

        _ ->
            ( data, [], env )
```

Then users can combine it with `updateComponents` to define the `update` function in layers (provided in the Messenger template):

```elm
update : LayerUpdate SceneCommonData UserData LayerTarget LayerMsg SceneMsg Data
update env evt data =
    let
        ( comps1, msgs1, ( env1, block1 ) ) =
            updateComponents env evt data.components

        ( data1, msgs2, env2 ) =
            handleComponentMsgs env1 msgs1 { data | components = comps1 } [] handleComponentMsg
    in
    ( data1, msgs2, ( env2, block1 ) )
```

## Example

A, B are two layers (or components) which are in the same scene (or layer) C. The logic of these objects are as follows:

- If A receives an integer $0 \leq x \leq 10$, then A will send $3x$ and $10-3x$ to B, and send $x$ to C.
- If B receives an integer $x$, then B will send $x-1$ to A.

Now at some time B sends $2$ to A, what will C finally receive?

Answer: 2, 5, 3, 8, 0, 9.