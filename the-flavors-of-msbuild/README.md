# The Flavors of MSBuild
Here's a brief overview of how MSBuild ships with different products.

## Visual Studio MSBuild, the Neapolitan Ice Cream
The different flavors of MSBuild live in the Visual Studio folder. Typically, the path looks like `C:/Program Files/Microsoft Visual Studio/<year>/<version>/MSBuild/Current/Bin/`.

### The three flavors: x86, x64, and arm64
The flavor of MSBuild that's chosen is the same architecture as your computer. For example, 64-bit computers will pick `C:/Program Files/Microsoft Visual Studio/<year>/<version>//MSBuild/Current/Bin/amd64/MSBuild.exe`. Note that the x64 and arm64 flavors of MSBuild will use the dll's in the "root" folder, `C:/Program Files/Microsoft Visual Studio/<year>/<version>/MSBuild/Current/Bin/`, whenever possible.

### If You Want To Replace VS Binaries...
First, understand which binaries you want to replace. 

x86 MSBuild.exe:  `bin/MSBuild.exe`
x64 MSBuild.exe:  `bin/amd64/MSBuild.exe`
arm64 MSBuild.exe `bin/arm64/MSBuild.exe`

Note: For the most part, replacing the binaries in `bin/` is sufficient. Just in case, you can check `MSBuild.exe.config` and look for `BindingRedirect` entries to see which binaries are used for that particular flavor of MSBuild.

Next, create a local backup of the binaries or you could break your Visual Studio install. 

Finally, manually back up and replace individual binaries, or use [our recommended script](https://github.com/dotnet/msbuild/blob/main/scripts/Deploy-MSBuild.ps1) if you're unsure of what to replace.

## .NET Core MSBuild, the Vanilla one
We only need a single version of MSBuild because .NET Core is cross-platform.

### If You Want To Replace .NET SDK Binaries...
First, make sure you know which SDK you want to place custom binaries into. From your project's root, run:

```
dotnet --info
where dotnet
```

This will give you which version of the SDK you want and where your SDK lives. Typically, it's something like `C:\Program Files\dotnet\sdk\7.0.100`.

Finally, manually back up and replace individual binaries, or use [our recommended script](https://github.com/dotnet/msbuild/blob/main/scripts/Deploy-MSBuild.ps1) if you're unsure of what to replace.

**Note**: Make sure you replace them with .NET Core binaries! These come from `<msbuild-repo>/artifacts/bin/botstrap/<not-net472>/MSBuild/Current`.

## How To Replace Binaries
The [Deploy-MSBuild.ps1](https://github.com/dotnet/msbuild/blob/main/scripts/Deploy-MSBuild.ps1) script in the MSBuild repo will automatically back up and replace _all_ relevant binaries. Use this method when you're unsure of exactly what to copy.

The usage is `Deploy-MSBuild.ps1 --Destination=<path-to-your-install>`