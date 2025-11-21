# Maintainable VS Code Extension

### Introduction

Developing VS Code extensions is pretty cool because, compared to other IDEs,
it's extremely easy to come up with code that does something useful.

The VS Code API is kinda flat, it's not exposed as a multi-layered architecture
that we need to master before being able to code (I'm looking at you, IntelliJ).
The JavaScript environment allows developer to iterate fast, to access millions
of useful libraries, and so on.

This is all nice and cool at the very beginning, in the "Hello World" situation
where the extension's contribution are relatively low in quantity and mostly do
not have to cooperate. However, as soon as we start registering multiple commands,
multiple tree providers or custom editors, the situation changes rapidly.
We begin to see new top-level functions spawning everywhere, increasingly deep
parameter passing where the lifetime of those parameters is lost along the way,
global mutable state (because well, JS is single-threaded so who cares, right?),
and much more.

A couple of months after the first successful activation and boom! The entry point
file is now 3000 LOC long with no sign of slowing down in size, because everything's.
so tied up together that we cannot even begin to extract pieces of code without
breaking anything. And I'm not even mentioning testing... Testable components?
I wish!

But let's stop ranting and let's try to reason about whether architectural
patterns that can help us actually exist. My experience with the JVM (mainly Java
and Kotlin) makes me think about Spring, or Micronaut, or Quarkus.
These frameworks are all mainstream at this point, and they all have one aspect
in common: the use of the Dependency Injection pattern, and more specifically
of Java's Contexts and Dependency Injection (CDI) specification.

To put it shortly, it's a way of creating a network of (isolated, in some sense)
components that can interact without having to be wired up manually. It's the
Dependency Injection design pattern, just done automagically. And in addition
to Dependency Injection, CDI also offers an Event Bus to be able to dispatch
messages to the entire network, without dealing with explicit references between
components. Very, very cool stuff.

The natural question we might have now is: can we do the same for VS Code extensions?
And the answer is, _kinda_. We do have Dependency Injection container implementations
in JavaScript, and we do have Message/Event bus implementations in JavaScript,
so it's all a matter of understanding their limitations, and how to apply them
to our situation.

#### The VS Code platform

Before beginning to showcase some examples, it's worth mentioning the even VS Code,
internally, uses Dependency Injection! In a limited form, and sometimes it's more
of a Service Locator pattern, but let's see a piece of code:

```ts
// File: configuration.ts
export const IConfigurationService = createDecorator<IConfigurationService>('configurationService');

export interface IConfigurationService {
  /* interface methods ... */
}

// File: main.ts
private createServices() /* ... */ {
  const services = new ServiceCollection();
  /* ... */

  // Environment
  const environmentMainService = new EnvironmentMainService(/* ... */);
  const instanceEnvironment = this.patchEnvironment(environmentMainService);
  services.set(IEnvironmentMainService, environmentMainService);

  // Logger
  const loggerService = new LoggerMainService(/* ... */);
  services.set(ILoggerMainService, loggerService);

  // Configuration
  const configurationService = new ConfigurationService(/* ... */);
  services.set(IConfigurationService, configurationService);

  /* ... */
}

// File: mcpDiscovery.ts
export class McpDiscovery extends Disposable implements IWorkbenchContribution {
  public static readonly ID = 'workbench.contrib.mcp.discovery';

  constructor(
    @IInstantiationService instantiationService: IInstantiationService,
    @IConfigurationService configurationService: IConfigurationService,
  ) { /*...*/ }
}
```

As you can see a `IConfigurationService` constant has been declared to represent
the `IConfigurationService` interface, and that constant is then used as a decorator
in the `McpDiscovery`'s constructor to mark the parameter as _injectable_.

The internal DI container will take care of constructing a new instance of
`McpDiscovery` as soon as it is requested, automatically resolving and injecting
an instance of `IConfigurationService` and an instance of `IInstantiationService`.
And that's how `McpDiscovery` becomes part of the dependencies graph managed
by the container.

### Our own extension

Now that we know this is a viable approach, let's apply it to a new extension.  
Instead of setting up loggers, registering commands, tree providers, and so on,
inside the `activate` entry point:

```ts
// File: activate.ts
export function activate(context: vscode.ExtensionContext): void {
  // Avoid doing everything here!
}
```

We may think of the extension as an aggregation of "subjects" we want to implement,
each adding and handling its own contributions in isolation. I'll use one of the
extensions I'm developing as an example:

```text
                                    ┌────────────────────────┐  registerCommand()
                                    │                        │
                            ┌──────►│ HistoryFileContributor │  registerCommand()
                            │       │                        │
                            │       └────────────────────────┘  registerCustomEditorProvider()
                            │
                            │
               ┌────────────┴┐      ┌────────────────────────┐
               │             │      │                        │  registerCommand()
activate()────►│  Extension  ├─────►│ FaultReportContributor │
               │             │      │                        │  registerCommand()
               └────────────┬┘      └────────────────────────┘
                            │
                            │
                            │       ┌────────────────────────┐
                            │       │                        │  registerCommand()
                            └──────►│   ZoweCICSContributor  │
                                    │                        │  ResourceExtender.registerAction()
                                    └────────────────────────┘
```

Organizing the code in such a way helps with separation of concerns, and limits
the amount of interactions between unrelated pieces of code.

To proceed further and actually showcase the setup of a DI container, we need
an additional dependency:

```text
npm install @lppedd/di-wise-neo
```

We can now go back to the `activate` function and refactor it to initialize a
container and register the various components into the dependencies graph:

```ts
// File: types.ts
export const IExtensionContext = createType<vscode.ExtensionContext>("vscode.ExtensionContext");
export const ILogOutputChannel = createType<vscode.LogOutputChannel>("vscode.LogOutputChannel");
export const IContributor = createType<Contributor>("Contributor");

// File: activate.ts
export function activate(context: vscode.ExtensionContext): void {
  const container = initContainer(context);
  const extension = container.resolve(Extension);
  extension.init();
}

function initContainer(context: vscode.ExtensionContext): Container {
  const container = createContainer({ defaultScope: "Container" });
  context.subscriptions.push(container);

  const logChannel = vscode.window.createOutputChannel("Extension", { log: true });
  context.subscriptions.push(logChannel);

  // Register the core platform objects
  container.register(ILogOutputChannel, { useValue: logChannel });
  container.register(IExtensionContext, { useValue: context });

  // Register the business logic services
  container.register(FaultReportService);
  container.register(HistoryFileService);
  
  // Register the extension points contributors
  container.register(IContributor, { useClass: HistoryFileContributor });
  container.register(IContributor, { useClass: FaultReportContributor });
  container.register(IContributor, { useClass: ZoweCICSContributor });

  // Register the extension entry point
  container.register(Extension);
  return container;
}
```

But... what's the code for `Extension`? Or for `HistoryFileContributor`?
I'm not going to show the entirety of the code, but the following snippet should
be sufficient to understand how every piece fits into the overall architecture,
especially if you're already familiar with the JVM ecosystem.

```ts
// File: historyFileContributor.ts
export class HistoryFileContributor implements Contributor, vscode.Disposable {
  private readonly editorProvider: HistoryFileEditorProvider;

  constructor(
    @Inject(ILogOutputChannel) readonly log: vscode.LogOutputChannel,
    @Inject(IExtensionContext) readonly context: vscode.ExtensionContext,
    @Inject(HistoryFileService) readonly service: HistoryFileService,
  ) {
    this.editorProvider = new HistoryFileEditorProvider(context);
  }

  contribute(): void {
    vscode.commands.registerCommand("extension.historyFile.open", (fileName) => {
      this.log.trace("called extension.historyFile.open");
      return this.service.openHistoryFile(fileName);
    });

    vscode.window.registerCustomEditorProvider("HistoryFile", this.editorProvider, {
      webviewOptions: {
        retainContextWhenHidden: true,
      },
    }),
  }
  
  dispose(): void {
    this.editorProvider.dispose();
  }
}

// File: extension.ts
export class Extension {
  constructor(@InjectAll(IContributor) readonly contributors: Contributor[]) {}

  init(): void {
    // Initialize all contributions
    for (const contributor of this.contributors) {
      contributor.contribute();
    }
  }
}
```

Work in progress...
