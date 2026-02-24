# WebACL — Web Access Control for Solid

Generate `.acl` files for Solid pods using the WAC (Web Access Control) specification.

## What This Skill Does

Creates valid WebACL authorization resources (`.acl` files) in Turtle or JSON-LD format. These files control who can read, write, append to, or manage access for resources on a Solid pod.

## Quick Reference

An `.acl` file protects a resource. Each authorization needs four things:

| Component | Predicate | Example |
|-----------|-----------|---------|
| **Type** | `a acl:Authorization` | Always required |
| **Who** | `acl:agent`, `acl:agentClass`, `acl:agentGroup` | A WebID, public, or group |
| **What** | `acl:mode` | `acl:Read`, `acl:Write`, `acl:Append`, `acl:Control` |
| **Which** | `acl:accessTo`, `acl:default` | The target resource or container |

## JSON-LD Context

Use this context for all WebACL JSON-LD documents:

```json
{
  "@context": {
    "acl": "http://www.w3.org/ns/auth/acl#",
    "foaf": "http://xmlns.com/foaf/0.1/",
    "vcard": "http://www.w3.org/2006/vcard/ns#",

    "Authorization": "acl:Authorization",
    "Read": "acl:Read",
    "Write": "acl:Write",
    "Append": "acl:Append",
    "Control": "acl:Control",

    "accessTo": { "@id": "acl:accessTo", "@type": "@id" },
    "default": { "@id": "acl:default", "@type": "@id" },
    "agent": { "@id": "acl:agent", "@type": "@id" },
    "agentClass": { "@id": "acl:agentClass", "@type": "@id" },
    "agentGroup": { "@id": "acl:agentGroup", "@type": "@id" },
    "mode": { "@id": "acl:mode", "@type": "@id", "@container": "@set" },
    "origin": { "@id": "acl:origin", "@type": "@id" },

    "AuthenticatedAgent": "acl:AuthenticatedAgent",
    "Agent": "foaf:Agent",
    "Group": "vcard:Group",
    "hasMember": { "@id": "vcard:hasMember", "@type": "@id" }
  }
}
```

## Access Modes

| Mode | HTTP Methods | Description |
|------|-------------|-------------|
| `acl:Read` | GET, HEAD | View resource contents |
| `acl:Write` | PUT, POST, PATCH, DELETE | Full write access (implies Append) |
| `acl:Append` | POST, INSERT-only PATCH | Add data only, cannot remove |
| `acl:Control` | GET/PUT on the `.acl` file | Manage the ACL itself |

## Access Subjects

| Predicate | Value | Meaning |
|-----------|-------|---------|
| `acl:agent` | WebID URI | One specific person |
| `acl:agentGroup` | Group URI | Members of a `vcard:Group` |
| `acl:agentClass` | `foaf:Agent` | Everyone (public, no auth needed) |
| `acl:agentClass` | `acl:AuthenticatedAgent` | Anyone with a WebID |
| `acl:origin` | Origin URI | Restrict to a web app origin |

## Scope Predicates

| Predicate | Applies to |
|-----------|-----------|
| `acl:accessTo` | The specific named resource only |
| `acl:default` | Child resources that lack their own `.acl` |

**Important**: These are independent. `acl:default` does NOT apply to the container itself. Use both to cover a container and its children.

## File Naming Convention

| Resource | ACL File |
|----------|----------|
| `https://pod/docs/file1` | `https://pod/docs/file1.acl` |
| `https://pod/docs/` (container) | `https://pod/docs/.acl` |
| `https://pod/` (root) | `https://pod/.acl` |

## Inheritance Rules

1. If a resource has its own `.acl` file → use it exclusively (no inheritance)
2. If not → walk up the container hierarchy looking for `acl:default`
3. The root container `.acl` always exists (server requirement)

## Directories (Containers)

In Solid, directories are called **containers**. Their URIs always end with `/`.

| Directory | ACL File |
|-----------|----------|
| `https://pod/docs/` | `https://pod/docs/.acl` |
| `https://pod/` (root) | `https://pod/.acl` |

**How access modes work on directories:**

| Mode | Effect on Directory |
|------|-------------------|
| `acl:Read` | List the directory contents |
| `acl:Write` | Create and delete resources inside the directory |
| `acl:Append` | Create resources inside (but not delete) |
| `acl:Control` | Manage the directory's `.acl` file |

**Scope predicates for directories:**

| Predicate | Applies to |
|-----------|-----------|
| `acl:accessTo <./>;` | The directory itself only |
| `acl:default <./>;` | Files and subdirectories inside (that lack their own `.acl`) |
| Both together | The directory AND all its contents |

