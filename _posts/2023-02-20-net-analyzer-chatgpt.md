--- 

layout: post 

title: "Creating a .NET Analyzer with ChatGPT" 

--- 

## Introduction 

[ChatGPT](https://openai.com/blog/chatgpt/) is a natural language processing tool developed by OpenAI on top of their language models. ChatGPT was built and designed to mimic a conversation you might have with another human - input something and get a response. There is a huge amount that ChatGPT can do: answer questions, compose songs and even write code. 

I won't go into the details of how ChatGPT works, mainly because I don't understand them but there are many, many, papers and posts about the tool if you're interested. 

## Getting to know ChatGPT 

Once I had access to the tool, my first inclination was to ask ChatGPT for answers to simple questions: 

- What is the largest 32-bit integer in C#? 

- > The largest 32-bit integer in C# is 2,147,483,647 

- How far is a parsec? 

- > A parsec is equal to 3.26 light-years or approximately 19 trillion miles 

These questions were easy to verify and were returned very quickly; ChatGPT even added some helpful information on other data types in C# and their corresponding max values. 

Naturally, this progressed into me asking ChatGPT to produce poems, songs and stories about various subjects. All of which showed an insight into the subject, often with extra information often associated with that subject. For example, asking ChatGPT to write a haiku about my hometown produced a poem that included local landmarks. 

With that, an entire day was wasted on asking benign questions to a chatbot. 

## .NET Analyzers 

### What are Analyzers 

Onto the coding. ChatGPT is able to interpret my requests for the length of an answer. For example, I can request an entire essay on a single subject or a brief summary: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-3.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Microsoft also has a very good explanation of these, if you're interested [here](https://learn.microsoft.com/en-us/dotnet/framework/code-analyzers). 

That is as good an explanation as I'd be able to come up with. Essentially, .NET Analyzers let you write custom checks for you code with possible fixes so that developers can follow guidelines and best practices your team have in place. You will have seen them in Visual Studio before, highlighted by yellow squiggles: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-1.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Not only can you write rules as above, you can also create accompanying code fixes to apply to those rules. For example, you might create a rule that checks opening braces are on new lines and a code fix provider to move that opening brace onto a new line: 

![Example Analyzer 2](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-2.gif){:style="display:block; margin-left:auto; margin-right:auto"} 

### How Could an Analyzer Help 

Within the team, we maintain a [Styleguide](https://pages.ghe.service.group/risk-data-systems/styleguide/#/standards-dotnet) which details best practices when writing .NET and C# code. This helps us create a consistent code style across our applications and means that developers should recognise that style when picking up work on an app they've not used before. 

There is an issue here though: does anyone have time to read and commit to memory the entire styleguide before making a change on an application? For me, the answer is a definite no. 

On our latest Dev Meeting, we discussed these Analyzers and how it might be a useful tool to accompany the style guide - no need to remember the rules when Visual Studio will point them out for you and recommend a fix! 

### Asking ChatGPT to Create a New Analyzer 

We know that ChatGPT can figure out what an Analyzer is, how about creating one? 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-4.png){:style="display:block; margin-left:auto; margin-right:auto"} 

It's a slightly contrived example, I'm not sure anyone would ever want to start method names with an underscore, but it demonstrates the idea. ChatGPT took longer to produce this but still returned the entire response in just seconds. As you can see from the output, not only did it write code for us, it gave meaningful descriptions of each step. The full answer below: 

> Sure, here's an example of a .NET Analyzer and code fix provider that enforces a naming convention rule that requires method names to start with an underscore. 

> First, we'll define the Analyzer and its associated diagnostic: 

```csharp 

[DiagnosticAnalyzer(LanguageNames.CSharp)] 

public class MethodNameAnalyzer : DiagnosticAnalyzer 

{ 

public const string DiagnosticId = "MethodNameAnalyzer"; 

private const string Title = "Method names must start with an underscore"; 

private const string MessageFormat = "Method '{0}' does not start with an underscore"; 

private const string Category = "Naming"; 

private static readonly DiagnosticDescriptor Rule = new DiagnosticDescriptor(DiagnosticId, Title, MessageFormat, Category, DiagnosticSeverity.Warning, isEnabledByDefault: true); 

public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => ImmutableArray.Create(Rule); 

public override void Initialize(AnalysisContext context) 

{ 

context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None); 

context.RegisterSymbolAction(AnalyzeSymbol, SymbolKind.Method); 

} 

private static void AnalyzeSymbol(SymbolAnalysisContext context) 

{ 

var methodSymbol = (IMethodSymbol)context.Symbol; 

if (!methodSymbol.Name.StartsWith("_")) 

{ 

var diagnostic = Diagnostic.Create(Rule, methodSymbol.Locations[0], methodSymbol.Name); 

context.ReportDiagnostic(diagnostic); 

} 

} 

} 

``` 

> This analyzer examines each method in a C# code file and reports a warning diagnostic if the method name does not start with an underscore. 

> Next, we'll define the code fix provider that automatically adds an underscore to the method name: 

```csharp 

[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(MethodNameCodeFixProvider)), Shared] 

public class MethodNameCodeFixProvider : CodeFixProvider 

{ 

private const string title = "Add underscore to method name"; 

public sealed override ImmutableArray<string> FixableDiagnosticIds => ImmutableArray.Create(MethodNameAnalyzer.DiagnosticId); 

public sealed override FixAllProvider GetFixAllProvider() => WellKnownFixAllProviders.BatchFixer; 

public sealed override async Task RegisterCodeFixesAsync(CodeFixContext context) 

{ 

foreach (var diagnostic in context.Diagnostics) 

{ 

var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken).ConfigureAwait(false); 

var node = root.FindNode(diagnostic.Location.SourceSpan); 

context.RegisterCodeFix( 

CodeAction.Create( 

title: title, 

createChangedDocument: c => AddUnderscoreToMethodName(context.Document, node, c), 

equivalenceKey: title), 

diagnostic); 

} 

} 

private async Task<Document> AddUnderscoreToMethodName(Document document, SyntaxNode node, CancellationToken cancellationToken) 

{ 

var identifier = (IdentifierNameSyntax)node; 

var newIdentifier = SyntaxFactory.Identifier("_" + identifier.Identifier.Text); 

var newNode = identifier.WithIdentifier(newIdentifier); 

var root = await document.GetSyntaxRootAsync(cancellationToken).ConfigureAwait(false); 

var newRoot = root.ReplaceNode(identifier, newNode); 

return document.WithSyntaxRoot(newRoot); 

} 

} 

``` 

So with very little effort, ChatGPT has generated for us a new Analyzer check and an accompanying code fix. I won't go into the code in detail but will provide some highlights: 

1. Naming 

- The naming ChatGPT used appears to be good: 

- The title is Method names must start with an underscore which it has inferred from my question 

- The title of the code fix provider is Add underscore to method name which accurately reflects what the method is doing 

1. Description 

- The description of each file adequately describes what is being done in each method 

1. Quality 

- I can't speak too much on the quality of this code as I've not written many Analyzers but I can confirm it all built right away once I had added it 

### Using the Analyzer 

Now for the fun part, using the Analyzer that ChatGPT created for us. 

The first step here is to ensure you have the right tooling installed. For Analyzers, you are going to need to make sure that the Visual Studio Extension Development package is installed. From there, it was as simple as creating a new Code Analyzer project: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-5.png){:style="display:block; margin-left:auto; margin-right:auto"} 

