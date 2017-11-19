# Contents
  - [Motivation](#motivation)
  - [What they look like](#looks)
  - [Other kinds of mixins](#other-mixins)
  - [Protocols](#protocols)
  - [Syntax and semantics](#syntax-and-semantics)

<a name="motivation"></a>
# Definition mixins

Based on the [class fields proposal](https://tc39.github.io/proposal-class-fields/).

Mixins are an essential OO feature. They allow code reuse in defining concepts in an OO style. There have been noticeable attempts to implement mixins in the JS community, but all had important downsides. See the section about [other kinds of mixins](#other-mixins) for more details. Beside fixing the issues with other kinds of mixins, definition mixins also allow sharing private state between the mixin context and the mixin functions. Since private state is an essential encapsulation tool in OOP, being able to use it with mixins is important. Since private state shouldn't be accessible for everybody, the private state needs to be explicitly introduced in trustworthy mixin providers from the context where the mixins are mixed in. Definition mixins can be used in both class contexts and function contexts.

The reason the semantics described in this proposal are difficult to implement in a library is that the mix object that is passed as the second argument to the mixin function providers needs to be managed. Having this in the language allows managing that object automatically based on the [[ScriptOrModule]] slot, that library code doesn't have access to.

<a name="looks"></a>
# What they look like

```js
function mixinFunc1(obj, mix, privateState) {
  return (v) => obj.y + mix.mixinFunc2(3, privateState.x);
}

function mixinFunc2(obj) {
  return (a, b) => a + b + obj.z;
}

function mixinFunc3() {
  return () => { console.log('mixin func 3'); };
}

// mixin context 1
class Context1 {
  #privateState = {x: 100};
  y = 3;
  z = 1000;

  mixin this:
    mixinFunc1,
    mixinFunc2:
      this.#privateState;

  mixin on this: mixinFunc3;

  method() {
    /* ... */
    this.#mixinFunc1(35);
    this.mixinFunc3();
  }
}

// mixin context 2
function Context2() {
  const privateState = {x: 100};
  const obj = {
    y: 3,
    z: 1000
  };

  // mixinFunc1 and mixinFunc2 become constant bindings inside Context2
  mixin obj:
    mixinFunc1,
    mixinFunc2:
      privateState;

  mixin on obj: mixinFunc3 as method2;

  obj.method = () => {
    /* ... */
    mixinFunc1(35);
    obj.method2();
  };

  return obj;
}
```

<a name="other-mixins"></a>
# Other kinds of mixins

The common ways mixins have been implemented so far were:
  - naive approach - Object.assign, Reactjs mixins, Typescript mixins
  - based on prototypal inheritance
  - [functional mixins](https://medium.com/javascript-scene/functional-mixins-composing-software-ffb66d5e731c)

The main downsides of these approaches are:
  - Not being able to easily tell what your class/object can do by simply looking at its definition context (and having to jump to different files)
  - Not being able to easily tell where your class'/object's capabilities come from by simply looking at its definition context
  - Avoiding property collisions by making the last mixin object always win (which is not the best solution)
  - The gorilla-banana problem. You automatically get all the methods in the mixin objects even though you might not need all of them. This is a violation of the least knowledge principle.
  - The inability of sharing private state between the mixin context and the mixin functions.

The first 4 issues are solved in a composition based mixin mechanism by explicitly picking every mixin function.

<a name="protocols"></a>
# Protocols

Currently there is a [protocols proposal](https://github.com/michaelficarra/proposal-first-class-protocols). Protocols, in their current form, can act as naive mixins, which means they have the issues that the other kinds of mixins have. While protocols usually use symbols and property collisions can be avoided this way, they also allow using strings as property keys. If protocols are used as mixins it's very likely that string keys will be used because they're easier to work with.

As I understand it, the main purpose of protocols is implementing contracts. But I think it's important to be clear about whether the protocols mechanism is meant to be used as a general purpose mixin mechanism and not only for contracts. If it's a general mixin mechanism, it has all these disadvantages in comparison to definition mixins. If it's mainly for contracts and it also uses a mixin mechanism in order to provide default implementations, then 1) we should have a separate more capable and well designed mixin mechanism that can be used without protocols and 2) it would make more sense for the protocols to use the same mixin mechanism, instead of using a different mixin mechanism.

I also worry that if there is no separate mixin mechanism the protocols will be abused as naive mixins.

Definition mixins could be used together with protocols. The protocols could then be used to impose a contract, while the mixin mechanism can be used to pick the default implementations from the protocols.

```js
protocol P {
  a() { /* ... */ }
}

class C implements P {
  mixin on this: P.a;
}
```

This way protocols could also use private state.

<a name="syntax-and-semantics"></a>
# Syntax and semantics

There are two versions of the syntax and semantics (mostly the evaluation section). The first one
is simpler and implementable starting from the current specs. The second one is based on
indirect bindings in a function context, which are currently not used anywhere in the language, so it
might not be pragmatic. The idea behind this version is that the context object and the private state that is shared with the mixin functions can be created after evaluating the mixins declarations. The second version only contains the differences from the first version.

Version 1
---

Syntax
---

```
Declaration:
    HoistableDeclaration
    ClassDeclaration
    LexicalDeclaration
    MixinDeclaration

ClassElement:
    MethodDefinition
    static MethodDefinition
    FieldDefinition
    static FieldDefinition
    MixinDeclaration
    ;

MixinDeclaration:
    MixinPrefix MixinContext : MixinSourceList ;
    MixinPrefix MixinContext : MixinSourceList : MixinBindingList ;

MixinPrefix:
    mixin
    mixin on

MixinContext:
    LeftHandSideExpression

MixinSourceList:
    MixinSource
    MixinSourceList , MixinSource

MixinSource:
    MixinSourceExpression
    MixinSourceExpression as IdentifierName

MixinSourceExpression:
    MemberExpression . IdentifierName
    IdentifierName

MixinBindingList:
    MixinBinding
    MixinBindingList , MixinBinding

MixinBinding:
    MemberExpression . PrivateName
    IdentifierName
```

---

Early Errors
---

```
Mixin declarations must be allowed only at the top level of a FunctionDeclaration,
FunctionExpression or class constructor or as a ClassElement.
```

---

Evaluation
---

```
Let newTarget be GetNewTarget().
If newTarget is undefined, throw a ReferenceError exception.

Let mixinContextReference be the result of evaluating MixinContext.
Let mixinContextObject be GetValue(mixinContextReference).
If Type(mixinContextObject) is not Object throw a TypeError exception.

Let mixinFunctionProviders be a new List.
For each MixinSource in MixinSourceList
    Let ref be the result of evaluating the MixinSourceExpression of MixinSource.
    Assert: Type(ref) is Reference.
    
    Let mixinFunctionProvider be GetValue(ref).
    If IsCallable(mixinFunctionProvider) is false, throw a TypeError exception.

    If mixinFunctionProvider doesn't have a [[ScriptOrModule]] slot or if its value is null
        Throw a TypeError exception.
        NOTE:
            We could allow the host environment to generate a value for non-Ecmascript
            functions, but let's throw. Unfortunately this also disallows proxy functions
            and bound functions to be mixin function providers since they don't have
            a [[ScriptOrModule]] slot.

    Add mixinFunctionProvider to mixinFunctionProviders.

Let mixinBindings be ArgumentListEvaluation of MixinBindingList if it was provided
or an empty list otherwise.

If the MixinDeclaration is a ClassElement
    Let isClassElement be true.
    Let env be the running execution context's PrivateNameEnvironment.
Else
    Let isClassElement be false.
    Let env be the running execution context's LexicalEnvironment.
Let envRec be env's EnvironmentRecord.

For each MixinSource in MixinSourceList
    Let mixinFunctionProvider be the item in mixinFunctionProviders on the same position as
    MixinSource is in MixinSourceList.
    NOTE: We avoid calling GetValue multiple times.
    
    Let mixinSourceMappingKey be mixinFunctionProvider.[[ScriptOrModule]].
  
    If the MixinSource has an IdentifierName, let resultedBindingIdentifierName be the StringValue
    of IdentifierName.
    Else let resultedBindingIdentifierName be null.

    Let mixinSourceExpresssion be the MixinSourceExpression of MixinSource.
    Let mixinFuncKey be the StringValue of mixinSourceExpresssion's IdentifierName.
    If resultedBindingIdentifierName is null, let resultedBindingIdentifierName be mixinFuncKey.

    Let mixinMappingObject be envRec.ResolveMixinMapping(mixinSourceMappingKey).
    If HasProperty(mixinMappingObject.target, mixinFuncKey) throw a ReferenceError exception.

    Let args be << mixinContextObject, mixinMappingObject.proxy >>.
    Add the items in mixinBindings to the args list.

    Let mixinFunction be Call(mixinFunctionProvider, undefined, args).
    If IsCallable(mixinFunction) is false, throw a TypeError exception.

    mixinMappingObject.target.[[Set]](mixinFuncKey, mixinFunction, mixinMappingObject.target).
    If the MixinPrefix of MixinContext contains 'on'
        CreateDataPropertyOrThrow(mixinContextObject, resultedBindingIdentifierName, mixinFunction).      
    Else
        NOTE:
            This is probably wrong, in the sense that probably the binding should be
            created at a different point in time, for instance during the function declaration
            instantiation, and later it should be initialized.
        If isClassElement
            Let resultedBindingIdentifierName be '#' + resultedBindingIdentifierName.    
            envRec.createImmutableBinding(resultedBindingIdentifierName, true).
            Let field be NewPrivateName(resultedBindingIdentifierName).
            Perform envRec.InitializeBinding(resultedBindingIdentifierName, field).
            PrivateFieldSet(field, ResolveThisBinding(), mixinFunction).
        Else
            envRec.createImmutableBinding(resultedBindingIdentifierName, true).
            Perform envRec.InitializeBinding(resultedBindingIdentifierName, mixinFunction).
```

---

Declarative Environment Records
---
ResolveMixinMapping(mixinSourceMappingKey)
---

```
If environmentRecord has a mapping object for mixinSourceMappingKey return it.
Else
    Let mixinMappingObject be NewMixinMappingObject.
    Record the mixinMappingObject as a mapping object of environementRecord for mixinSourceMappingKey.
    return mixinMappingObject.
```

---

Mixin Mapping Objects
---

```
A mixin mapping object is an ordinary object with a target and a proxy property.
The proxy will be a Proxy whose target is the target object.
An implementation can use a different mechanism with the same effect.
```

---

NewMixinMappingObject
---

```
Let mixinMappingObject be ObjectCreate(null).
Let target be ObjectCreate(null).
Let proxy be new Proxy(target, {
    set() { return false; },
    deleteProperty() { return false; }
    preventExtensions() { return false; }
    defineProperty() { return false; }
    setPrototypeOf() { return false; }
}).
mixinMappingObject.[[Set]]('target', target, mixinMappingObject).
mixinMappingObject.[[Set]]('proxy', proxy, mixinMappingObject).
return mixinMappingObject.
```

---

Version 2
---

Syntax
---

```
MixinContext:
    MemberExpression . IdentifierName
    MemberExpression . PrivateName
    IdentifierName
```

---

Evaluation
---

```
Let newTarget be GetNewTarget().
If newTarget is undefined, throw a ReferenceError exception.

Let mixinContextReference be the result of evaluating MixinContext.
Assert: Type(mixinContextReference) is Reference.

If the MixinPrefix of MixinContext contains 'on'
    Let mixinContextObject be GetValue(mixinContextReference).
    If Type(mixinContextObject) is not Object throw a TypeError exception.
Else let mixinContextObject be null. 

Let mixinSourceReferences be a new List.
For each MixinSource in MixinSourceList
    Let mixinSourceReference be the result of evaluating the MixinSourceExpression of MixinSource.
    Add mixinSourceReference to the mixinSourceReferences list.

Let mixinBindings be the result of evaluating MixinBindingList if it was provided
or an empty list otherwise.

If the MixinDeclaration is a ClassElement
    Let isClassElement be true.
    Let env be the running execution context's PrivateNameEnvironment.
Else
    Let isClassElement be false.
    Let env be the running execution context's LexicalEnvironment.
Let envRec be env's EnvironmentRecord.

For each MixinSource in MixinSourceList
    Let mixinSourceReference be the item in the mixinSourceReferences list on the same position
    as MixinSource is in MixinSourceList.
    Assert: Type(mixinSourceReference) is Reference.
    
    Let mixinFunctionProvider be GetValue(mixinSourceReference).
    If IsCallable(mixinFunctionProvider) is false, throw a TypeError exception.
    
    If mixinFunctionProvider has a [[ScriptOrModule]] slot and it's not null
        Let mixinSourceMappingKey be mixinFunctionProvider.[[ScriptOrModule]].
    Else
        Throw a TypeError exception.
  
    If the MixinSource has an IdentifierName, let resultedBindingIdentifierName be the StringValue
    of IdentifierName.
    Else let resultedBindingIdentifierName be null.

    Let mixinSourceExpresssion be the MixinSourceExpression of MixinSource.
    Let mixinFuncKey be the StringValue of mixinSourceExpresssion's IdentifierName.
    If resultedBindingIdentifierName is null, let resultedBindingIdentifierName be mixinFuncKey.

    Let mixinMappingObject be envRec.ResolveMixinMapping(mixinSourceMappingKey).
    If HasProperty(mixinMappingObject.target, mixinFuncKey) throw a ReferenceError exception.
    Let mixinMappingObjectReference be a value of type Reference whose base value component
    is mixinMappingObject, whose referenced name component is 'proxy', whose strict reference
    flag is true, and whose private reference component is true.

    Let args list be a new List.
    Add mixinContextReference to the args list.
    Add mixinMappingObjectReference to the args list.
    Add the items in mixinBindings to the args list.

    Let mixinFunction be the result of calling mixinFunctionProvider with undefined
    as the thisArgument and the args list as the argumentsList in a way that makes the arguments
    in argumentsList indirect bindings in the mixinFunctionProvider function call.
    NOTE:
        This means they are read only inside mixinFunctionProvider. I think that in order
        to obtain this the current spec would require significant changes. If this is not possible,
        then at least we have version 1.

    If IsCallable(mixinFunction) is false, throw a TypeError exception.

    mixinMappingObject.target.[[Set]](mixinFuncKey, mixinFunction, mixinMappingObject.target).
    If mixinContextObject is not null
        CreateDataPropertyOrThrow(mixinContextObject, resultedBindingIdentifierName, mixinFunction).      
    Else        
        NOTE:
            This is probably wrong, in the sense that probably the binding should be
            created at a different point in time, for instance during the function declaration
            instantiation, and later it should be initialized.
        If isClassElement
            Let resultedBindingIdentifierName be '#' + resultedBindingIdentifierName.    
            envRec.createImmutableBinding(resultedBindingIdentifierName, true).
            Let field be NewPrivateName(resultedBindingIdentifierName).
            Perform envRec.InitializeBinding(resultedBindingIdentifierName, field).
            PrivateFieldSet(field, ResolveThisBinding(), mixinFunction).
        Else
            envRec.createImmutableBinding(resultedBindingIdentifierName, true).
            Perform envRec.InitializeBinding(resultedBindingIdentifierName, mixinFunction).
```