**Important**: `acl:default` does NOT apply to the directory itself. These are independent — you need both predicates to cover a directory and everything inside it.

## Templates

### Template 1: Owner-Only Resource

**Use when**: A private resource only the owner should access.

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read, acl:Write, acl:Control.
```

JSON-LD:
```json
{
  "@context": {
    "acl": "http://www.w3.org/ns/auth/acl#"
  },
  "@id": "#owner",
  "@type": "acl:Authorization",
  "acl:agent": { "@id": "{{OWNER_WEBID}}" },
  "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
  "acl:mode": [
    { "@id": "acl:Read" },
    { "@id": "acl:Write" },
    { "@id": "acl:Control" }
  ]
}
```

### Template 2: Public Read, Owner Write

**Use when**: A publicly visible resource that only the owner can modify (profiles, public pages).

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read, acl:Write, acl:Control.

<#public>
    a acl:Authorization;
    acl:agentClass foaf:Agent;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read.
```

JSON-LD:
```json
[
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#owner",
    "@type": "acl:Authorization",
    "acl:agent": { "@id": "{{OWNER_WEBID}}" },
    "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" },
      { "@id": "acl:Control" }
    ]
  },
  {
    "@context": {
      "acl": "http://www.w3.org/ns/auth/acl#",
      "foaf": "http://xmlns.com/foaf/0.1/"
    },
    "@id": "#public",
    "@type": "acl:Authorization",
    "acl:agentClass": { "@id": "foaf:Agent" },
    "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
    "acl:mode": [{ "@id": "acl:Read" }]
  }
]
```

### Template 3: Root Container ACL

**Use when**: Setting up the root ACL for a pod. This is the foundation — owner gets full access, public gets read access to the root, and defaults cascade to all children.

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <./>;
    acl:default <./>;
    acl:mode acl:Read, acl:Write, acl:Control.

<#public>
    a acl:Authorization;
    acl:agentClass foaf:Agent;
    acl:accessTo <./>;
    acl:mode acl:Read.
```

JSON-LD:
```json
[
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#owner",
    "@type": "acl:Authorization",
    "acl:agent": { "@id": "{{OWNER_WEBID}}" },
    "acl:accessTo": { "@id": "./" },
    "acl:default": { "@id": "./" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" },
      { "@id": "acl:Control" }
    ]
  },
  {
    "@context": {
      "acl": "http://www.w3.org/ns/auth/acl#",
      "foaf": "http://xmlns.com/foaf/0.1/"
    },
    "@id": "#public",
    "@type": "acl:Authorization",
    "acl:agentClass": { "@id": "foaf:Agent" },
    "acl:accessTo": { "@id": "./" },
    "acl:mode": [{ "@id": "acl:Read" }]
  }
]
```

### Template 4: Append-Only Inbox

**Use when**: An inbox or submission endpoint where anyone authenticated can add items but only the owner can read/delete.

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <./inbox/>;
    acl:default <./inbox/>;
    acl:mode acl:Read, acl:Write, acl:Control.

<#appendPublic>
    a acl:Authorization;
    acl:agentClass acl:AuthenticatedAgent;
    acl:accessTo <./inbox/>;
    acl:default <./inbox/>;
    acl:mode acl:Append.
```

JSON-LD:
```json
[
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#owner",
    "@type": "acl:Authorization",
    "acl:agent": { "@id": "{{OWNER_WEBID}}" },
    "acl:accessTo": { "@id": "./inbox/" },
    "acl:default": { "@id": "./inbox/" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" },
      { "@id": "acl:Control" }
    ]
  },
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#appendPublic",
    "@type": "acl:Authorization",
    "acl:agentClass": { "@id": "acl:AuthenticatedAgent" },
    "acl:accessTo": { "@id": "./inbox/" },
    "acl:default": { "@id": "./inbox/" },
    "acl:mode": [{ "@id": "acl:Append" }]
  }
]
```

### Template 5: Group Access

**Use when**: A team or group of people should have shared access to a resource.

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read, acl:Write, acl:Control.

<#group>
    a acl:Authorization;
    acl:agentGroup <{{GROUP_URI}}>;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read, acl:Write.
```

JSON-LD:
```json
[
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#owner",
    "@type": "acl:Authorization",
    "acl:agent": { "@id": "{{OWNER_WEBID}}" },
    "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" },
      { "@id": "acl:Control" }
    ]
  },
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#group",
    "@type": "acl:Authorization",
    "acl:agentGroup": { "@id": "{{GROUP_URI}}" },
    "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" }
    ]
  }
]
```

### Template 6: Origin-Restricted App Access

**Use when**: Restricting access to a specific web application (by its origin URL).

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .

<#appAccess>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:origin <{{APP_ORIGIN}}>;
    acl:accessTo <{{RESOURCE_PATH}}>;
    acl:mode acl:Read, acl:Write.
```

