[//]: # (title: AssemblyInfo Patcher)
[//]: # (auxiliary-id: AssemblyInfo Patcher)

The _AssemblyInfo Patcher_ [build feature](adding-build-features.md) allows setting a build number to an assembly automatically, without having to patch the `AssemblyInfo.cs` files manually. When adding this build feature, you only need to specify the version format. You can use TeamCity [parameter references](using-build-parameters.md).

AssemblyInfo Patcher should be used with the [automatic checkout](vcs-checkout-mode.md) only: after this build feature is configured, it will run __before the first build step__. TeamCity will first perform replacement in the files found in the build checkout directory and then run your build.

TeamCity searches for all AssemblyInfo (including GlobalAssemblyInfo) files: `.cs`, `.cpp`, `.fs`, `.vb` in their standard locations under the checkout directory and replaces the parameter for the `AssemblyVersion`, `AssemblyFileVersion`, and `AssemblyInformationalVersion` attributes with the build number you have specified in the TeamCity web UI. Read more about these attributes in the [Visual Studio documentation](https://docs.microsoft.com/en-gb/troubleshoot/visualstudio/general/assembly-version-assembly-file-version).

At the end of the build, the files are reverted to the initial state.

Note that this feature will work only for standard projects, that are created by means of the Visual Studio wizard, where all the `AssemblyInfo` files and content have a standard location.

>For replacing a wider range of values in a larger number of files, consider using the [File Content Replacer](file-content-replacer.md) build feature.
