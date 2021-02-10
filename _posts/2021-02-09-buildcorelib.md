---
layout:  post
title:   "Howo to build System.Private.CoreLib code"
summary: "Steps to build code that lives in the middle subset."
date:    2021-02-09 19:36:00 -0800
categories: all
---

When working in the Runtime repo, sometimes you need to make changes in code that lives in the `System.Private.CoreLib` folder.

The code that lives in `System.Private.CoreLib` is neither part of the `libraries` subset, or of the `clr` subset. Instead, it has its own subset: `clr.corelib`. Keep in mind that it will be built with the same configuration used for `clr`.

The unit tests for code that lives in `System.Private.CoreLib` are located in `System.Runtime` and `System.Runtime.Extensions`.

The solution is located in `runtime\src\coreclr\System.Private.CoreLib\System.Private.CoreLib.sln`.

The project is split in a special way in these files:

- `runtime\src\coreclr\System.Private.CoreLib\System.Private.CoreLib.csproj`.
- `runtime\src\libraries\System.Private.CoreLib\src\System.Private.CoreLib.Shared.shproj`.
- `runtime\src\libraries\System.Private.CoreLib\src\System.Private.CoreLib.Shared.projitems`.

An example of a Type that lives in `System.Private.CoreLib` is `System.IO.Path`. This type has its elements located in the following places:

- Solution: `runtime\src\coreclr\System.Private.CoreLib\System.Private.CoreLib.sln`.
- Source code: `runtime\src\libraries\System.Private.CoreLib\src\System\IO\Path.cs`.
- Ref file: `runtime\src\libraries\System.Runtime\ref\System.Runtime.cs`.
- Unit test project: `runtime\src\libraries\System.Runtime.Extensions\tests\System.Runtime.Extensions.Tests.csproj`.
- Unit tests source code: `runtime\src\libraries\System.Runtime.Extensions\tests\System\IO\*`.

When you make changes to existing `Path` members, you must build:

- Build everything once in Debug so you can step into the code when debugging:
    `build.cmd clr+libs+libs.pretest -c debug`

- When you make changes to the `Path` members, you build:
    `build.cmd clr.corelib+libs.pretest -c debug`

- When you add new APIs to the ref file, you build its ref project:
    `dotnet build runtime\src\libraries\System.Runtime\ref\System.Runtime.csproj`

- When you want to run unit tests:
    `build.cmd clr.corelib+libs.pretest -c debug && dotnet build src\libraries\System.Runtime.Extensions\tests\System.Runtime.Extensions.Tests.csproj`
