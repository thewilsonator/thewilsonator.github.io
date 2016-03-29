---
layout: post
title:  "Generating better Vulkan bindings"
date:   2016-03-28 19:46:13 +0800
categories: update
---

An adventure in introspective and meta-programming in D
=======================================================

Intro
-----

With the release of the new graphics API Vulkan from Khronos I thought it would good to try and wrap my head around it. Unfortunately Vulkan is a C API, and is therefore not type safe (although waaaay better than OpenGL) and rather clunky to use. However as Vulkan is a C API, it is easily accessible in D as they share a common memory model, so we can make a better API that forwards with no overhead to the C API in a type safe way that is much cleaner and nicer to use way.

I originally intended to generate a D API straight from the spec, but I quickly came to the conclusion that 
that was not a good use of my time for several reasons:

1. The spec is a massively convoluted and annoyingly inconsistent xml document (tags in weird places ect.).
2. It is very C oriented ( full of preprocessor directives, typedefs, C-style array declarations, ...) and ,
3. I would have to make reference to the free functions anyway...

So I decided to start with a D translation of the C API taking care of all the C-isms. I used 
https://github.com/Rikarin/VulkanizeD/blob/master/Vulkan.d  @ 272a8e1 as a starting point and used the power that D offers in introspective and meta-programming to turn the C API into an idiomatic D API.

TL;DR turn 

```C++
struct VkInstanceCreateInfo ici;
ici.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
ici.pNext = null;
ici.flags = 0;
ici.pApplicationInfo = null;
ici.enabledLayerCount = 0;
ici.ppEnabledLayerNames = null;
ici.enabledExtensionsCount = 0;
ici.ppEnabledExtensionNames = null;
VkInstance instance;
VkResult err = VkCreateInstance(&ici,null,&instance);
if(err < 0) bail_out();
unsigned len;
vkEnumeratePhysicalDevices(instance,&len,null);
VkPhysicalDevice* pdevs = new VkPhysicalDevice(len);
vkEnumeratePhysicalDevices(instance,len,pdevs);
```

into

```D
auto ici = Instance.CreateInfo(null,[],[]);
auto instance = Instance(ici,null);
auto pdevs = instance.physicalDevices();
```

with the power that D offers.

Goals
-----

