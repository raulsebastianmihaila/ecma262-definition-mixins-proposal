# Definition mixins

Based on the [class fields proposal](https://tc39.github.io/proposal-class-fields/).

There are two versions of the syntax and semantics (mostly the evaluation section). The first one
is simpler and implementable starting from the current specs. The second one is based on
indirect bindings in a function context, which are currently not used anywhere in the language, so it
might not be pragmatic. The second version only contains the differences from the first version.

# Version 1

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
            PrivateFieldSet(field, this, mixinFunction).
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

# Version 2

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
            PrivateFieldSet(field, this, mixinFunction).
        Else
            envRec.createImmutableBinding(resultedBindingIdentifierName, true).
            Perform envRec.InitializeBinding(resultedBindingIdentifierName, mixinFunction).
```
