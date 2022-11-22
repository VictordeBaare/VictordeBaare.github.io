<style> .blue-border {border-left:solid 4px lightblue; padding-left:20px; border-top:dotted 4px darkcyan;} </style> <style> [class$="post"] { border-bottom:dotted 4px darkcyan; } img { max-width: 20%; height: auto; border-radius: 10%; opacity: 0.9; } </style>
Blog posts

---

# Software Composition Analysis

Open source software is incredibly popular, it is safe to say that it changed the software landscape in the last decade.
While this is incredible, it also comes with risks. Code is written by humans and yesterdays code might contain a critical vulnerabilty tomorrow.
Now how are we to mitigate these security vulnerabilties? We can hardly ask every developer to check the code which is used from third parties, especially now that a short release cycle is expected in the development process. 
Luckily there is a solution for this problem, Software Composition Analysis or SCA.

SCA is used to scan your solution for possible security vulnerabilities in your dependencies. Currently there are multiple solutions which you can use to identify and fix security vulnerabilities here are a few of the most popular ones:

- [Mend](https://docs.mend.io/)
- [Snyk](https://docs.snyk.io/products/snyk-open-source)
- [GitGuardian](https://docs.gitguardian.com/internal-repositories-monitoring/home) (GitHub)
- [GitLab](https://docs.gitlab.com/ee/user/application_security/dependency_scanning/)

Mend and Snyk can be integrated in your current DevOps process. Whereas GitLab and GitGuardian are full platforms which offer these services.

Now adding SCA is not enough. Integrating it in your development process is crucial. These tools can highlight issues, but the development team needs to resolve them. I would advice to add the scans to your pull request process. Similar to a gated build, add the SCA check to your pipeline. With this you are ahead of some issues, but we aren't there yet. We also need to scan existing application which might not deploy very often anymore. So a nightly build is also crucial to get informed over new security issues in existing packages!

If you combine these two pipelines with a team which is eager to fix these kind of issues then you already tackled a set of security issues and made your software a bit more secure.

---

# Code refactoring made easy with Roslyn Part 1

I have worked on a number on projects and there are certain implementations you keep on doing. For example the mapping from a data object to a business object, or creating builders to help you with testing. These implementations don't consume a lot of time, but at the end of the day all the time they did consume is time you could have spent doing more epic stuff.

In this post we will be creating a code refactoring which can create a mapping based on two objects. The full repository can be found here: [GitHub](https://github.com/VictordeBaare/Capsicum)
Another usefull link to help with using roslyn in your code [Quoter](https://roslynquoter.azurewebsites.net/). You can write the defintion for which you would like the Roslyn syntax. This will help you get an idea which syntax to use for which statements!

Now for starting your first Roslyn code refactoring tool, Visual Studio has some ready to use project templates:
![image](https://user-images.githubusercontent.com/22788227/202998953-25a4eaab-d9ec-42b3-940a-842c600bc2f2.png)
Now Visual studio will create two projects, a dotnet-standard project and a dotnet-framework project. The dotnet-framework project is for the Vsix project which will be used to deploy the code refactoring installer.

The next step is ensuring the Vsix project can de used by Visual Studio 2022 and other versions of Visual Studio. To do this we must adjust the Vsix project installation which resides in the source.extension.vsixmanifest.

Out of the box you'll receive something like this:
```
  <Installation>
    <InstallationTarget Id="Microsoft.VisualStudio.Community" Version="[15.0,)" />
  </Installation>
```

I advise you to adjust it to the following so you can use it for Visual Studio 2022:
```
  <Installation>
	  <InstallationTarget Id="Microsoft.VisualStudio.Community" Version="[15.0,17.0)">
		  <ProductArchitecture>x86</ProductArchitecture>
	  </InstallationTarget>
	  <InstallationTarget Id="Microsoft.VisualStudio.Community" Version="[17.0,18.0)">
		  <ProductArchitecture>amd64</ProductArchitecture>
	  </InstallationTarget>
  </Installation>
```

Now that we finished adjusting the Vsix file lets start with creating a test-project. The reason for using a test project is that it'll make your life a whole lot easier. Code Refactoring or any roslyn tooling often involves starting a Experimental instance of Visual Studio. This is time consuming especially in the initial stages of development. But testing requires a bit more setting up. So lets start with creating a normal test project. 

The next step is installing the necessary nuget packages for running the tests on a Code Refactoring tool.
I prefer using MsTest but other test frameworks are also supported.
The following Nuget package is needed:
```
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp.CodeRefactoring.Testing.MSTest" Version="1.1.0" />
```
In addtion I prefer to also use FluentAssertions and Moq but that is just a personal preference.

The next step is adding a class which will do the heavy lifting for the test, this base class is based on the base class you get when you choose the CodeFix project template. For some reason they included it for code fixes but not for Code Refactoring.

```
    using System.Threading;
    using System.Threading.Tasks;
    using Microsoft.CodeAnalysis.CodeRefactorings;
    using Microsoft.CodeAnalysis.CSharp.Testing;
    using Microsoft.CodeAnalysis.Testing;
    using Microsoft.CodeAnalysis.Testing.Verifiers;

    /// <summary>
    /// This code is automatically added when you create a CodeFix project.
    /// Sadly they didnt implement this for a CodeRefactoring Project.
    /// </summary>
    /// <typeparam name="TCodeRefactoring"></typeparam>
    public static partial class CSharpCodeRefactoringVerifier<TCodeRefactoring>
        where TCodeRefactoring : CodeRefactoringProvider, new()
    {
        public static async Task VerifyRefactoringAsync(string source, string fixedSource)
        {
            await VerifyRefactoringAsync(source, DiagnosticResult.EmptyDiagnosticResults, fixedSource);
        }

        /// <inheritdoc cref="CodeRefactoringVerifier{TCodeRefactoring, TTest, TVerifier}.VerifyRefactoringAsync(string, DiagnosticResult, string)"/>
        public static async Task VerifyRefactoringAsync(string source, DiagnosticResult expected, string fixedSource)
        {
            await VerifyRefactoringAsync(source, new[] { expected }, fixedSource);
        }

        /// <inheritdoc cref="CodeRefactoringVerifier{TCodeRefactoring, TTest, TVerifier}.VerifyRefactoringAsync(string, DiagnosticResult[], string)"/>
        public static async Task VerifyRefactoringAsync(string source, DiagnosticResult[] expected, string fixedSource)
        {
            var test = new Test
            {
                TestCode = source,
                FixedCode = fixedSource,
            };

            test.ExpectedDiagnostics.AddRange(expected);
            await test.RunAsync(CancellationToken.None);
        }

        public class Test : CSharpCodeRefactoringTest<TCodeRefactoring, MSTestVerifier>
        {
            public Test()
            {
                SolutionTransforms.Add((solution, projectId) =>
                {
                    var compilationOptions = solution.GetProject(projectId).CompilationOptions;
                    compilationOptions = compilationOptions.WithSpecificDiagnosticOptions(
                        compilationOptions.SpecificDiagnosticOptions.SetItems(CSharpVerifierHelper.NullableWarnings));
                    solution = solution.WithProjectCompilationOptions(projectId, compilationOptions);

                    return solution;
                });
            }
        }
    }
    
    using System;
    using System.Collections.Immutable;
    using Microsoft.CodeAnalysis;
    using Microsoft.CodeAnalysis.CSharp;

    /// <summary>
    /// This code is automatically added when you create a CodeFix project.
    /// Sadly they didnt implement this for a CodeRefactoring Project.
    /// </summary>
    internal static class CSharpVerifierHelper
    {
        /// <summary>
        /// By default, the compiler reports diagnostics for nullable reference types at
        /// <see cref="DiagnosticSeverity.Warning"/>, and the analyzer test framework defaults to only validating
        /// diagnostics at <see cref="DiagnosticSeverity.Error"/>. This map contains all compiler diagnostic IDs
        /// related to nullability mapped to <see cref="ReportDiagnostic.Error"/>, which is then used to enable all
        /// of these warnings for default validation during analyzer and code fix tests.
        /// </summary>
        internal static ImmutableDictionary<string, ReportDiagnostic> NullableWarnings { get; } = GetNullableWarningsFromCompiler();

        private static ImmutableDictionary<string, ReportDiagnostic> GetNullableWarningsFromCompiler()
        {
            string[] args = { "/warnaserror:nullable" };
            var commandLineArguments = CSharpCommandLineParser.Default.Parse(args, baseDirectory: Environment.CurrentDirectory, sdkDirectory: Environment.CurrentDirectory);
            var nullableWarnings = commandLineArguments.CompilationOptions.SpecificDiagnosticOptions;

            // Workaround for https://github.com/dotnet/roslyn/issues/41610
            nullableWarnings = nullableWarnings
                .SetItem("CS8632", ReportDiagnostic.Error)
                .SetItem("CS8669", ReportDiagnostic.Error);

            return nullableWarnings;
        }
    }
```

Now we have created the class for our testing, the next step is implementing a test for our refactoring code.
Because the input of the test expects example code I prefer to use resources which contain the testing code. It keeps the tests nice and clean, and you can easily adjust the example code. An example of a test is as follows:
```
[TestMethod]
public async Task HappyFlowTestWithOnlyBasicProperties()
{
    await CSharpCodeRefactoringVerifier<AutoMapRefactoring>.VerifyRefactoringAsync(TestCases.PropertyClass_BeforeMapping, TestCases.PropertyClass_AfterMapping);
}
```

In this example I created the TestCases Resource and added a [before mapping code example](https://github.com/VictordeBaare/Capsicum/blob/main/AutoMapCodeRefactoringTests/Resources/PropertyClass_BeforeMapping.txt) and an [after mapping code example](https://github.com/VictordeBaare/Capsicum/blob/main/AutoMapCodeRefactoringTests/Resources/PropertyClass_AfterMapping.txt).
The part to focus on in these examples is the [|Map|] the [| |] notation tells the test where to start for the code refactoring.
```
    public static class Mapper
    {
        public static Target [|Map|](Source source)
        {
            throw new NotImplementedException();
        }
    }
```

In the after mapping code example you can see the result that is expected. With this we finished the necessary resources to easily create a code refactoring tool for Roslyn. The next step would be creating the implementation itself.

---

# Code refactoring made easy with Roslyn Part 2

In the first part of this tutorial we created a solution in which we can easily develop a Roslyn Code Refactoring tool.
The next step is to actually develop the tool itself. For this I decided to create a mapping tool. Nothing fancy you start with defining the method itself and continue from there. For example:
```
    public static class Mapper
    {
        public static Target [|Map|](Source source)
        {
            throw new NotImplementedException();
        }
    }
```
You dont want to write the mapping code. You just want to create the method with the return object and the input parameter.
The mapping itself should be added by Roslyn when you decide to do so. The following mapping output should be something like:

```
public static class Mapper
    {
        public static Target Map(Source source)
        {
            //source.TestChild: UnmappedProp could not be matched
            //sourcechildItem: UnmappedProp could not be matched
            //source: UnmappedProp could not be matched
            return new Target
            {
                TestString = source.TestString,
                TestInt = source.TestInt,
                TestEnum = MapSourceEnum(source.TestEnum),
                TestChild = new TargetChild
                {
                    TestString = source.TestChild.TestString,
                    TestInt = source.TestChild.TestInt
                },
                TestChildren = source.TestChildren.Select(sourcechildItem => new TargetChild
                {
                    TestString = sourcechildItem.TestString,
                    TestInt = sourcechildItem.TestInt
                }).ToList(),
                TestChildrenSimpleType = source.TestChildrenSimpleType,
                TestIntNullable = source.TestIntNullable ?? throw new ArgumentNullException(nameof(source.TestIntNullable))
            };
        }

        public static TargetEnum MapSourceEnum(SourceEnum testenum)
        {
            switch (testenum)
            {
                case SourceEnum.None:
                    return TargetEnum.None;
                case SourceEnum.Test:
                    return TargetEnum.Test;
                default:
                    throw new ArgumentOutOfRangeException(nameof(testenum));
            }
        }
    }
```

Now we will be creating the logic for the mapping itself. 
It starts with implementing the ComputeRefactoringsAsync method, this method receives the CodeRefactoringContext to which you are trying to make adjustments.
To receive the actual SyntaxNode which you selected, you can use the GetSyntaxRootAsync method and from that root you can find the actual node with the FindNode method.

```
public sealed override async Task ComputeRefactoringsAsync(CodeRefactoringContext context)
{
    var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken).ConfigureAwait(false);
    var node = root.FindNode(context.Span);

    ImplementRefactoringCommand(context, node);
}
```

From here on out we can start adding our custom logic. Because we are making an rather dumb mapping method I added the logic to check if its a method with a return type and a single input parameter. If thats the case we can go further to create a mapping code action and register it to Visual Studio!

The mapping itself contains a bit more complex logic. Because we are trying to create a mapper there is the potential for some recursion. For example it is possible that the root object contains a property which itself is a complex object which needs to be mapped. For this reason we create an object to hold these extra statements.
Now for the next step we check what type the returntype and input type is. For example is it an Enum, is it a list or is it a standard complex object. 
For now lets start with the easiest one; the complex object. In the case that it is a complex object we will get all the properties on the source.

```
var sourceProperties = source.GetMembers().Where(x => !x.IsStatic && !x.IsImplicitlyDeclared && x is IPropertySymbol && x.DeclaredAccessibility == Accessibility.Public).Cast<IPropertySymbol>().ToList();
```

In this case its important that the members we are working with are PropertySymbols and Public settable. The next step is creating the mapping, often there are two ways just use the public setters or use an constructor. For now I ignored the combination of these two flavours, to keep the code a bit more readable. 
For now lets go with the flavour of setting the properties directly. The syntax of Roslyn dictates that the first step that we must do is declare the new object syntax. Using the target ITypeSymbol we can call the SyntaxGenerator to create a type expression, and with that create the [ObjectCreationExpression](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntaxfactory.objectcreationexpression?view=roslyn-dotnet-4.3.0#microsoft-codeanalysis-csharp-syntaxfactory-objectcreationexpression(microsoft-codeanalysis-syntaxtoken-microsoft-codeanalysis-csharp-syntax-typesyntax-microsoft-codeanalysis-csharp-syntax-argumentlistsyntax-microsoft-codeanalysis-csharp-syntax-initializerexpressionsyntax)) on the [SyntaxFactory](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntaxfactory?view=roslyn-dotnet-4.3.0). This step can also be done at a later stage but I prefer to do it in the beginning. Now the next step is creating the actual mapping. To do this we get the target member properties the same way we got the source properties. Now we try to match the target and the source properties. 
