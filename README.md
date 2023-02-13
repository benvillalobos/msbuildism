# MSBuildism
Because MSBuild is just as complex as it is powerful, it has quirks that I like to call "msbuildisms." When your incremental build starts to fail for seemingly no reason, or when changing one property suddenly causes your build to fail in four different places. Those are MSBuildisms.

It's important to acknowledge that, because MSBuild is so flexible, **there is no single way to accomplish something**. Every repo is unique, every build is complicated, and every workaround is just one of many you could have done. Your mileage may vary.

This repo is a collection README's that document what I've learned in my ~4 years on the MSBuild team ❤️. I hope it helps make your MSBuildisms a little easier to solve.

# Quick Links
- [The Flavors of MSBuild](./the-flavors-of-msbuild/README.md) Did you know there are actually _three_ MSBuild.exe's that come with your install?
- [Debugging MSBuild](./debugging-msbuild/README.md) The tricks I learned to debug various forms of MSBuild.
- [Diagnosing Build Failures](./build-spelunking/README.md) How to "Go to definition," and more!
- [Including Generated Files Into Your Build](./including-generated-files/README.md) Includes a simplified explanation on `Evaluation` vs `Execution` phases of your build.
- [MSBuild & the .NET SDK](./debugging-msbuild/README.md#msbuild--the-net-sdk) Notes on debugging custom MSBuild bits, SDK bits, or both!
- [Multi-Platform Builds](./multi-platform-builds/README.md) (e.g. how to get a single build to generate x64 & arm64 outputs in one go)
- [The Packaging Project](./the-packaging-project/README.md) How to create a separate project that packages all of your outputs into a single NuGet Package `*.nupkg`.
- [Notable Targets](./notable-targets/README.md) Unofficial docs on some relevant targets during the build.
- [Tools and Resources](./tools-and-resources/README.md) Links for further reading.

## How to Use MSBuild
`msbuild` works when called within a `Developer Command Prompt` and uses that Visual Studio's respective copy of `MSBuild.exe`
`dotnet msbuild` is the .NET SDK's equivalent. It accepts all the same arguments as `msbuild.exe`.

## Set Up MSBuild To Auto-Launch In Windows Terminal
Here's the profile I used for windows terminal:
```json
{
    "commandline": "cmd.exe /k \"C:\\Program Files\\Microsoft Visual Studio\\2022\\Dogfood\\Common7\\Tools\\VsDevCmd.bat",
    "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
    "hidden": false,
    "name": "Dev Command Prompt"
},
```
**NOTE:** I found that using `cmd.exe /k` and a path that contained a space did _not_ play well with VS Code. To work around this, I created a shortcut to a directory that didn't contain a space. For example: `C:\\src\VsDevCmd.lnk`. Alt + click + drag is an easy way to create a shortcut.