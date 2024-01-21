ResolvePackageAssets - это [[MSBuild task]], которая формирует список зависимостей проекта от [[Nuget]] файлов.

Пример вызова этой таски в .NET 8:
```xml
<ResolvePackageAssets
  ProjectAssetsFile="$(ProjectAssetsFile)"
  ProjectAssetsCacheFile="$(ProjectAssetsCacheFile)"
  ProjectPath="$(MSBuildProjectFullPath)"
  ProjectLanguage="$(Language)"
  CompilerApiVersion="$(CompilerApiVersion)"
  EmitAssetsLogMessages="$(EmitAssetsLogMessages)"
  TargetFramework="$(TargetFramework)"
  RuntimeIdentifier="$(RuntimeIdentifier)"
  PlatformLibraryName="$(MicrosoftNETPlatformLibrary)"
  RuntimeFrameworks="@(RuntimeFramework)"
  IsSelfContained="$(SelfContained)"
  MarkPackageReferencesAsExternallyResolved="$(MarkPackageReferencesAsExternallyResolved)"
  DisablePackageAssetsCache="$(DisablePackageAssetsCache)"
  DisableFrameworkAssemblies="$(DisableLockFileFrameworks)"
  CopyLocalRuntimeTargetAssets="$(CopyLocalRuntimeTargetAssets)"
  DisableTransitiveProjectReferences="$(DisableTransitiveProjectReferences)"
  DisableTransitiveFrameworkReferences="$(DisableTransitiveFrameworkReferences)"
  DotNetAppHostExecutableNameWithoutExtension="$(_DotNetAppHostExecutableNameWithoutExtension)"
  ShimRuntimeIdentifiers="@(_PackAsToolShimRuntimeIdentifiers)"
  EnsureRuntimePackageDependencies="$(EnsureRuntimePackageDependencies)"
  VerifyMatchingImplicitPackageVersion="$(VerifyMatchingImplicitPackageVersion)"
  ExpectedPlatformPackages="@(ExpectedPlatformPackages)"
  SatelliteResourceLanguages="$(SatelliteResourceLanguages)"
  DesignTimeBuild="$(DesignTimeBuild)"
  ContinueOnError="$(ContinueOnError)"
  PackageReferences="@(PackageReference)"
  DefaultImplicitPackages= "$(DefaultImplicitPackages)">

  <!-- NOTE: items names here are inconsistent because they match prior implementation
	  (that was spread across different tasks/targets) for backwards compatibility.  -->
  <Output TaskParameter="Analyzers" ItemName="ResolvedAnalyzers" />
  <Output TaskParameter="ApphostsForShimRuntimeIdentifiers" ItemName="_ApphostsForShimRuntimeIdentifiersResolvePackageAssets" />
  <Output TaskParameter="ContentFilesToPreprocess" ItemName="_ContentFilesToPreprocess" />
  <Output TaskParameter="DebugSymbolsFiles" ItemName="_DebugSymbolsFiles" />
  <Output TaskParameter="ReferenceDocumentationFiles" ItemName="_ReferenceDocumentationFiles" />
  <Output TaskParameter="FrameworkAssemblies" ItemName="ResolvedFrameworkAssemblies" />
  <Output TaskParameter="FrameworkReferences" ItemName="TransitiveFrameworkReference" />
  <Output TaskParameter="NativeLibraries" ItemName="NativeCopyLocalItems" />
  <Output TaskParameter="ResourceAssemblies" ItemName="ResourceCopyLocalItems" />
  <Output TaskParameter="RuntimeAssemblies" ItemName="RuntimeCopyLocalItems" />
  <Output TaskParameter="RuntimeTargets" ItemName="RuntimeTargetsCopyLocalItems" />
  <Output TaskParameter="CompileTimeAssemblies" ItemName="ResolvedCompileFileDefinitions" />
  <Output TaskParameter="TransitiveProjectReferences" ItemName="_TransitiveProjectReferences" />
  <Output TaskParameter="PackageFolders" ItemName="AssetsFilePackageFolder" />
  <Output TaskParameter="PackageDependencies" ItemName="PackageDependencies" />
  <Output TaskParameter="PackageDependenciesDesignTime" ItemName="_PackageDependenciesDesignTime" />
</ResolvePackageAssets>
```

Вызывая эту таску можно получить:
- [[Roslyn analyzers]], например:
	- Microsoft.CodeAnalysis.Analyzers.dll
	- xunit.analyzers.dll
- DebugSymbolsFiles - пути к сгенерированным pdb [[Dotnet Debug symbols|символам]]:
	- Microsoft.Diagnostics.Runtime.pdb
	- Mono.Cecil.pdb
- RuntimeAssemblies -  пути к dll, которые получены из нюгет пакетов:
	- CommandLine.dll
	- Microsoft.Extensions.Options.dll
- RuntimeTargets - platform specific files:
	- gee.external.capstone\\2.3.0\\runtimes\\linux-arm\\native\\libcapstone.so
	- gee.external.capstone\\2.3.0\\runtimes\\linux-arm64\\native\\libcapstone.so