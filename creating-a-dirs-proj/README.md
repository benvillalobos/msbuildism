# Creating a dirs.proj

#### What's a dirs.proj?
A dirs.proj is a project that builds other projects via `ProjectReference` items. It does not need to have (and sometimes shouldn't have) the `.proj` extension.

## Examples of these projects
- I built a packaging project (which is technically a dirs.proj) for the [Microsoft.NET.Build.Containers](https://github.com/dotnet/sdk-container-builds/blob/main/packaging/package.csproj) repo. See [the-packaging-project](..\the-packaging-project\README.md) for info on how to create one.
- [The MSBuild repo does this in a somewhat unique way](https://github.com/dotnet/msbuild/tree/main/src/Package). The VSSetup projects import the "GetBinPaths" targets and gather the outputs themselves. The "GetBinPaths" targets files include the `ProjectReference` items.