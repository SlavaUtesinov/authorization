# GraphQL Authorization

[![Build Status](https://ci.appveyor.com/api/projects/status/github/graphql-dotnet/authorization?branch=master&svg=true)](https://ci.appveyor.com/project/graphql-dotnet-ci/authorization)
[![NuGet](https://img.shields.io/nuget/v/GraphQL.Authorization.svg)](https://www.nuget.org/packages/GraphQL.Authorization/)
[![Join the chat at https://gitter.im/graphql-dotnet/graphql-dotnet](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/graphql-dotnet/graphql-dotnet?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

A toolset for authorizing access to graph types for [GraphQL .NET](https://github.com/graphql-dotnet/graphql-dotnet).

# Usage

* Register the authorization classes in your DI container (`IAuthorizationEvaluator`, `AuthorizationSettings`, and the `AuthorizationValidationRule`).
* Provide a `UserContext` class that implements `IProvideClaimsPrincipal`.
* Add policies to the `AuthorizationSettings`.
* Apply a policy to a GraphType or Field (which implement `IProvideMetadata`) using `AuthorizeWith(string policy)`.
* Make sure the `AuthorizationValidationRule` is registered with your Schema (depending on your server implementation, you may only need to register it in your DI container)
* The `AuthorizationValidationRule` will run and verify the policies based on the registered policies.
* You can write your own `IAuthorizationRequirement`.
* Use `GraphQLAuthorize` attribute if using Schema First syntax.

# Examples

```csharp
namespace BasicSample
{
    using System;
    using System.Collections.Generic;
    using System.Linq;
    using System.Security.Claims;
    using System.Threading.Tasks;
    using Microsoft.Extensions.DependencyInjection;
    using GraphQL;
    using GraphQL.Types;
    using GraphQL.Validation;
    using GraphQL.SystemTextJson;

    using GraphQL.Authorization;

    class Program
    {
        static async Task Main(string[] args)
        {
            var services = new ServiceCollection();
            services.AddSingleton<IAuthorizationEvaluator, AuthorizationEvaluator>();
            services.AddTransient<IValidationRule, AuthorizationValidationRule>();
            services.AddTransient(s =>
            {
                var authSettings = new AuthorizationSettings();
                authSettings.AddPolicy("AdminPolicy", p => p.RequireClaim("role", "Admin"));
                return authSettings;
            });

            var serviceProvider = services.BuildServiceProvider();

            var definitions = @"
                type User {
                    id: ID
                    name: String
                }

                type Query {
                    viewer: User
                    users: [User]
                }
            ";
            var schema = Schema.For(
                definitions,
                _ =>
                {
                    _.Types.Include<Query>();
                });

            // remove claims to see the failure
            var authorizedUser = new ClaimsPrincipal(new ClaimsIdentity(new[] { new Claim("role", "Admin") }));

            var json = await schema.ExecuteAsync(_ =>
            {
                _.Query = "{ viewer { id name } }";
                _.ValidationRules = serviceProvider.GetServices<IValidationRule>().Concat(DocumentValidator.CoreRules);
                _.RequestServices = serviceProvider;
                _.UserContext = new GraphQLUserContext { User = authorizedUser };
            });

            Console.WriteLine(json);
        }
    }

    public class GraphQLUserContext : Dictionary<string, object>, IProvideClaimsPrincipal
    {
        public ClaimsPrincipal User { get; set; }
    }

    public class Query
    {
        [GraphQLAuthorize(Policy = "AdminPolicy")]
        public User Viewer()
        {
            return new User { Id = Guid.NewGuid().ToString(), Name = "Quinn" };
        }

        public List<User> Users()
        {
            return new List<User> { new User { Id = Guid.NewGuid().ToString(), Name = "Quinn" } };
        }
    }

    public class User
    {
        public string Id { get; set; }
        public string Name { get; set; }
    }
}
```

GraphType first syntax - use `AuthorizeWith`.

```csharp
public class MyType : ObjectGraphType
{
    public MyType()
    {
        this.AuthorizeWith("AdminPolicy");
        Field<StringGraphType>("name").AuthorizeWith("SomePolicy");
    }
}
```

Schema first syntax - use `GraphQLAuthorize` attribute.

```csharp
[GraphQLAuthorize(Policy = "MyPolicy")]
public class MutationType
{
    [GraphQLAuthorize(Policy = "AnotherPolicy")]
    public async Task<string> CreateSomething(MyInput input)
    {
        return Guid.NewGuid().ToString();
    }
}
```

# Known Issues

* It is currently not possible to add a policy to Input objects using Schema first approach.