JSON-LD:
```json
{
  "@context": {
    "acl": "http://www.w3.org/ns/auth/acl#"
  },
  "@id": "#appAccess",
  "@type": "acl:Authorization",
  "acl:agent": { "@id": "{{OWNER_WEBID}}" },
  "acl:origin": { "@id": "{{APP_ORIGIN}}" },
  "acl:accessTo": { "@id": "{{RESOURCE_PATH}}" },
  "acl:mode": [
    { "@id": "acl:Read" },
    { "@id": "acl:Write" }
  ]
}
```

### Template 7: Container with Default Inheritance

**Use when**: Setting permissions on a container that should cascade to all children.

Turtle:
```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

<#owner>
    a acl:Authorization;
    acl:agent <{{OWNER_WEBID}}>;
    acl:accessTo <{{CONTAINER_PATH}}>;
    acl:default <{{CONTAINER_PATH}}>;
    acl:mode acl:Read, acl:Write, acl:Control.

<#readDefault>
    a acl:Authorization;
    acl:agentClass foaf:Agent;
    acl:accessTo <{{CONTAINER_PATH}}>;
    acl:default <{{CONTAINER_PATH}}>;
    acl:mode acl:Read.
```

JSON-LD:
```json
[
  {
    "@context": { "acl": "http://www.w3.org/ns/auth/acl#" },
    "@id": "#owner",
    "@type": "acl:Authorization",
    "acl:agent": { "@id": "{{OWNER_WEBID}}" },
    "acl:accessTo": { "@id": "{{CONTAINER_PATH}}" },
    "acl:default": { "@id": "{{CONTAINER_PATH}}" },
    "acl:mode": [
      { "@id": "acl:Read" },
      { "@id": "acl:Write" },
      { "@id": "acl:Control" }
    ]
  },
  {
    "@context": {
      "acl": "http://www.w3.org/ns/auth/acl#",
      "foaf": "http://xmlns.com/foaf/0.1/"
    },
    "@id": "#readDefault",
    "@type": "acl:Authorization",
    "acl:agentClass": { "@id": "foaf:Agent" },
    "acl:accessTo": { "@id": "{{CONTAINER_PATH}}" },
    "acl:default": { "@id": "{{CONTAINER_PATH}}" },
    "acl:mode": [{ "@id": "acl:Read" }]
  }
]
```

## Template Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `{{OWNER_WEBID}}` | The pod owner's WebID URI | `https://alice.pod/profile/card#me` |
| `{{RESOURCE_PATH}}` | Relative path to the protected resource | `./docs/file1` |
| `{{CONTAINER_PATH}}` | Relative path to a container (ends with `/`) | `./public/` |
| `{{GROUP_URI}}` | URI of a `vcard:Group` resource | `https://alice.pod/groups#Team` |
| `{{APP_ORIGIN}}` | Web application origin | `https://myapp.example.com` |

## How to Generate an .acl File

1. **Identify the resource** to protect and its URI
2. **Determine the ACL file path**: append `.acl` to the resource URI (or use `.acl` in the container)
3. **Choose a template** from above that matches the access pattern
4. **Replace template variables** with actual values
5. **Serialize** as Turtle (`text/turtle`) — this is required by the spec. JSON-LD is optional but equivalent
6. **For containers**: decide whether to use `acl:accessTo` (this container only), `acl:default` (children), or both

## Validation Checklist

Every `acl:Authorization` must have all four:
- [ ] `a acl:Authorization` (the type)
- [ ] At least one access subject (`acl:agent`, `acl:agentClass`, or `acl:agentGroup`)
- [ ] At least one access mode (`acl:mode`)
- [ ] At least one access target (`acl:accessTo` or `acl:default`)

## Content Type

ACL resources MUST be served as `text/turtle`. JSON-LD (`application/ld+json`) may be supported via content negotiation but is not guaranteed by all servers.

## Specification

- [Web Access Control (WAC)](https://solid.github.io/web-access-control-spec/) — W3C Solid CG Draft, Version 1.0.0
- [ACL Ontology](https://www.w3.org/ns/auth/acl) — http://www.w3.org/ns/auth/acl#
- [Solid Protocol](https://solid.github.io/specification/protocol) — mandates WAC or ACP support
- [W3C WebAccessControl Wiki](https://www.w3.org/wiki/WebAccessControl) — original proposal by Tim Berners-Lee (2009)

## License

AGPL-3.0
