---
title: Skip HHVM Interop Internals
author: Aaron Orenstein
---

## Overview

The first step with interop is getting Skip and HHVM to talk to each other.  Interop between HHVM and Skip is done through the HHVM extensions mechanism.  We register an extension which adds a `Skip::init()` function to Hack.  When Hack calls `Skip::init()` we load or reload the skip shared object (DLL), ask it what functions it provides and register them with the current HHVM request as native functions.

### Skip normally generates executables.  How do we generate this DLL for HHVM to load?

Normally our compiler generates llvm assembly, we compile that with clang and then link that against our runtime and a stub called sk_standalone.cpp which contains some simple initialization code.

This is problematic for HHVM use for several reasons:

1. HHVM owns main
2. We want to be able to reload the skip code and if our runtime is linked against it then we won't be able to share data (such as memoization data).
3. We want our runtime to use the same libraries as HHVM (such as folly).  If we link our runtime against our compiled code then we would either have to discover and export the entire folly ABI from HHVM or have two separate copies of folly loaded in one process.
4. The runtime does a lot of low-level hooks to handle memory and singleton initialization.  Those are really hard to make sure they unload cleanly in a DLL.


To handle these issues we statically link the Skip runtime directly against HHVM.  This way all loaded Skip binaries share a single runtime and we only have to expose a small ABI for Skip binaries to use.  To compute the ABI to expose, we link a small stub DLL and look at the symbols that were exported.  The skip code is then linked against a small stub called sk_hhvm.cpp which provides initialization and routing helper code (instead of linking against sk_standalone.cpp).

The next issue we have is when the runtime wants to call into SKIPC functions.  SKIPC functions are helper functions generated by skip_to_llvm and intended to be called by the Skip runtime (such as functions for creating specialized types of objects like `Array<Int>`) as opposed to Skip code written by the user and exported so it can be called from Hack.

