+++
title = 'API Changes Should Be Backward Compatible Until All Clients Use the Updated Version'
date = 2024-09-13T19:24:15+03:00
draft = true
showtoc = true
tocopen = false
tags = ['api-design', 'breaking-changes', 'backward-compatibility', 'consumer-driven-testing','pact']
+++

Whether you are in the middle of the development, or in the maintenance of a project involving several services, you should never introduce a "breaking" change in the way that systems interact. Instead, you should keep all clients working by providing an API with changes that are backward compatible, marking old usage as deprecated, so both 'old' and 'new' clients can use the API without issues. Once all the clients use only the new API, then and only then you can remove the deprecated parts of the API.

## Who are the clients that should not be broken?

The clients you need to consider include all active branches of the repositories of the project services: released branches, main and develop branches. These are permanent branches that, unless updated, they should always be in a fully working, "potentially releasable" condition. Client code in other branches (usually feature branches) could be considered as work in progress and "could" be broken (unless they have been already verified).

## Non-breaking changes

A consumer makes an API request at some endpoint and expects some response.

Not all changes in an API are breaking. In general, most additions are not breaking. Examples of non-breaking changes include:

- Addition of extra fields in existing responses. "Old" clients will simply ignore these "extra" fields.
- Addition of new endpoints. "Old clients" do not call these anyway.

Concerning the API request parameters and bodies, you could make some fields optional. This will not break the client request. Be careful however, not to add required fields in the request parameters or request bodies of the endpoints, since "old" clients know nothing of these yet!

So to recap, you can expand the existing responses, and reduce the required fields in the requests, without introducing any breaking changes.

## Breaking changes

Possible breaking changes are:

- Removal of an expected field from the response. Even a renaming of the field is actually a removal and an addition at the same time. Additions are not breaking, but the removal is.
- Changing the type of an expected field. The name of the field is the same, but the Provider returns a different type that the one expected by the client.
- Removing an endpoint. Even the change of the HTTP method is actually a removal and an addition of another endpoint.
- Change of the query parameters, headers and/or request body.

## Keep it backward compatible

If you are planning to change the response of an endpoint, keep all the fields of the "old" response exactly as they were (name and type) and add new fields that provide the changed information, either the new names or the new types or both.

Let's say for example, that we wish to change the name of a field `user` to `username` in the response:

```json
{
  "user": "dimitris"
}
```

A backward compatible change would be:

```json
{
  "user": "dimitris",
  "username": "dimitris"
}
```

In this example, adding the 'username' field ensures backward compatibility while preparing for the eventual removal of the 'user' field. This response will satisfy both new and old clients, till all old clients start using the renamed field. We could then remove the deprecated `user` field.

Another example would be to change the type of a field `age`, from integer to double:

```json
{
  "age": 35
}
```

A backward compatible change would be:

```json
{
  "age": 35,
  "ageDouble": 35.5
}
```

It is not a perfect solution, especially the name, but it does the job of satisfying both old and new clients. Once all old clients start using the renamed `ageDouble` field, you could change the response to:

```json
{
  "age": 35.5,
  "ageDouble": 35.5
}
```

and mark `ageDouble` as deprecated, till all clients use the `age` field as a double number.

If you are planning to remove or change an endpoint, instead keep the "old" endpoint as it is and add a new endpoint that will provide the new or updated functionality.

## Deprecate 'old' usages

When updating an API, it’s crucial to mark outdated response fields, endpoints, or parameters as deprecated.

Deprecation allows both the old and new versions to coexist temporarily, ensuring that no consumers are abruptly affected. Once all consumers have migrated to the updated API and no longer rely on the deprecated parts, these elements can be safely removed, minimising the risk of breaking changes.

## Remove deprecated fields and endpoints once unused

After deprecating fields, parameters, or endpoints and allowing sufficient time for clients to migrate, it's important to regularly monitor usage to ensure that no clients are still relying on these outdated elements. Once you confirm that no clients are using the deprecated fields or endpoints you can safely proceed with their removal.

Removing these deprecated elements helps maintain a clean and efficient API, reducing clutter and potential confusion for future development. It also minimises maintenance overhead and ensures that the API remains streamlined, secure, and easier to manage over time. However, ensure that all stakeholders are informed of the removal to avoid any unexpected issues.

## Tools

Maintaining backward compatibility in APIs is crucial to ensuring that changes do not disrupt existing clients. Several practices and strategies can help automate the testing and validation of API changes, providing confidence that no clients will break when updates are deployed.

One effective approach is to use **consumer-driven contract testing**, where consumers (clients) define a contract specifying their expected interactions with the API provider. The provider then verifies these contracts against its implementation to ensure that any updates to the API remain compatible with all clients. This method ensures that changes are safe and will not lead to breaking changes, as the provider must satisfy the expectations of all consumers before any deployment. An example of a tool using consumer-driven contract testing is [Pact](https://docs.pact.io/).

Integrating contract testing into your **CI/CD pipeline** can catch potential breaking changes early in the development process. Every time a new version of the API is built or deployed, the contracts from all relevant branches (such as released, main, and development branches) are fetched and verified against the provider's API. This continuous verification process helps ensure stability and compatibility across different versions of the API.

Additionally, using **API specifications** like OpenAPI can help standardise the API design and documentation process. By defining API endpoints, request/response schemas, teams can ensure consistent communication and alignment about the API's expected behaviour. This reduces the chances of introducing unintended breaking changes and allows for automated regression testing.

## Conclusion

Maintaining backward compatibility in APIs is essential to ensure uninterrupted functionality for all clients, both old and new. By carefully managing changes—such as adding new fields, endpoints, or making fields optional—you can expand the API without causing breaks. However, avoid removing or renaming fields, changing data types, or modifying endpoints without proper planning. Deprecating old usages allows for a smooth transition, and once all clients have migrated to the updated API, deprecated elements can be safely removed. Adopting these practices will lead to more stable and reliable API management, minimising disruptions and maintaining seamless integration across services.
