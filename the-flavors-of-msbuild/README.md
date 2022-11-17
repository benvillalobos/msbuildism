# The Flavors of MSBuild
Here's a brief overview of how MSBuild ships with different products.

## Visual Studio MSBuild, the Neapolitan Ice Cream
You run MSBuild on the command line via a Developer Command Prompt.

The different flavors of MSBuild live in Visual Studio under `<path-to-VS>/2022/<your-version>/MSBuild/Current/Bin/`.

### The three flavors: x86, x64, and arm64
The flavor of MSBuild that's chosen is the same architecture as your computer. For example, 64-bit computers will pick `/MSBuild/Current/Bin/amd64/MSBuild.exe`.

It's important to understand this distinction when inserting custom DLL's. If you pick the wrong folder you may not see your changes reflected in a build.

x86 MSBuild.exe: `bin/`
x64 MSBuild.exe: `bin/amd64`
arm64 MSBuild.exe `bin/arm64`

## .NET Core MSBuild (Vanilla)
If VS' MSBuild is neapolitan, the .NET SDK's msbuild is vanilla. Because .NET Core is cross-platform, we only have a single version of MSBuild here.

`dotnet msbuild` is the equivalent of `msbuild` from a developer command prompt, and would accept all the same arguments.

The entrypoint here is `MSBuild.dll`, and replacing any custom binaries would be placed directly into your SDK's folder. Your folder might look something like `C:\Program Files\dotnet\sdk\7.0.100`.