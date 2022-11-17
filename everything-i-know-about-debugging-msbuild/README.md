# Seriously, README
Debugging MSBuild is difficult. There are tons of different ways to go about it, but this is what I've found works for me in my 3+ years on the team.

## The Binlog

## Debugging into Custom MSBuild Tasks


## Debugging into MSBuild
First, you need to understand [which MSBuild you want to debug into](). Once you know which one, pick your flavor.

### Setting Up Visual Studio for Debugging
- Open VS
- Tools -> Options -> Debugging
    - Uncheck "Enable Just My Code"
    - Check .NET Framework source stepping
    - Uncheck "Require source files to exactly match the original version)
        - This may be necessary if you're using the MSBuild repo's source to step through, and the source doesn't match 1:1.

### Setting Up A Custom MSBuild To Debug Into
This is for those _very_ complex scenarios where you need to manually modify the code and test builds using those new binaries.

```
cd <wherever you place your code>
git clone https://github.com/dotnet/msbuild
cd msbuild
<if necessary, make changes or checkout a specific branch>
build.cmd
```

Running build.cmd is the proper way to build the repository. Once built, your binaries should exist at: `<repoRoot>/artifacts/bin/bootstrap/<yourTargetFramework>/MSBuild/`.

**IMPORTANT NOTE**: [All the different flavors of MSBuild]().
---

### MSBuild in Visual Studio


### MSBuild in the SDK
