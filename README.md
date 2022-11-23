# MSBuildism
Because MSBuild is just as complex as it is powerful, it has its own quirks that we can collectively call "msbuildisms." Your incremental build broke for seemingly no reason? Call it an MSBuildism. You changed one property and are getting the most random build failures? Yep, that's an MSBuildism.

It's important to acknowledge that because MSBuild is so flexible, **there is no single way to accomplish something**. Every build is unique, and not every solution works for every repo. Your mileage may vary. I'll post real world examples when I can.

This repo is a collection of what I've learned in my ~4 years on the MSBuild team ❤️.

# The Absolute Basics

- [Useful Tools](tools-and-resources\README.md)

## How to Use MSBuild
For the MSBuild in Visual Studio, you want to run a `Developer Command Prompt`. This should install along with VS. You call MSBuild directly via `msbuild`.

.NET Core MSBuild is typically accessed on any command prompt via `dotnet msbuild`. It accepts al the same arguments as `msbuild.exe`.