Although you can only load a single Skip DLL for a given request, in sandbox mode you could have several DLLs loaded each being used by different ongoing requests (imagine starting a long-running request, changing some Skip code and running a new request - you'll have the old Skip code loaded in the old request until it finishes and the new Skip code loaded in the new request until it finishes).  The runtime needs to know to which binary to route its calls.

To handle routing calls each SKIPC function has a stub compiled into the extension which looks up the current binary in a thread-local variable and forwards the call through a big interface which calls the underlying SKIPC function.  Both the stubs and the interface are generated during buck builds.

## Skip Runtime API

Functions which are defined in the Skip runtime and expected to be called from the Skip generated code are prefixed with 'SKIP_' or are in the 'skip::' namespace.

SKIPC functions are so named because they're prefixed with 'SKIPC_'.  (Note that for historical reasons there are a number of functions which should be prefixed with 'SKIPC_' but are prefixed with 'SKIP_'.  These are listed in the variable HACK_REMOVE_LIST).

## Calling Hack from Skip

In Skip you can import a Hack function by declaring a prototype and annotating it with @hhvm_import:

```
@hhvm_import
untracked native fun my_php_function(x: Int, y: Int): String;
```

This prototype allows you to call the named Hack function as if it were a Skip function.

### How it works under the covers

Calling a Hack function is relatively easy - The user Skip code calls into the extension and passes it the function name, parameters, and a description of what types those parameters are.  The extension converts those parameter values from Skip representations to HHVM representations and asks HHVM to call the function.  When the Hack function returns we convert the HHVM return value to a Skip value and return it.

## Calling Skip from Hack

You can export a Skip function so it can be called from Hack by annotating it with @hhvm_export:

```
@hhvm_export
fun add_two(a: Int, b: Int): Int {
  a + b
}
```

Calling Skip from Hack is a little harder because we have to generate a stub for each Skip function we want to call from Hack.  So for every @hhvm_export function the Skip compiler generates a wrapper function which expects the HHVM types, converts those types to Skip, calls the original Skip function, converts the return value back to HHVM and returns it.

During compilation skip_to_llvm collects a list of all the @hhvm_export functions and signatures and the extension fetches that data when the Skip DLL is loaded.  This allows the extension to tell HHVM what native functions are available.

## Portable Types

This document refers to “portable types” as the types which can be passed between Skip and HHVM (whether by copying or proxying).  The following table enumerates these portable types.

|Skip Type	|HHVM Type	|Mode	|Notes	|
|---	|---	|---	|---	|
|Bool	|bool	|copy	|	|
|Int	|int	|copy	|	|
|Float	|float	|copy	|	|
|String	|string	|copy	|	|
|@hhvm_import("T")	|T	|proxy	|	|
|@hhvm_shape("T")	|shape	|proxy	|	|
|@hhvm_copy("T")	|T	|copy	|	|
|@hhvm_shape_copy("T")	|shape	|copy	|	|
|Vector.HH_varray2<Tv>	|varray	|proxy	|Tv must be a portable type.	|
|Map.HH_darray2<Tk, Tv>	|darray	|proxy	|Tk must be Int, String or a subtype of HH.Arraykey.  Tv must be a portable type.	|
|Set.HH_keyset2<Tk>	|keyset	|proxy	|Tk must be Int, String or a subtype of HH.Arraykey.	|
|HH.Mixed	|mixed	|copy	|Excludes HH.Object and HH.Resource.	|
|Array<Tv>	|varray	|copy	|Tv must be a portable type.	|
|Vector<Tv>	|varray	|copy	|Tv must be a portable type.	|
|Map<Tk, Tv>	|darray	|copy	|Tk must be Int, String or a subtype of HH.Arraykey.  Tv must be a portable type.	|
|UnorderedMap<Tk, Tv>	|darray	|copy	|Tk must be Int, String or a subtype of HH.Arraykey.  Tv must be a portable type.	|
|Set<Tk>	|keyset	|copy	|Tk must be Int, String or a subtype of HH.Arraykey.	|
|UnorderedSet<Tk>	|keyset	|copy	|Tk must be Int, String or a subtype of HH.Arraykey.	|
|Tuple<T0...>	|array	|copy	|The types of each tuple element must be portable types.	|
|?T	|?T	|copy	|T must be a portable type.	|
|??T	|?T	|copy	|Special case for optional fields in shapes - see below	|

An example of a non-portable type would be a normal Skip class without any @hhvm annotation.

### ??T - Optional fields in shapes

In Hack a shape can be defined to have an optional field:

```
type Point = shape('x' => int, 'y' => int, ?'z' => ?int);
```

In this case the Skip class could be defined as either:

```
@hhvm_shape_copy
class Point {
  x: Int,
  y: Int,
  z: ?Int
}
```

or:

```
@hhvm_shape_copy
class Point {
  x: Int,
  y: Int,
  z: ??Int
}
```

Values are copied as follows:

|Hack Value	|Skip Value for ?T	|Skip Value for ??T	|
|---	|---	|---	|
|undefined	|None()	|None()	|
|null	|None()	|Some(None())	|
|value	|Some(value)	|Some(Some(value))	|

## Type Table

When the Skip DLL is loaded the extension discovers the set of HHVM classes and the fields and types expected on each by calling the SKIPC function SKIPC_hhvmTypeTable().  The format for this table is defined as the class TypeTable in github:src/runtime/prelude/native/Svmi.sk.

During discovery the extension iterates through all the classes and ensures that their definition in Skip matches the definitions from Hack.  It also iterates through all the fields and caches the HHVM slot offset for each field.

## Talking About Objects

Objects are converted from HHVM to Skip or from Skip to HHVM in a few places. When Skip code calls a @hhvm_import function the parameters need to be converted from Skip to HHVM and the return value needs to be converted from HHVM to Skip.  When Hack code calls a Skip @hhvm_export function the parameters need to be converted from HHVM to Skip and the return value needs to be converted from Skip to HHVM.  Reading fields from an @hhvm_import object needs to convert the result from HHVM to Skip and writing fields to an @hhvm_import object needs to convert the value from Skip to HHVM.

### Copying OBJECTS

To mark classes as copying you annotate them with either @hhvm_copy or @hhvm_shape_copy.

Copying objects from HHVM to Skip is done via a "gather" operation.  When skip_to_llvm realizes it needs to copy an object from HHVM to Skip it generates code to call SKIP_HhvmInterop_Gather_gatherCollect(), passing it the object and type table classId it wants to collect.  The extension will recursively visit each field in the class as specified in the type table, check that it's the correct type and then serialize it to a buffer in the form that Skip expects. Finally it returns a pointer to a filled buffer and a cleanup handle.  When the code is finished building the Skip objects it calls SKIP_HhvmInterop_Gather_gatherCleanup() to free the buffer.

Copying objects from Skip to HHVM is done via a “push” operation.  When skip_to_llvm realizes it needs to copy an object from Skip to HHVM it generates code to call SKIP_HhvmInterop_*DataType*Cons_create() (where *DataType* is either “Object” or “Shape”) telling it the type table classId it wants to construct.  The generated code then calls SKIP_HhvmInterop_*DataType*Cons_setField*Type*() once for each field (where *Type* is the field type such as 'Int' or 'Mixed'). Finally the generated code calls SKIP_HhvmInterop_*DataType*Cons_finish() to note that construction is done.

### Proxying OBJECTS

When proxying objects skip_to_llvm replaces the Skip fields with a single “proxyPointer” field.

Field gets are turned into calls to HhvmInterop.propertyGetHelper() which in turn is lowered into calls to SKIP_HHVM_*DataType*_getField_*Type*() (where *DataType* is either “Object” or “Shape” and *Type* represents a specialized type like 'Int' or a generic type like 'Mixed').

Field sets are turned into calls to HhvmInterop.propertySetHelper() which in turn is lowered into calls to SKIP_HHVM_*DataType*_setField_*Type*().

## FAQ

**Q. How does the extension know what DLL to load?**

A. The path to the shared object is given to HHVM as a config variable called `Skip.Binary`.

**Q. How does the extension know when it's safe to unload a DLL?**

A. It uses the HHVM treadmill to defer unloading until after the last HHVM request which could still be using the DLL has completed.

**Q. What if I want to have my exported or imported function named differently in Hack and Skip?**

A. Both @hhvm_import and @hhvm_export annotations accept a parameter which is the Hack name to use.  For example

```
@hhvm_import(“my_hack_function”)
native fun my_skip_function(x: Int): Int;
```

will call the Hack function my_hack_function when a Skip program calls my_skip_function().

**Q. How does Hack/Skip interop handle copy-on-write HHVM arrays?**

A. It doesn't.  For now, HHVM arrays are read-only.

**Q: Does the gather operation really serialize everything?  For example - do string contents get serialized into the temporary buffer?**

A: It stores all the copied values with some special cases.  So for primitive types (Int, Float, Bool) it stores the value.  For Strings it copies the string into Skip's memory and format and stores the pointer to the string (technically the StringRep).  For proxied objects it stores the HhvmHandle pointer.  For copied objects it recursively stores all the fields.  The details of the format can be found at the top of src/native/hhvm/gather.sk.
