# Seriously, README
Debugging MSBuild is difficult. There are tons of different ways to go about it, but this is what I've found works for me in my 3+ years on the team.

## The Binlog

## Debugging Into MSBuild
0. [Understand which MSBuild you want to debug into](..\the-flavors-of-msbuild\README.md), then pick your flavor.
1. [Set up Visual Studio for Debugging]().
1. [Set up the MSBuild repo]() if you're modifying the source code or are unable to step into MSBuild's source for some reason.
1. Launch a developer command prompt
1. Set the right environment variable: `set MSBUILDDEBUGONSTART=1`
    - 1 = Show a prompt that displays which debuggers to attach to.
    - 2 = Display a process ID, manually attach to it, then continue the build.
1. Run your scenario, break into the process using Visual Studio.
1. [Breaking into the right function]().

### Setting Up Visual Studio for Debugging
- Open VS
- Tools -> Options -> Debugging
    - Uncheck "Enable Just My Code"
    - Check .NET Framework source stepping
    - Uncheck "Require source files to exactly match the original version)
        - This may be necessary if you're using the MSBuild repo's source to step through, and the source doesn't match 1:1.

### Set Up The MSBuild Repo
```
cd <wherever you place your code>
git clone https://github.com/dotnet/msbuild
cd msbuild
<if necessary, make changes or checkout a specific branch>
build.cmd
devenv MSBuild.Dev.slnf
```

**Note:** Say you want to make sure your MSBuild source code matches the version that's in the .NET SDK or Visual Studio exactly. To find the hash to check out, run `msbuild --version` or `dotnet msbuild --version`. You should see a message like `MSBuild version 17.4.0+18d5aef85 for .NET Framework`, `18d5aef85` is the hash to checkout.

### Replacing MSBuild Binaries
This is for those _very_ complex scenarios where you need to manually modify MSBuild and test builds using those new binaries. I typically do this when logging isn't an option and I need a task to spit out some `Console.WriteLine`s.

Step 1: [Set up the MSBuild Repo]().
Step 2: `scripts/Deploy-MSBuild.ps1 <path-to-VS-or-SDK>`
Step 3: ???
Step 4: Don't forget to replace your custom bits with the originals afterward.

```cmd
cd <wherever you place your code>
git clone https://github.com/dotnet/msbuild
cd msbuild
<if necessary, make changes or checkout a specific branch>
build.cmd
scripts/Deploy-MSBuild.ps1 <path-to-VS-or-SDK>
```

If you're manually replacing binaries, they should exist at: `<repoRoot>/artifacts/bin/bootstrap/<yourTargetFramework>/MSBuild/`. Use net472 for MSBuild under VS, and net7.0 for .NET SDK.

## Breaking Into the Right Function
Once your debugger (I'm assuming Visual Studio) is attached to MSBuild, it should be somewhere in XMake.cs at `Debugger.Launch();`. Now it's time to set breakpoints.

1. Debug -> Windows -> Breakpoints
1. New -> Function Breakpoint

It's easiest to **include the fully-qualified name** as your function breakpoint. That means `Namespace.ClassName.FunctionName`. For example, 


### Debugging into MSBuild Tasks
All MSBuild tasks have the same entrypoint, `Execute()`. Hooking into MSBuild tasks means adding a breakpoint that looks similar to `Microsoft.Build.Tasks.Message.Execute`. Custom MSBuild tasks work the same way.