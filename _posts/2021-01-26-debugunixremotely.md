---
layout: post
title:  "How to remotely debug a Unix .NET process using VS Code"
summary: "This is how I attach my VS Code debugger to a remote Unix machine running a .NET process."
date:   2021-01-26 10:27:00 -0800
categories: all
---

## Attaching to a process

When I want to run a remote process from the SSH or WSL console, and then attach VS Code to that process, I add the this configuration to the remote VS Code `launch.json` file (which happens to be the default "attach" configuration for a .NET Core process):

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

Then I add this code to my .NET program right where I want the debugger to get attached:

```cs
while (!System.Diagnostics.Debugger.IsAttached)
{
    System.Console.WriteLine($"Attach to {Environment.ProcessId}");
    System.Threading.Thread.Sleep(1000);
}
System.Diagnostics.Debugger.Break();
```

Finally, I build and run the process, wait for the message with the process ID to start showing up, then I hit Control+Shift+D to switch to the "Run" extension, make sure the dropdown at the top has ".NET Core Attach" selected, then press F5, and finally select the process with that ID from the dropdown list that shows up.


## Runtime unit tests

When running unit tests from the console in the runtime repo, you use `dotnet build /t:Test <TestProjectFile>.csproj`.

If you try to add a VS Code configuration that runs that process and attaches to it, you won't be able to step into the unit test code.

If you look at output of that command, you'll notice that the unit tests are executed by a forked process with a much different command and arguments. So if you create a configuration that executes that command directly, you'll be able to step into the unit test code.

Let's assume you want to run a specific FileSystem unit test in your remote SSH or WSL machine. You would run the following command in the console, which would generate the output below:

```
carlos@Carloss-MacBook-Pro > ~/Public/repos/runtime/src/libraries/System.IO.FileSystem/tests > IsHidden2 >
❯ dotnet build /t:Test ./System.IO.FileSystem.Tests.csproj /p:XUnitMethodName=System.IO.Tests.FileInfo_Exists.NonExistentFile
Microsoft (R) Build Engine version 16.8.3+39993bd9d for .NET
Copyright (C) Microsoft Corporation. All rights reserved.

  Determining projects to restore...
  All projects are up-to-date for restore.
...
  System.IO.FileSystem.Tests -> /Users/carlos/Public/repos/runtime/artifacts/bin/System.IO.FileSystem.Tests/net6.0-Unix-Debug/System.IO.FileSystem.Tests.dll
  ----- start Tue Jan 26 16:59:01 PST 2021 =============== To repro directly: =====================================================
  pushd /Users/carlos/Public/repos/runtime/artifacts/bin/System.IO.FileSystem.Tests/net6.0-Unix-Debug
  /Users/carlos/Public/repos/runtime/artifacts/bin/testhost/net6.0-OSX-Debug-x64/dotnet exec --runtimeconfig System.IO.FileSystem.Tests.runtimeconfig.json --depsfile System.IO.FileSystem.Tests.deps.json xunit.console.dll System.IO.FileSystem.Tests.dll -xml testResults.xml -nologo -method System.IO.Tests.FileInfo_Exists.NonExistentFile -notrait category=OuterLoop -notrait category=failing
  popd
  ===========================================================================================================
  ~/Public/repos/runtime/artifacts/bin/System.IO.FileSystem.Tests/net6.0-Unix-Debug ~/Public/repos/runtime/src/libraries/System.IO.FileSystem/tests
    Discovering: System.IO.FileSystem.Tests (method display = ClassAndMethod, method display options = None)
    Discovered:  System.IO.FileSystem.Tests (found 1 of 3316 test case)
    Starting:    System.IO.FileSystem.Tests (parallel test collections = on, max threads = 8)
    Finished:    System.IO.FileSystem.Tests
  === TEST EXECUTION SUMMARY ===
     System.IO.FileSystem.Tests  Total: 1, Errors: 0, Failed: 0, Skipped: 0, Time: 0.159s
  ~/Public/repos/runtime/src/libraries/System.IO.FileSystem/tests
  ----- end Tue Jan 26 16:59:03 PST 2021 ----- exit code 0 ----------------------------------------------------------
  exit code 0 means Exited Successfully

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:11.21
```

The _real_ commands are those executed after the _start_ line. The header even says "To repro directly". So I extracted that command and put it in the remote machine's launch.json file (right after the launch configuration mentioned in the previous section):

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Attach",
            "type": "coreclr",
            "request": "attach",
            "processId": "${command:pickProcess}"
        },
        {
            "name": "Test System.IO.FileSystem",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "Build System.IO.FileSystem",
            "program": "${workspaceFolder}/artifacts/bin/testhost/net6.0-OSX-Debug-x64/dotnet", // Slightly different location than cwd below
            "args": [
                "exec",
                "--runtimeconfig", "System.IO.FileSystem.Tests.runtimeconfig.json", // These files are located in cwd
                "--depsfile", "System.IO.FileSystem.Tests.deps.json", "xunit.console.dll", "System.IO.FileSystem.Tests.dll",
                "-xml", "testResults.xml",
                "-nologo",
                "-notrait", "category=OuterLoop",
                "-notrait", "category=failing",
                "-method", "System.IO.Tests.FileInfo_Exists.NonExistentFile"
            ],
            "cwd": "${workspaceFolder}/artifacts/bin/System.IO.FileSystem.Tests/net6.0-Unix-Debug",
            "stopAtEntry": false,
            "justMyCode": false,
            "console": "internalConsole",
            "suppressJITOptimizations": true,
            "enableStepFiltering": false
        }
    ]
}
```

The `${workspaceFolder}` expects that you are working on the `runtime` root folder.

Notice the `preLaunchTask` is the build command. I added it in the tasks.json file:

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build System.IO.FileSystem",
            "command": "dotnet",
            "type": "shell",
            "args": [
                "build",
                "${workspaceFolder}/src/libraries/System.IO.FileSystem/src/System.IO.FileSystem.csproj",
                "/property:GenerateFullPaths=true",
                "/consoleloggerparameters:NoSummary"
            ],
            "group": "build",
            "presentation": {
                "reveal": "always",
                "revealProblems": "always"
            },
            "problemMatcher": "$msCompile"
        }
    ]
}
```

Now you can press Control+Shift+D to switch to the Run extension, select "Test System.IO.FileSystem" from the dropdown list at the top, put a breakpoint at the beginning of the unit test, then press F5. The execution will build, then run the unit test, and will stop at your breakpoint.