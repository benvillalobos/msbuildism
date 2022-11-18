# Seriously, README
Debugging MSBuild is difficult. There are tons of different ways to go about it, but this is what I've found works for me in my 3+ years on the team.

## Debugging Into MSBuild
1. [Understand which MSBuild you want to debug into](..\the-flavors-of-msbuild\README.md).
1. [Set up Visual Studio for Debugging]().
1. [Set up the MSBuild repo]() if you're unable to step into MSBuild code for some reason.
1. [Replace MSBuild Binaries]() if you want to customize MSBuild.
1. [Reproduce your scenario]().
1. [Break into the right function]().

### Setting Up Visual Studio for Debugging
- Open VS
- Tools -> Options -> Debugging
    - Uncheck "Enable Just My Code"
    - Check .NET Framework source stepping
    - Uncheck "Require source files to exactly match the original version)
        - This may be necessary if you're using the MSBuild repo's source to step through, and the source doesn't match 1:1.

**NOTE:** You may want to set a custom symbol path to force Visual Studio to use your binaries for source-stepping. To do this, Tools -> Options -> Symbols -> + -> `C:/Path/To/Your/Custom/Binaries`. You may want to point it to your SDK folder, for example. Remember to undo this afterward.

### Set Up The MSBuild Repo
```
cd <wherever you place your code>
git clone https://github.com/dotnet/msbuild
cd msbuild
<if necessary, make changes or checkout a specific branch>
build.cmd
devenv MSBuild.Dev.slnf
```

**Note:** Say you want to make sure your MSBuild source code matches the version that's in the .NET SDK or Visual Studio exactly. Run `msbuild --version` (dev cmd prompt) or `dotnet msbuild --version` to figure out which commit hash to checkout. You should see a message like `MSBuild version 17.4.0+18d5aef85 for .NET Framework`, `18d5aef85` is the hash to checkout.

### Replacing MSBuild Binaries
This is for those _very_ complex scenarios where you need to manually modify MSBuild and test builds using those new binaries. I typically do this when logging isn't an option and I need a task to spit out some `Console.WriteLine`s.

Step 1: [Set up the MSBuild Repo]().
Step 2: `scripts/Deploy-MSBuild.ps1 <path-to-VS-or-SDK>`
Step 3: ???
Step 4: Don't forget to replace your custom bits with the originals afterward.

If you're manually replacing binaries, they should exist at: `<repoRoot>/artifacts/bin/bootstrap/<yourTargetFramework>/MSBuild/`. Use net472 for MSBuild under VS, and net* for .NET SDK.

## Reproduce Your Scenario
You want at least one instance of Visual Studio open, ideally with the `MSBuild.Dev.slnf` project open. If your repro scenario requires VS, you'll have two instances of VS open at the same time.

### Via Command Line
1. Launch a developer command prompt
1. Set the right environment variable: `set MSBUILDDEBUGONSTART=1`
    - 1 = Show a prompt that displays which debuggers to attach to.
    - 2 = Display a process ID, manually attach to it, then continue the build.
1. Run your scenario, break into the process using the Visual Studio that has the MSBuild source open.

### Via Visual Studio
1. Launch a developer command prompt
1. Set the right environment variable: `set MSBUILDDEBUGONSTART=1`
    - 1 = Show a prompt that displays which debuggers to attach to.
    - 2 = Display a process ID, manually attach to it, then continue the build.
1. `devenv YourProject.sln`
1. Attach with the other instance of Visual Studio.

## Break Into the Right Function
Once your debugger (I'm assuming Visual Studio) is attached to MSBuild, it should be somewhere in XMake.cs at `Debugger.Launch();`. Now it's time to set breakpoints.

1. Debug -> Windows -> Breakpoints
1. New -> Function Breakpoint

It's easiest to **include the fully-qualified name** as your function breakpoint. That means `Namespace.ClassName.FunctionName`. All MSBuild tasks, custom or not, have the same entrypoint, `Execute()`. Hooking into official MSBuild tasks might look like `Microsoft.Build.Tasks.Message.Execute`, where custom tasks have unique namespaces & class names.

## The Binlog