Upon creation, the new Visual Studio project creates a few projects, notably: 

- AnalyzerCodeFix 

- A project containing any Analyzers we create 

- A dummy analyzer is created to give an idea of how to create one 

- AnalyzerCodeFix.CodeFixes 

- A project containing any Code Fix Providers we create 

- These fix the issues that our new Analyzer highlights 

- Again, a dummy code fix provider is created to give an idea of how to create one 

- AnalyzerCodeFix.Vsix 

- The extension project 

To start, we'll run the newly created project with the dummy check and code fix. The dummy check has an analyzer for method names that are lower case and a fix provider that makes them upper case (I think they were thinking along the same lines as I was). We'll set the VSIX project as the startup project and start debugging. 

As we're debugging an extension project, the debugger will open a new instance of Visual Studio for us to debug. We can open a project as we normally would and see our dummy analyzer in action: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-7.png){:style="display:block; margin-left:auto; margin-right:auto"} 

As you can see, both parts of the analyzer are working together here: it's noticed that the class name is not all uppercase and then provided us with a potential fix (as well as that nice preview of the change). So far, so good! 

Let's now create a new analyzer and code fix provider using the code that ChatGPT gave us: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-6.png){:style="display:block; margin-left:auto; margin-right:auto"}{:style="display:block; margin-left:auto; margin-right:auto"} 

Thankfully, everything compiles just fine with the ChatGPT code - no issues in sight. Let's try debugging the project again and see if our underscore method name analyzer works: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-8.png){:style="display:block; margin-left:auto; margin-right:auto"} 