While automatically generating 100% of it is nice in theory, in order to minimise the number of special cases (and there are a lot) we will be doing some manual adjustments at the end as well as spitting the generated code through [dfmt](https://github.com/Hackerpilot/dfmt) to make the code readable. In my code I included some rudimentary indentation code for the ease of debugging which I will leave out for here.

But first what do I mean by an idiomatic D API?

- functions

	- replacing len/ptr pairs with slices and len*/ptr pairs to return arrays
	- wrapping call that could fail (i.e. return a VkResult) with something that throws but only if you try to use it in an invalid state
	- const (char)* to string and const (char*)* to string[]
	- those that return a struct through a pointer in their last arg should return that normally
	- basically the user shouldn't have to deal with (non-handle) pointers EVER.

- enums

	- remove redundant prefix and members
	- conform to D naming conventions

- Handles

	- almost all (vkEnumerateInstance{Extension,Layer}Properties are the exceptions and they should, but they don't for whatever reason) functions take one of these as their first parameter so have them as methods

	- Createinfo

		- nice to use constructors
		- nest these inside their Handle

	- __ctor/__dtor => vk*{Create,Destroy}Foo
		
There is a problem with extensions methods: namely that we have to load them into a function pointer. This wouldn't be a problem except that in vulkan you can have multiple devices, unlike gl where you can have a bunch of function pointers and populate them at load time. I suppose that you could all possible function pointers in the handle wrapper but for the sake of this article we will leave them out.

The preamble code
--------------

The basic method is to import the module containing the C API and iterate through each symbol and emitting it 
as appropriate.

But first the imports:

```D
import std.stdio;	// for File & friends.
import std.algorithm;	// map findSplit & friends startsWith & friends.
import std.traits;	// for inspecting the signatures of function and layout of structs
import std.string;	// dealing with string
import std.array;
import std.conv;	// converting between enum value and name
import std.uni;		// isUpper / toLower & friends 
import std.meta;	// NoDuplicates
import std.exception;	// assumeUnique

import vulkan;	//the C Vulkan API module 
```

In D enums should be in `camelCase` so a few functions for converting between that and `UPPERCASE_SEPARATED_WITH_UNDERSCORES`

```D
string camelToUpper_(string s)
{
    string ret;
    bool first = true;
    foreach(c; s)
    {
        if(isUpper(c) && first)
        {
            ret ~= c;
            first = false;
        }
        else if(isUpper(c))
        {
            ret ~= "_" ~ c;
        }
        else
            ret ~= toUpper(c);
    }
    return ret;
}
string upper_ToCamel(string s)
{
    char[] ret;
    bool saw_;
    foreach(c; s[VKL ..$])
    {
        if(saw_)
        {
            saw_ = false;
            ret ~= c;
        }
        else if (c == '_')
        {
            saw_ = true;
        }
        else
            ret ~= toLower(c);
    }
    if (s.endsWith("KHR"))
    {
        ret[$-3] = 'K';
        ret[$-2] = 'H';
        ret[$-1] = 'R';
    }  
    else if (s.endsWith("EXT"))
    {
        ret[$-3] = 'E';
        ret[$-2] = 'X';
        ret[$-1] = 'T';
    }
    return assumeUnique(ret);
}
```

Yes this is very inefficient but we cannot reserve capacity at compile time, slow is better than broken.

A few globals 

```D
bool[string] emittedSymbols; //keep track of how many symbols we have not yet covered
enum VKL = 2; 	// length of the string "Vk" to differentiate between other meanings of two
File f;		// the file we will write to
bool _debug;	// runtime arg
```
and a debugging utility function

```D
void fwrite(int line = __LINE__, Args... )(Args args)
{
    if (_debug) f.write("/*",line,"*/ "); // saves us from having to recompile
    f.write(args);
}
```

line is evaluated in the context of the caller NOT the callee (like a C macro). This allows us to keep track of which line 
generated what. This is a HUGE sanity saver and I wish that I had this idea earlier. 

## The method

```D
void main(string[] args)
{
    if (args.length == 1)
        f = stdout;
    else
    {
        f = File(args[1],"w");
        if (args.length == 3 && args[2] == "-d")
        {
            _debug = true;
        }
    }
```

select the output file and enable runtime debugging.
Next we output the licence imports and a throw if error code is in error struct 

```D
    fwrite(
`//Autogenerated.
//licence omitted for brevity 
module vk;
import vulkan;

import std.algorithm;
import std.string;
import std.array;

// this is returned by the wrapped methods that call functions that return VkResult by value and
// the actual return value as a pointer parameter
struct ReturnResult(T)
{
    T       t;
    Result  result; //the error code
    
    auto get()
    {
    	import std.conv : to;
        if (result < 0)
            throw new Exception("Vulcan call failed: " ~ result.to!string);
        return t;
    }
    alias get this;
}

`   );
```

I feel I should explain this a bit more. `ReturnResult` is a struct template that holds an instance of its 
template parameter and `Result` (a renamed `VkResult` but we'll get to that later). `result` is placed after
`t` so a to not muck up the alignment of `t` and consume more memory than is needed
. If we go `auto handle  = someFuncCall(args);` and then use `handle` one of two thing will happen. If the call succeeded then `handle` will effectively alias
itself to `handle.t` and will continue on and behave as if it were of type `T`. However if the call 
failed and `handle.result < 0` i.e. an error code, and then we use it is some way e.g. call 
one of its methods it will throw, but only then. We still have time to check `handle.result` 
for an error condition to avoid throwing if we want.

Next comes the main "loop" .Its actually unrolled at compile time because its a for each on a Tuple.

```D
foreach(m; __traits(allMembers, vulkan))
{
	if(!(m in emittedSymbols))
    	emittedSymbols[m] = false;
	//other code here
	foreach(k,v;emittedSymbols)
  	{
    	if(!v) writeln(k);
	}
}
```

`__traits(allMembers, vulkan)` yields a Tuple of strings.
If we run this we get a list of all the symbols declared in the Vulkan API and it's extensions.
(You probably want to pipe this through `sort` and redirect it to a file as its rather long)
We will use this to track our progress.

Next we want to decide what to do based on what the "type" of the symbol. Looking through that list
we can see that there are some symbols that we don't care about, basically everything that came from 
a `#define` all of the function prototypes `PFN_vk*` and most of the manifest constants as these are 
still available from the C API module.

We can remove them from our list of things to be done

```D
if (m.startsWith("PFN_vk") || m.startsWith("VK"))
{
	emittedSymbols[m] = true;
}
```

this leaves us with enums (in the C sense), functions, structs, handles. Let's start with the enums
first.

```D
static if( __traits(compiles, mixin(m~"."~m.camelToUpper_ ~ "_MAX_ENUM")) 
			|| m.endsWith("Bits"))
{
	emitEnum!m;
}
```

the function emitEnum requires a template string argument as we will do further introspection using it.

```D
void emitEnum(string m)()
{
    emittedSymbols[m] = true;
    if (m.endsWith("Bits"))
    {
        fwrite("\nenum ", m[VKL .. $-"Bits".length] , " : uint \n{");
    }
    else
    	fwrite("\nenum ", m[VKL .. $] , " : int \n{");
    
    foreach(e; NoDuplicates!(EnumMembers!(__traits(getMember,vulkan, m))))
    {
        string es = e.to!string;
        if ( es.endsWith("_RANGE_SIZE")|| es.endsWith("_MAX_ENUM"))
            continue;
        static if ( m.endsWith("Bits"))//N.B: must be static if
        {
            if (es.endsWith("_FRONT_AND_BACK"))
                es = "FrontAndBack";
            else if (es.endsWith("_BIT"))
                es = es.upper_ToCamel.findSplitAfter(m[VKL .. $ - "FlagBits".length])[1][ 0 .. $ - "Bit".length];
            else
                es = es.upper_ToCamel.findSplitAfter(m[VKL .. $ - "FlagBits".length])[1];
        }
        else
            es = es.upper_ToCamel.findSplitAfter(m[VKL .. $])[1];
        // prepend an 'e' to the start of the name because some start with a number
        // so consistency and stuff.
        fwrite("e", es, " = ",e.to!int ,",");
    }
    fwrite("\n}\n");
}

```

We filter out duplicate members as those are for aiding for iteration in C.
Some 'if's must be static as normal runtime 'if's would generate invalid indices, despite the fact that 
that branch would never be taken. And because the string is known at compile time dmd rejects it statically.

This leaves us with structs and functions and handles to go. 

Back to the main loop. As I am on a 64 bit machine (i.e. version = D\_LP64) all handles end with "\_T", including the non-dispatchable ones. This makes detecting them as simple as 

```D
else static if (m.endsWith("_T")) // instansiated with VK_DEFINE{_NON_DISPATCHABLE}_HANDLE
{
    emitHandle!m;
}
```

There are several thing the we need to do here. Emit the {con,de}structors (set up and call vk*{Create,Destroy}Foo)  emit the CreateInfo and a constructor for it, loop through and find all functions that have this handle as their first parameter and emit them as methods to our wrapper. There are several things that cause problems here, the most annoying being that the names of aliased types appear as the aliaed type NOT the alias, i.e. introspecting a function that takes a size_t will give a ulong not a size_t. This means that we have to check for this EVERYWHERE, function parameter lists, struct fields, and we have to surmise the correct type or it won't be portable (see non-dispatchable handles) or safe ( calling a function with the wrong flags type). So to aid in this we have 

```D
string fixup_T(string s)
{
    if(s.endsWith("_T*"))
    {
        return s[0..$-"_T*".length];
    }
    else if (s.endsWith("_T**"))
    {
        return s[0..$-"_T**".length] ~ "*";
    }
    else if (s.endsWith("_T**)"))
    {
        return s[0..$-"_T**)".length] ~ "*)";
    }
    else
        return s;
}
``` 

to turn handles back to the portable type. Now on to emitHandle preamble 

```D
void emitHandle(string m)()
{
    emittedSymbols[m] = true;
    if(  cannonicalName.endsWith("KHR") ||
         cannonicalName.endsWith("EXT"))
    {
    	return;//ignore extensions
    }
    enum cannonicalName = m[0 .. $-"_T".length];
    enum createInfoName = cannonicalName ~ "CreateInfo";
    emittedSymbols[createInfoName] = true; // might not exist but we don't care
    enum createFnName   = "vkCreate" ~ cannonicalName[VKL .. $];
    enum destroyFnName  = "vkDestroy" ~ cannonicalName[VKL .. $];

    fwrite("struct ", cannonicalName[VKL .. $], "\n{\n");
    enum createFlagsName    = cannonicalName ~ "CreateFlags";
    emittedSymbols[createFlagsName] = true;//ditto
    enum mainVarName        = cannonicalName[VKL .. $].toLower();
    fwrite( cannonicalName, "\t", mainVarName,";\n");
    fwrite("alias ", mainVarName ," this;\n");
```

next we emit the constructor and create info by querying if the functions we will forward to exist and loop through the symbols in vulkan and emit them as a method if they are. Some handles are created from an instance or a device for those we want to not emit a constructor (as it will be a method of either Device or Instance) and hold a reference to it for the destructor.

```D
static if(__traits(hasMember, vulkan,createFnName) &&
            // if the create function of this type is a method of another type don't emit it here
            !Parameters!(__traits(getMember, vulkan,createFnName))[0].stringof.endsWith("_T*"))
{
    emitTors!m;
}
//build CreateInfo
static if(__traits(hasMember, vulkan,createFnName))
{
    emitCreateInfo!m;
}
foreach(m2; __traits(allMembers, vulkan_input))
{
	static if (__traits(isStaticFunction,__traits(getMember,vulkan,m2)))
 	{
     	     //ignore trailing '*'
     	     static if (Parameters!(__traits(getMember,vulkan_input, m2))[0].stringof[0..$-1] == m)
     	     {
     	         ...
     	     }
 	}
}
```

The first dichotomy in the API methods is whether they return `void` or `VkResult`. The second is whether or not they return a result through a pointer in their last argument(s). In addition to this we have to handle len(\*)/ptr to array and back conversions, inspect parameters -> array in method -> len/ptr for calling. The methods that take a len\*/ptr pair we need to call twice, once with null do determine the size and once again to retrieve the data. The constructors are a special case of returns VkResult and returns through a pointer. In the interest of keeping this post short I will only present the VkResult pointer returning case, the rest are left as an exercise to the reader ;). The other cases are relatively similar but the structure is the same, constrain for the case,determine the correct return type, introspect the parameters, detect arrays, output the argument and then the call to the wrapped function. But I haven't yet factored out the code. 

```D
static if(!is(RT == void) && RT.stringof == "VkResult"
        && (PR[$-1].stringof.canFind("_T**") || (PR[$-1].stringof[$-1] == '*' && PR[$-1].stringof[$-2] != 'T'))
        && !PR[$-1].stringof.canFind("const")
        && PR.length >1)
 {
      emittedSymbols[m2] = true;
      bool returnsarray;
      if (PI[$-2].canFind("Count") && !(PI[$-1].canFind("const")))
      {
            returnsarray = true;
            fwrite("ReturnResult!(",PR[$-1].stringof[VKL..$-1].fixup_T, "[])");
      }
      else
      {
          if (PR[$-1].stringof == "void*") //getPipelineCacheData
          {
                returnsarray = true;
                fwrite("ReturnResult!(void[])");
          }
          else if (PR[$-1].stringof == "void**") //mapMemory
          {
               fwrite("ReturnResult!(void*)");
           }
           else if (PR[$-1].stringof == "uint*")
           {
                fwrite("ReturnResult!(uint)");
           }
           else
           {
                  fwrite("ReturnResult!(",PR[$-1].stringof[VKL .. $-1].fixup_T,")");
           }
      }
     fwrite(emitName, "(");
     for(auto i=1; i < pits.length-1;i++)
     {
         if (returnsarray && pits.length == 3)//handle + len + ptr
         {
            fwrite(")");
            break;
         }
         bool lastRound = (returnsarray || emitName.startsWith("create")) && (i == pits.length-2);
         fwrite( params[i].fixup_T, "\t", pits[i] ,!lastRound ? ",":")");
         if (lastRound)
               break;
      }
      if (!returnsarray && !emitName.startsWith("create"))
      {
          fwrite(params[$-1].fixup_T, "\t_", pits[$-1],")");
      }
      fwrite("{");
      fwrite("typeof(return) _result;");
      if (returnsarray)
      {
           if(PR[$-1].stringof == "void*")// for vkGetPipelineCacheData
           {
                  fwrite("ulong _len;");
           }
           else
           {
                fwrite("uint _len;");
           }
          fwrite(m2,"(", mainVarName,",");
          foreach(p;PI[1..$-2])
           {
                fwrite( p,",");
           }
            fwrite("&_len,");
            fwrite("null);");
            fwrite("typeof(_result.t) _p;\n\t\t_p = new typeof(_p)(_len);");
            fwrite("typeof(_p.ptr) _ptr = _p.ptr;\n");
            fwrite("_result.result = cast(typeof(_result.result))",m2,"(", mainVarName,",");
            foreach(p;PI[1..$-2])
            {
                  fwrite(p,",");
            }
            fwrite("&_len,");
                        
            fwrite("cast(",PR[$-1].stringof.fixup_T,")_ptr);");
            fwrite("_result.t = _p;");
       }
       else
       {
            fwrite("_result.result = cast(typeof(_result.result))", m2,"(");
            fwrite( mainVarName,",");
            foreach(p;PI[1..$-1])
            {
                fwrite( p,",");
            }
            fwrite("cast(",PR[$-1].stringof.fixup_T,")&_result.t);");
       }
       fwrite("return _result;");
       fwrite("}\n");
 }

```

The generated code looks like 

```D
struct Instance
{
	VkInstance	instance;
	alias instance this;
	const(VkAllocationCallbacks*) ac;

	this(
	const ref CreateInfo createInfo,
	const(VkAllocationCallbacks*)	pAllocator
	)
	{
		vkCreateInstance(&createInfo.ci,pAllocator,&instance);
		ac = pAllocator;
	}
	~this()
	{
		vkDestroyInstance(instance,ac);
	}
	static struct CreateInfo
	{
		VkInstanceCreateInfo	ci;
		alias ci this;
		this(
			const(VkApplicationInfo*)		_pApplicationInfo,
			string[]		_enabledLayers,
			string[]		_enabledExtensions
			, uint 		_flags =0
			)
		{
		auto __enabledLayers= _enabledLayers.map!(s=> s.toStringz).array;
		auto __enabledExtensions= _enabledExtensions.map!(s=> s.toStringz).array;
		ci = typeof(ci)(
		cast(typeof(ci.sType))StructureType.eInstanceCreateInfo,
		null,
		0,
		_pApplicationInfo,
		cast(uint)__enabledLayes.length,
		__enabledLayers.ptr,
		cast(uint)__enabledExtensions.length,
		__enabledExtensions.ptr);
		}

	}

	ReturnResult!(PhysicalDevice[])	physicalDevices()
	{
		typeof(return) _result;
		uint _len;
		vkEnumeratePhysicalDevices(instance,
			&_len,
			null);
		typeof(_result.t) _p;
							_p = new typeof(_p)(_len);
		typeof(_p.ptr) _ptr = _p.ptr;

		_result.result = cast(typeof(_result.result))vkEnumeratePhysicalDevices(instance,
		&_len,
		cast(VkPhysicalDevice*)_ptr);
		_result.t = _p;
		return _result;
	}
	...
}
```

and we can use it like this

```D
auto ici = Instance.CreateInfo(null,[],[]);
auto instance = Instance(ici,null);
auto pdevs = instance.physicalDevices();
```

So there you have it! Generation of a modified interface through introspection.

I would like to thank Adam Ruppe, Ali Cehreli, Jack Stouffer and the D forumites for their help in debugging and suggestions. I would also like to thank Rikarin for providing the translation of the C header.