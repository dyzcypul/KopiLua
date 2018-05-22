# Kopi Lua v 0.1 (Alpha) #
Copyright © 2009 Mark Feldman. All Rights Reserved.

## Introduction ##

Kopi Lua is a C# port of the Lua v.5.1.4 virtual machine, parser, libraries and command-line utilities. KL is not simply an interface to a compiled DLL, all original code has been ported over to managed C#.

Please note that this is an alpha release only. I've tested it against numerous test samples and it seems to mostly hold up as well as the x86 C implementation upon which it was based. That said, I'm sure there are many bugs lurking within, so if you encounter any then please let me know. Bear in mind that managed environments are notoriously restrictive, so I'm sure the VM can be forcibly crashed without too much trouble. The code has not been subjected to rigorous scrutiny and is by no means meant to be regarded as secure in its current state.

## Download ##

The latest version of Kopi Lua can be downloaded from the GitHub repository: [https://github.com/Myndale/KopiLua](https://github.com/Myndale/KopiLua)

## Projects and Solutions ##

Kopi Lua provides a project file for Microsoft Visual C# 2008 Express Edition containing three solutions:

- KopiLua is a class library containing the port of the core Lua VM and parser.
- Lua is a Windows console application port of the Lua 5.1.4 interpreter.
- Luac is a Windows console application port of the Lua 5.1.4 compiler.

Note that although the KopiLua solution compiles into a DLL, the source files have also been added to Lua and Luac so that they produce standalone executables. At this time all projects containing the Lua core files must be compiled with the following compilation flags, changing them will break the compile:

    LUA_CORE;_WIN32;LUA_COMPAT_VARARG;LUA_COMPAT_MOD;LUA_COMPAT_GFIND;CATCH_EXCEPTIONS

The only new flag is CATCH_EXCEPTIONS and this is the one flag that can be safely disabled. When the Lua VM encounters an error it typically breaks out of execution with a long jump, although this can be changed to throw an exception in C++ builds. The C# port also throws exceptions and catches them higher up the execution heirarchy, but the catch is only done when CATCH_EXCEPTIONS is specified during the build. In general, this is the desired behavior, but turning it off can make it easier to track down exceptions in the code while debugging the VM code.

## Known Issues ##

- There is currently no support for locale.
- There are several differences in the behaviour of calls to operating system functions, particularly with undefined operations e.g. renaming an open file works in C but throws an exception in C#.
- The strftime() function has not yet been ported, so os_date() currently has an incomplete implementation.
- The handling of lua_str2number() errors is a bit of a hack job at the moment. If the number being converted is outside the allowable range then the function is supposed to return either positive or negative infinity. The current code will return the correct value in most non-pathological cases but it needs additional work to make it more robust.
- Memory allocation and tracking has not been properly implemented with user data objects so the behaviour of the garbage collector won't be 100% identical to the C version when user data objects are in use. This is currently high on the list of things to be fixed.

## Porting Notes ##

The primary goal of KL was for the port to match the behaviour of the C code as closely as possible. A secondary goal was to maintain a code base that is as close to the original C version as possible in order to assist with future version upgrades and patches.

The C code has been modified to remove all structs with the exception of Value; all other objects are passed by reference. This was done to assist in a future port to Java and/or any other languages that don't support passing objects via the stack. At some point the KL code will be modified so that Value is implemented with a class as well.

The project currently does not employ the use of any pre-processor, so all function #defines etc have been implemented by expanding them out to full-bodied functions. This will be changed in future as it obviously has a negative effect on performance.

The original source is littered with C-style conditionals of the form “if (val)” where val is a pointer or integer variable. These have all been expanded out to their explicit forms i.e. “if (val != 0)” etc.

There is a significant amount of mixing of 16-bit/32-bit and signed/unsigned arithmetic; in most cases a simple cast was enough to fix this. It will, however, need to be cleaned up at some point as it may be having an effect on performance.

The C code makes judicious use of pointer arithmetic, which presented a particularly challenging technical difficulty. When KL creates or resizes an array it loops through every element and stores a pointer to the array as well as it's position index within it. Overloaded operators are then used to reference neighbouring elements. Similarly, statements that use indirection such as “*s” etc have been converted to array look-ups of the form “s[0]”. In certain cases the code works with pointer pointers (e.g. GCObject **), this was solved by creating an intermediate class to represent a pointer-to-array (i.e. CGObjectRef).

The handling of strings is a very specific example of arrays that needed to be solved with a custom object (CharPtr) which stores a pointer to the char array and an offset into that array. Operator overloading then allows for the code to work with the string similar to the way in which the original code works with the char *. One notable difference here is the fact that in the original C code a char * passed to a function resides on the stack, so the function can change it at will. In KL the CharPtr object is a reference, so if any changes to the index itself are made then a local copy of the CharPtr has to be made first so that the object passed in by the calling function isn't corrupted. Chars are stored internally as Unicode but are treated as ASCII when converting to and from their byte representations i.e. the source code uses Unicode chars but from the perspective of the Lua scripts all strings appear as ASCII.

Several variables in the C code use C# reserved words, notably “base” and “ref”. These have been post-fixed with an underscore i.e. “base_” and “ref_”.

In order for KL to behave exactly in the same way as the original code the Lua garbage collector needed to be ported as well. This is particularly important for the behaviour of things such as weak tables etc. This presented a few problems, however, since the garbage collector relies on the total size of allocated objects, and generally speaking the “size” of a class is a meaningless concept in managed environments like C#. In order to get around this the memory allocation routines have been hard-coded with the size of all objects in the original C code. This is used to ensure that the value of the totalbytes member in global_State behaves identically to its C counterpart, which in turn ensures that the garbage collector behaves the same as well (an obvious downside is that this value is no longer an accurate representation of the true number of bytes allocated). When the Lua garbage collector frees an object it simply releases all handles to it and allows the C# garbage collector to then truly free the object as and when it chooses.

The Lua source code calls a lot of standard C functions (memcpy, gets, ungetc, fopen etc, to name a few). C# implementations for all of these have been developed and added to the bottom of the luaconf.cs file.

The most challenging problem by far was overcoming the C code's use of unions. Unions of the type used by the Lua code are not possible in managed code so a variety of tricks were employed to remove them. In cases where unions are used to save memory (e.g. Value) the union itself was unecessary and the members were simply added to a regular class. In some cases, e.g. the Closure class, a common header was added to the members CClosure and LClosure and casting was used depending on the closure type; this has been implemented by seperating the header out into a seperate class (ClosureHeader), adding a pointer to it to both the CClosure and LClosure members and using inherited operator overloading so that they can be “cast” to the ClosureHeader type.

There were many language-specific features in the code that caused problems such as using typed enums with ints, using fall-through in switch statements, failing to break from default handlers in switch statements, using commas to seperate statements, using “0” to represent NULL etc. In general though, the KL code follows the original C code very closely and shouldn't be too difficult to work with if you're familiar with the original code base.

## Licensing Rights ##

Kopi Lua is licensed under the terms of the MIT license, the same terms under which Lua is licensed. This means that like Lua, Kopi Lua is free software and can be used for both academic and commercial purposes at absolutely no cost. The terms of the MIT license state:

"Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions: The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software."

The full text of the license can be found on the Lua site at [http://www.lua.org/license.html](http://www.lua.org/license.html), a copy of the original Lua license is also distributed with Kopi Lua. Additional details can be found at [http://en.wikipedia.org/wiki/MIT_License](http://en.wikipedia.org/wiki/MIT_License).

The C# implementation of printf that Kopi Lua uses was developed by Richard Prinz and is bundled as-is and unmodified. Its use is subject to the terms of The Code Project Open License (CPOL) 1.02 which state that both source code and executable "can be used in commercial applications", "can be redistributed" and "can be modified to create derivative works".

## Acknowledgements ##

Thanks to Niki Bowe, Nick McVeity and Pete Henry at Infinite Interactive for introducing me to Lua and engaging in so many discussions about it. Richard Prinz saved me a considerable amount of work by making his C# implementation of printf available to the public. And of course thanks to the creators of Lua and the members of the Lua mailing list for all their ongoing work.

Mark Feldman, November 2009.

mailto:lua NOTTHISBIT @ppl-pilot.com