At this point, it's highlighting our test method with a yellow squiggle and shows the correct warning/analyzer message as well as showing the correct name for the fix. However, when clicking the icon for a list of suggestions, we see an exception is thrown and the analyzer gets disabled: 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-9.png){:style="display:block; margin-left:auto; margin-right:auto"} 

System.InvalidCastException : Unable to cast object of type 'Microsoft.CodeAnalysis.CSharp.Syntax.MethodDeclarationSyntax' to type 'Microsoft.CodeAnalysis.CSharp.Syntax.IdentifierNameSyntax'. 

The error seems quite clear, so we'll take a closer look at the code that ChatGPT generated: 

```csharp 

private async Task<Document> AddUnderscoreToMethodName(Document document, SyntaxNode node, CancellationToken cancellationToken) 

{ 

var identifier = (IdentifierNameSyntax)node; 

var newIdentifier = SyntaxFactory.Identifier("_" + identifier.Identifier.Text); 

var newNode = identifier.WithIdentifier(newIdentifier); 

var root = await document.GetSyntaxRootAsync(cancellationToken).ConfigureAwait(false); 

var newRoot = root.ReplaceNode(identifier, newNode); 

return document.WithSyntaxRoot(newRoot); 

} 

``` 

As you can see on the first line of this method, we're trying to cast the SyntaxNode parameter to a IdentifierNameSyntax. Looking back at our stack trace from the error, we can see the object being passed in is actually of type MethodDeclarationSyntax. Both of these types inherit from the SyntaxNode type but ChatGPT has chosen to cast to the latter, which is interesting as the rest of the code correctly deals with the method name rather than any identifier. 

If we cast to the type we need, we should get a better result: 

```csharp 

private async Task<Document> AddUnderscoreToMethodName(Document document, SyntaxNode node, CancellationToken cancellationToken) 

{ 

var identifier = (MethodDeclarationSyntax)node; 

var newIdentifier = SyntaxFactory.Identifier("_" + identifier.Identifier.Text); 

var newNode = identifier.WithIdentifier(newIdentifier); 

var root = await document.GetSyntaxRootAsync(cancellationToken).ConfigureAwait(false); 

var newRoot = root.ReplaceNode(identifier, newNode); 

return document.WithSyntaxRoot(newRoot); 

} 

``` 

![Example Analyzer](https://pages.ghe.service.group/James-Brett/blog/assets/images/analyzers-10.gif){:style="display:block; margin-left:auto; margin-right:auto"} 

Much better. Our 'code issue' is highlighted and we're given a code fix which works, too. 

## Conclusion 

From a very simple sentence, ChatGPT has effortlessly created a .NET Analyzer with a code fix. However, although the code compiled fine, we did hit a consistent error due to an incorrect type casting. 

Although the fix was fairly trivial to a human who can read the stack trace, there is quite a valuable lesson here. As soon as I had worked out the error in the code, I was reminded of a decision [StackOverflow had to make](https://meta.stackoverflow.com/questions/421831/temporary-policy-chatgpt-is-banned) a few months after ChatGPT's initial release: 

> Use of ChatGPT1 generated text for content on Stack Overflow is temporarily banned. Overall, because the average rate of getting correct answers from ChatGPT is too low, the posting of answers created by ChatGPT is substantially harmful to the site and to users who are asking and looking for correct answers. 

My initial reaction to the bug in the code was surprise - how did ChatGPT not know this error would occur? It told me what each method was doing in detail and gave me code that compiled right away, how could it not work? I believe this is why ChatGPT was banned from StackOverflow initially: the response is well formed, polite and gives additional information relevant to the question but it was simply incorrect (in this case, in a very minor way). 

There are a few lessons I might take away from this task; the first is that ChatGPT is extremely impressive. Colleges are already [writing software to detect AI-written essays](https://www.newscientist.com/article/2355035-chatgpt-detector-could-help-spot-cheaters-using-ai-to-write-essays/) as they were becoming prevalent and it turns out that ChatGPT is pretty good at writing [malware](https://www.zdnet.com/article/people-are-already-trying-to-get-chatgpt-to-write-malware/) and [phishing emails](https://www.cnet.com/tech/services-and-software/its-scary-easy-to-use-chatgpt-to-write-phishing-emails/) just to name a few. 

On the other hand, though, it might be a lesson about how we use these AI-assisted tools, at least for now. ChatGPT can point you in the right direction (it can almost piggyback you right to the door) but sometimes it's best to let it guide you (and force you to read Microsoft Documentation to find out where it went wrong). 
