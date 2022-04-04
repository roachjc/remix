---
title: Common
order: 1
---

# Server / Runtimes / Common

## HTTP Helpers

### `json`

This is a shortcut for creating `application/json` responses. It assumes you are using `utf-8` encoding.

```ts lines=[2,6]
import type { LoaderFunction } from "remix";
import { json } from "remix";

export const loader: LoaderFunction = async () => {
  // So you can write this:
  return json({ any: "thing" });

  // Instead of this:
  return new Response(JSON.stringify({ any: "thing" }), {
    headers: {
      "Content-Type": "application/json; charset=utf-8",
    },
  });
};
```

You can also pass a status code and headers:

```ts lines=[4-9]
export const loader: LoaderFunction = async () => {
  return json(
    { not: "coffee" },
    {
      status: 418,
      headers: {
        "Cache-Control": "no-store",
      },
    }
  );
};
```

### `redirect`

This is shortcut for sending 30x responses.

```ts lines=[2,8]
import type { ActionFunction } from "remix";
import { redirect } from "remix";

export const action: ActionFunction = async () => {
  const userSession = await getUserSessionOrWhatever();

  if (!userSession) {
    return redirect("/login");
  }

  return json({ ok: true });
};
```

By default it sends 302, but you can change it to whichever redirect status code you'd like:

```ts
redirect(path, 301);
redirect(path, 303);
```

You can also send a `ResponseInit` to set headers, like committing a session.

```ts
redirect(path, {
  headers: {
    "Set-Cookie": await commitSession(session),
  },
});

redirect(path, {
  status: 302,
  headers: {
    "Set-Cookie": await commitSession(session),
  },
});
```

Of course, you can do redirects without this helper if you'd rather build it up yourself:

```ts
// this is a shortcut...
return redirect("/else/where", 303);

// ...for this
return new Response(null, {
  status: 303,
  headers: {
    Location: "/else/where",
  },
});
```

## `unstable_parseMultipartFormData` (node)

Allows you to handle multipart forms (file uploads) for your app.

Would be useful to understand [the Browser File API](https://developer.mozilla.org/en-US/docs/Web/API/File) to know how to use this API.

It's to be used in place of `request.formData()`.

```diff
- const formData = await request.formData();
+ const formData = await unstable_parseMultipartFormData(request, uploadHandler);
```

For example:

```tsx lines=[4-7,9,25]
export const action: ActionFunction = async ({
  request,
}) => {
  const formData = await unstable_parseMultipartFormData(
    request,
    uploadHandler // <-- we'll look at this deeper next
  );

  // the returned value for the file field is whatever our uploadHandler returns.
  // Let's imagine we're uploading the avatar to s3,
  // so our uploadHandler returns the URL.
  const avatarUrl = formData.get("avatar");

  // update the currently logged in user's avatar in our database
  await updateUserAvatar(request, avatarUrl);

  // success! Redirect to account page
  return redirect("/account");
};

export default function AvatarUploadRoute() {
  return (
    <Form method="post" encType="multipart/form-data">
      <label htmlFor="avatar-input">Avatar</label>
      <input id="avatar-input" type="file" name="avatar" />
      <button>Upload</button>
    </Form>
  );
}
```

### `uploadHandler`

The `uploadHandler` is the key to the whole thing. It's responsible for what happens to the file as it's being streamed from the client. You can save it to disk, store it in memory, or act as a proxy to send it somewhere else (like a file storage provider).

Remix has two utilities to create `uploadHandler`s for you:

- `unstable_createFileUploadHandler`
- `unstable_createMemoryUploadHandler`

These are fully featured utilities for handling fairly simple use cases. It's not recommended to load anything but quite small files into memory. Saving files to disk is a reasonable solution for many use cases. But if you want to upload the file to a file hosting provider, then you'll need to write your own.

#### `unstable_createFileUploadHandler`

**Example:**

```tsx
const uploadHandler = unstable_createFileUploadHandler({
  maxFileSize: 5_000_000,
  file: ({ filename }) => filename,
});

export const action: ActionFunction = async ({
  request,
}) => {
  const formData = await unstable_parseMultipartFormData(
    request,
    uploadHandler
  );

  const file = formData.get("avatar");

  // file is a "NodeFile" which has a similar API to "File"
  // ... etc
};
```

**Options:**

| Property           | Type               | Default                         | Description                                                                                                                                               |
| ------------------ | ------------------ | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| avoidFileConflicts | boolean            | true                            | Avoid file conflicts by appending a timestamp on the end of the filename if it already exists on disk                                                     |
| directory          | string \| Function | os.tmpdir()                     | The directory to write the upload.                                                                                                                        |
| file               | Function           | () => `upload_${random}.${ext}` | The name of the file in the directory. Can be a relative path, the directory structure will be created if it does not exist.                              |
| maxFileSize        | number             | 3000000                         | The maximum upload size allowed (in bytes). If the size is exceeded an error will be thrown.                                                              |
| filter             | Function           | OPTIONAL                        | A function you can write to prevent a file upload from being saved based on filename, mimetype, or encoding. Return `false` and the file will be ignored. |

The function API for `file` and `directory` are the same. They accept an `object` and return a `string`. The object it accepts has `filename`, `encoding`, and `mimetype` (all strings).The `string` returned is the path.

The `filter` function accepts an `object` and returns a `boolean` (or a promise that resolves to a `boolean`). The object it accepts has the `filename`, `encoding`, and `mimetype` (all strings). The `boolean` returned is `true` if you want to handle that file stream.

#### `unstable_createMemoryUploadHandler`

**Example:**

```tsx
const uploadHandler = unstable_createMemoryUploadHandler({
  maxFileSize: 500_000,
});

export const action: ActionFunction = async ({
  request,
}) => {
  const formData = await unstable_parseMultipartFormData(
    request,
    uploadHandler
  );

  const file = formData.get("avatar");

  // file is a "File" (https://mdn.io/File) polyfilled for node
  // ... etc
};
```

**Options:** The only options supported are `maxFileSize` and `filter` which work the same as in `unstable_createFileUploadHandler` above. This API is not recommended for anything at scale, but is a convenient utility for simple use cases.

### Custom `uploadHandler`

Most of the time, you'll probably want to proxy the file stream to a file host.

**Example:**

```tsx
import type { UploadHandler } from "remix";
import type {
  UploadApiErrorResponse,
  UploadApiOptions,
  UploadApiResponse,
  UploadStream,
} from "cloudinary";
import cloudinary from "cloudinary";

export const action: ActionFunction = async ({
  request,
}) => {
  const userId = getUserId(request);

  function uploadStreamToCloudinary(
    stream: Readable,
    options?: UploadApiOptions
  ): Promise<UploadApiResponse | UploadApiErrorResponse> {
    return new Promise((resolve, reject) => {
      const uploader = cloudinary.v2.uploader.upload_stream(
        options,
        (error, result) => {
          if (result) {
            resolve(result);
          } else {
            reject(error);
          }
        }
      );

      stream.pipe(uploader);
    });
  }

  const uploadHandler: UploadHandler = async ({
    name,
    stream,
  }) => {
    // we only care about the file form field called "avatar"
    // so we'll ignore anything else
    // NOTE: the way our form is set up, we shouldn't get any other fields,
    // but this is good defensive programming in case someone tries to hit our
    // action directly via curl or something weird like that.
    if (name !== "avatar") {
      stream.resume();
      return;
    }

    const uploadedImage = await uploadStreamToCloudinary(
      stream,
      {
        public_id: userId,
        folder: "/my-site/avatars",
      }
    );

    return uploadedImage.secure_url;
  };

  const formData = await unstable_parseMultipartFormData(
    request,
    uploadHandler
  );

  const imageUrl = formData.get("avatar");

  // because our uploadHandler returns a string, that's what the imageUrl will be.
  // ... etc
};
```

The `UploadHandler` function accepts a number of parameters about the file:

| Property | Type     | Description                                                                  |
| -------- | -------- | ---------------------------------------------------------------------------- |
| name     | string   | The field name (comes from your HTML form field "name" value)                |
| stream   | Readable | The stream of the file bytes                                                 |
| filename | string   | The name of the file that the user selected for upload (like `rickroll.mp4`) |
| encoding | string   | The encoding of the file (like `7bit`)                                       |
| mimetype | string   | The mimetype of the file (like `video/mp4`)                                  |

Your job is to do whatever you need with the `stream` and return a value that's a valid [`FormData`](https://developer.mozilla.org/en-US/docs/Web/API/FormData) value: [`File`](https://developer.mozilla.org/en-US/docs/Web/API/File), `string`, or `undefined`.

### Upload Handler Composition

We have the built-in `unstable_createFileUploadHandler` and `unstable_createMemoryUploadHandler` and we also expect more upload handler utilities to be developed in the future. If you have a form that needs to use different upload handlers, you can compose them together with a custom handler, here's a theoretical example:

```tsx
import type { UploadHandler } from "remix";
import { unstable_createFileUploadHandler } from "remix";
import { createCloudinaryUploadHandler } from "some-handy-remix-util";

export const fileUploadHandler =
  unstable_createFileUploadHandler({
    directory: "public/calendar-events",
  });

export const cloudinaryUploadHandler =
  createCloudinaryUploadHandler({
    folder: "/my-site/avatars",
  });

export const multHandler: UploadHandler = (args) => {
  if (args.name === "calendarEvent") {
    return fileUploadHandler(args);
  } else if (args.name === "eventBanner") {
    return cloudinaryUploadHandler(args);
  } else {
    args.stream.resume();
  }
};
```

## Cookies

A [cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) is a small piece of information that your server sends someone in a HTTP response that their browser will send back on subsequent requests. This technique is a fundamental building block of many interactive websites that adds state so you can build authentication (see [sessions][sessions]), shopping carts, user preferences, and many other features that require remembering who is "logged in".

Remix's `Cookie` interface provides a logical, reusable container for cookie metadata.

### Using cookies

While you may create these cookies manually, it is more common to use a [session storage][sessions].

In Remix, you will typically work with cookies in your `loader` and/or `action` functions (see <Link to="../mutations">mutations</Link>), since those are the places where you need to read and write data.

Let's say you have a banner on your e-commerce site that prompts users to check out the items you currently have on sale. The banner spans the top of your homepage, and includes a button on the side that allows the user to dismiss the banner so they don't see it for at least another week.

First, create a cookie:

```js filename=app/cookies.js
import { createCookie } from "remix";

export const userPrefs = createCookie("user-prefs", {
  maxAge: 604_800, // one week
});
```

Then, you can `import` the cookie and use it in your `loader` and/or `action`. The `loader` in this case just checks the value of the user preference so you can use it in your component for deciding whether to render the banner. When the button is clicked, the `<form>` calls the `action` on the server and reloads the page without the banner.

**Note:** We recommend (for now) that you create all the cookies your app needs in `app/cookies.js` and `import` them into your route modules. This allows the Remix compiler to correctly prune these imports out of the browser build where they are not needed. We hope to eventually remove this caveat.

```tsx filename=app/routes/index.tsx lines=[3,7-8,14-15,19]
import { useLoaderData, json, redirect } from "remix";

import { userPrefs } from "~/cookies";

export async function loader({ request }) {
  const cookieHeader = request.headers.get("Cookie");
  const cookie =
    (await userPrefs.parse(cookieHeader)) || {};
  return json({ showBanner: cookie.showBanner });
}

export async function action({ request }) {
  const cookieHeader = request.headers.get("Cookie");
  const cookie =
    (await userPrefs.parse(cookieHeader)) || {};
  const bodyParams = await request.formData();

  if (bodyParams.get("bannerVisibility") === "hidden") {
    cookie.showBanner = false;
  }

  return redirect("/", {
    headers: {
      "Set-Cookie": await userPrefs.serialize(cookie),
    },
  });
}

export default function Home() {
  const { showBanner } = useLoaderData();

  return (
    <div>
      {showBanner ? (
        <div>
          <Link to="/sale">Don't miss our sale!</Link>
          <Form method="post">
            <input
              type="hidden"
              name="bannerVisibility"
              value="hidden"
            />
            <button type="submit">Hide</button>
          </Form>
        </div>
      ) : null}
      <h1>Welcome!</h1>
    </div>
  );
}
```

### Cookie attributes

Cookies have [several attributes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#attributes) that control when they expire, how they are accessed, and where they are sent. Any of these attributes may be specified either in `createCookie(name, options)`, or during `serialize()` when the `Set-Cookie` header is generated.

```js
const cookie = createCookie("user-prefs", {
  // These are defaults for this cookie.
  domain: "remix.run",
  path: "/",
  sameSite: "lax",
  httpOnly: true,
  secure: true,
  expires: new Date(Date.now() + 60_000),
  maxAge: 60,
});

// You can either use the defaults:
cookie.serialize(userPrefs);

// Or override individual ones as needed:
cookie.serialize(userPrefs, { sameSite: "strict" });
```

Please read [more info about these attributes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#attributes) to get a better understanding of what they do.

### Signing cookies

It is possible to sign a cookie to automatically verify its contents when it is received. Since it's relatively easy to spoof HTTP headers, this is a good idea for any information that you do not want someone to be able to fake, like authentication information (see [sessions][sessions]).

To sign a cookie, provide one or more `secrets` when you first create the cookie:

```js
const cookie = createCookie("user-prefs", {
  secrets: ["s3cret1"],
});
```

Cookies that have one or more `secrets` will be stored and verified in a way that ensures the cookie's integrity.

Secrets may be rotated by adding new secrets to the front of the `secrets` array. Cookies that have been signed with old secrets will still be decoded successfully in `cookie.parse()`, and the newest secret (the first one in the array) will always be used to sign outgoing cookies created in `cookie.serialize()`.

```js
// app/cookies.js
const cookie = createCookie("user-prefs", {
  secrets: ["n3wsecr3t", "olds3cret"],
});

// in your route module...
export async function loader({ request }) {
  const oldCookie = request.headers.get("Cookie");
  // oldCookie may have been signed with "olds3cret", but still parses ok
  const value = await cookie.parse(oldCookie);

  new Response("...", {
    headers: {
      // Set-Cookie is signed with "n3wsecr3t"
      "Set-Cookie": await cookie.serialize(value),
    },
  });
}
```

### `createCookie`

Creates a logical container for managing a browser cookie from the server.

```ts
import { createCookie } from "remix";

const cookie = createCookie("cookie-name", {
  // all of these are optional defaults that can be overridden at runtime
  domain: "remix.run",
  expires: new Date(Date.now() + 60_000),
  httpOnly: true,
  maxAge: 60,
  path: "/",
  sameSite: "lax",
  secrets: ["s3cret1"],
  secure: true,
});
```

To learn more about each attribute, please see the [MDN Set-Cookie docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie#attributes).

### `isCookie`

Returns `true` if an object is a Remix cookie container.

```ts
import { isCookie } from "remix";
const cookie = createCookie("user-prefs");
console.log(isCookie(cookie));
// true
```

### Cookie API

A cookie container is returned from `createCookie` and has handful of properties and methods.

```ts
const cookie = createCookie(name);
cookie.name;
cookie.parse();
// etc.
```

#### `cookie.name`

The name of the cookie, used in `Cookie` and `Set-Cookie` HTTP headers.

#### `cookie.parse()`

Extracts and returns the value of this cookie in a given `Cookie` header.

```js
const value = await cookie.parse(
  request.headers.get("Cookie")
);
```

#### `cookie.serialize()`

Serializes a value and combines it with this cookie's options to create a `Set-Cookie` header, suitable for use in an outgoing `Response`.

```js
new Response("...", {
  headers: {
    "Set-Cookie": await cookie.serialize({
      showBanner: true,
    }),
  },
});
```

#### `cookie.isSigned`

Will be `true` if the cookie uses any `secrets`, `false` otherwise.

```js
let cookie = createCookie("user-prefs");
console.log(cookie.isSigned); // false

cookie = createCookie("user-prefs", {
  secrets: ["soopersekrit"],
});
console.log(cookie.isSigned); // true
```

#### `cookie.expires`

The `Date` on which this cookie expires. Note that if a cookie has both `maxAge` and `expires`, this value will be the date at the current time plus the `maxAge` value since `Max-Age` takes precedence over `Expires`.

```js
const cookie = createCookie("user-prefs", {
  expires: new Date("2021-01-01"),
});

console.log(cookie.expires); // "2020-01-01T00:00:00.000Z"
```

## Sessions

Sessions are an important part of websites that allow the server to identify requests coming from the same person, especially when it comes to server-side form validation or when JavaScript is not on the page. Sessions are a fundamental building block of many sites that let users "log in", including social, e-commerce, business, and educational websites.

In Remix, sessions are managed on a per-route basis (rather than something like express middleware) in your `loader` and `action` methods using a "session storage" object (that implements the `SessionStorage` interface). Session storage understands how to parse and generate cookies, and how to store session data in a database or filesystem.

Remix comes with several pre-built session storage options for common scenarios, and one to create your own:

- `createCookieSessionStorage`
- `createMemorySessionStorage`
- `createFileSessionStorage` (node)
- `createCloudflareKVSessionStorage` (cloudflare-workers)
- `createArcTableSessionStorage` (architect, Amazon DynamoDB)
- custom storage with `createSessionStorage`

### Using Sessions

This is an example of a cookie session storage:

```js filename=app/sessions.js
// app/sessions.js
import { createCookieSessionStorage } from "remix";

const { getSession, commitSession, destroySession } =
  createCookieSessionStorage({
    // a Cookie from `createCookie` or the CookieOptions to create one
    cookie: {
      name: "__session",

      // all of these are optional
      domain: "remix.run",
      expires: new Date(Date.now() + 60_000),
      httpOnly: true,
      maxAge: 60,
      path: "/",
      sameSite: "lax",
      secrets: ["s3cret1"],
      secure: true,
    },
  });

export { getSession, commitSession, destroySession };
```

We recommend setting up your session storage object in `app/sessions.js` so all routes that need to access session data can import from the same spot (also, see our [Route Module Constraints][constraints]).

The input/output to a session storage object are HTTP cookies. `getSession()` retrieves the current session from the incoming request's `Cookie` header, and `commitSession()`/`destroySession()` provide the `Set-Cookie` header for the outgoing response.

You'll use methods to get access to sessions in your `loader` and `action` functions.

A login form might look something like this:

```tsx filename=app/routes/login.js lines=[3,6-8,10,15,19,25-27,38,43,48,53]
import { json, redirect, useLoaderData } from "remix";

import { getSession, commitSession } from "../sessions";

export async function loader({ request }) {
  const session = await getSession(
    request.headers.get("Cookie")
  );

  if (session.has("userId")) {
    // Redirect to the home page if they are already signed in.
    return redirect("/");
  }

  const data = { error: session.get("error") };

  return json(data, {
    headers: {
      "Set-Cookie": await commitSession(session),
    },
  });
}

export async function action({ request }) {
  const session = await getSession(
    request.headers.get("Cookie")
  );
  const form = await request.formData();
  const username = form.get("username");
  const password = form.get("password");

  const userId = await validateCredentials(
    username,
    password
  );

  if (userId == null) {
    session.flash("error", "Invalid username/password");

    // Redirect back to the login page with errors.
    return redirect("/login", {
      headers: {
        "Set-Cookie": await commitSession(session),
      },
    });
  }

  session.set("userId", userId);

  // Login succeeded, send them to the home page.
  return redirect("/", {
    headers: {
      "Set-Cookie": await commitSession(session),
    },
  });
}

export default function Login() {
  const { currentUser, error } = useLoaderData();

  return (
    <div>
      {error ? <div className="error">{error}</div> : null}
      <form method="POST">
        <div>
          <p>Please sign in</p>
        </div>
        <label>
          Username: <input type="text" name="username" />
        </label>
        <label>
          Password:{" "}
          <input type="password" name="password" />
        </label>
      </form>
    </div>
  );
}
```

And then a logout form might look something like this:

```tsx
import { getSession, destroySession } from "../sessions";

export const action: ActionFunction = async ({
  request,
}) => {
  const session = await getSession(
    request.headers.get("Cookie")
  );
  return redirect("/login", {
    headers: {
      "Set-Cookie": await destroySession(session),
    },
  });
};

export default function LogoutRoute() {
  return (
    <>
      <p>Are you sure you want to log out?</p>
      <Form method="post">
        <button>Logout</button>
      </Form>
      <Link to="/">Never mind</Link>
    </>
  );
}
```

<docs-warning>It's important that you logout (or perform any mutation for that matter) in an `action` and not a `loader`. Otherwise you open your users to [Cross-Site Request Forgery](https://developer.mozilla.org/en-US/docs/Glossary/CSRF) attacks. Also, Remix only re-calls `loaders` when `actions` are called.</docs-warning>

### Session Gotchas

Because of nested routes, multiple loaders can be called to construct a single page. When using `session.flash()` or `session.unset()`, you need to be sure no other loaders in the request are going to want to read that, otherwise you'll get race conditions. Typically if you're using flash, you'll want to have a single loader read it, if another loader wants a flash message, use a different key for that loader.

### `createSession`

TODO:

### `isSession`

Returns `true` if an object is a Remix session.

```js
import { isSession } from "remix";

const sessionData = { foo: "bar" };
const session = createSession(sessionData, "remix-session");
console.log(isSession(session));
// true
```

### `createSessionStorage`

Remix makes it easy to store sessions in your own database if needed. The `createSessionStorage()` API requires a `cookie` (or options for creating a cookie, see [cookies][cookies]) and a set of create, read, update, and delete (CRUD) methods for managing the session data. The cookie is used to persist the session ID.

The following example shows how you could do this using a generic database client:

```js
import { createSessionStorage } from "remix";

function createDatabaseSessionStorage({
  cookie,
  host,
  port,
}) {
  // Configure your database client...
  const db = createDatabaseClient(host, port);

  return createSessionStorage({
    cookie,
    async createData(data, expires) {
      // `expires` is a Date after which the data should be considered
      // invalid. You could use it to invalidate the data somehow or
      // automatically purge this record from your database.
      const id = await db.insert(data);
      return id;
    },
    async readData(id) {
      return (await db.select(id)) || null;
    },
    async updateData(id, data, expires) {
      await db.update(id, data);
    },
    async deleteData(id) {
      await db.delete(id);
    },
  });
}
```

And then you can use it like this:

```js
const { getSession, commitSession, destroySession } =
  createDatabaseSessionStorage({
    host: "localhost",
    port: 1234,
    cookie: {
      name: "__session",
      sameSite: "lax",
    },
  });
```

The `expires` argument to `readData` and `updateData` is the same `Date` at which the cookie itself expires and is no longer valid. You can use this information to automatically purge the session record from your database to save on space, or to ensure that you do not otherwise return any data for old, expired cookies.

### `createCookieSessionStorage`

For purely cookie-based sessions (where the session data itself is stored in the session cookie with the browser, see [cookies][cookies]) you can use `createCookieSessionStorage()`.

The main advantage of cookie session storage is that you don't need any additional backend services or databases to use it. It can also be beneficial in some load balanced scenarios. However, cookie-based sessions may not exceed the browser's max allowed cookie length (typically 4kb).

The downside is that you have to `commitSession` in almost every loader and action. If your loader or action changes the session at all, it must be committed. That means if you `session.flash` in an action, and then `session.get` in another, you must commit it for that flashed message to go away. With other session storage strategies you only have to commit it when it's created (the browser cookie doesn't need to change because it doesn't store the session data, just the key to find it elsewhere).

```js
import { createCookieSessionStorage } from "remix";

const { getSession, commitSession, destroySession } =
  createCookieSessionStorage({
    // a Cookie from `createCookie` or the same CookieOptions to create one
    cookie: {
      name: "__session",
      secrets: ["r3m1xr0ck5"],
      sameSite: "lax",
    },
  });
```

### `createMemorySessionStorage`

This storage keeps all the cookie information in your server's memory.

<docs-error>This should only be used in development. Use one of the other methods in production.</docs-error>

```js
// app/sessions.js
import {
  createCookie,
  createMemorySessionStorage,
} from "remix";

// In this example the Cookie is created separately.
const sessionCookie = createCookie("__session", {
  secrets: ["r3m1xr0ck5"],
  sameSite: true,
});

const { getSession, commitSession, destroySession } =
  createMemorySessionStorage({
    cookie: sessionCookie,
  });

export { getSession, commitSession, destroySession };
```

### `createFileSessionStorage` (node)

For file-backed sessions, use `createFileSessionStorage()`. File session storage requires a file system, but this should be readily available on most cloud providers that run express, maybe with some extra configuration.

The advantage of file-backed sessions is that only the session ID is stored in the cookie while the rest of the data is stored in a regular file on disk, ideal for sessions with more than 4kb of data.

<docs-info>If you are deploying to a serverless function, ensure you have access to a persistent file system. They usually don't have one without extra configuration.</docs-info>

```js
// app/sessions.js
import {
  createCookie,
  createFileSessionStorage,
} from "remix";

// In this example the Cookie is created separately.
const sessionCookie = createCookie("__session", {
  secrets: ["r3m1xr0ck5"],
  sameSite: true,
});

const { getSession, commitSession, destroySession } =
  createFileSessionStorage({
    // The root directory where you want to store the files.
    // Make sure it's writable!
    dir: "/app/sessions",
    cookie: sessionCookie,
  });

export { getSession, commitSession, destroySession };
```

### `createCloudflareKVSessionStorage` (cloudflare-workers)

For [Cloudflare KV](https://developers.cloudflare.com/workers/learning/how-kv-works) backed sessions, use `createCloudflareKVSessionStorage()`.

The advantage of KV backed sessions is that only the session ID is stored in the cookie while the rest of the data is stored in a globally replicated, low-latency data store with exceptionally high read volumes with low-latency.

```js
// app/sessions.server.js
import {
  createCookie,
  createCloudflareKVSessionStorage,
} from "remix";

// In this example the Cookie is created separately.
const sessionCookie = createCookie("__session", {
  secrets: ["r3m1xr0ck5"],
  sameSite: true,
});

const { getSession, commitSession, destroySession } =
  createCloudflareKVSessionStorage({
    // The KV Namespace where you want to store sessions
    kv: YOUR_NAMESPACE,
    cookie: sessionCookie,
  });

export { getSession, commitSession, destroySession };
```

### `createArcTableSessionStorage` (architect, Amazon DynamoDB)

For [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/) backed sessions, use `createArcTableSessionStorage()`.

The advantage of DynamoDB backed sessions is that only the session ID is stored in the cookie while the rest of the data is stored in a globally replicated, low-latency data store with exceptionally high read volumes with low-latency.

```
# app.arc
sessions
  _idx *String
  _ttl TTL
```

```js
// app/sessions.server.js
import {
  createCookie,
  createArcTableSessionStorage,
} from "remix";

// In this example the Cookie is created separately.
const sessionCookie = createCookie("__session", {
  secrets: ["r3m1xr0ck5"],
  maxAge: 3600,
  sameSite: true,
});

const { getSession, commitSession, destroySession } =
  createArcTableSessionStorage({
    // The name of the table (should match app.arc)
    table: "sessions",
    // The name of the key used to store the session ID (should match app.arc)
    idx: "_idx",
    // The name of the key used to store the expiration time (should match app.arc)
    ttl: "_ttl",
    cookie: sessionCookie,
  });

export { getSession, commitSession, destroySession };
```

### Session API

After retrieving a session with `getSession`, the session object returned has a handful of methods and properties:

```js
export async function action({ request }) {
  const session = await getSession(
    request.headers.get("Cookie")
  );
  session.get("foo");
  session.has("bar");
  // etc.
}
```

#### `session.has(key)`

Returns `true` if the session has a variable with the given `name`.

```js
session.has("userId");
```

#### `session.set(key, value)`

Sets a session value for use in subsequent requests:

```js
session.set("userId", "1234");
```

#### `session.flash(key, value)`

Sets a session value that will be unset the first time it is read. After that, it's gone. Most useful for "flash messages" and server-side form validation messages:

```js
import { getSession, commitSession } from "../sessions";

export async function action({ request, params }) {
  const session = await getSession(
    request.headers.get("Cookie")
  );
  const deletedProject = await archiveProject(
    params.projectId
  );

  session.flash(
    "globalMessage",
    `Project ${deletedProject.name} successfully archived`
  );

  return redirect("/dashboard", {
    headers: {
      "Set-Cookie": await commitSession(session),
    },
  });
}
```

Now we can read the message in a loader.

<docs-info>You must commit the session whenever you read a `flash`. This is different than you might be used to where some type of middleware automatically sets the cookie header for you.</docs-info>

```jsx
import { Meta, Links, Scripts, Outlet, json } from "remix";

import { getSession, commitSession } from "./sessions";

export async function loader({ request }) {
  const session = await getSession(
    request.headers.get("Cookie")
  );
  const message = session.get("globalMessage") || null;

  return json(
    { message },
    {
      headers: {
        // only necessary with cookieSessionStorage
        "Set-Cookie": await commitSession(session),
      },
    }
  );
}

export default function App() {
  const { message } = useLoaderData();

  return (
    <html>
      <head>
        <Meta />
        <Links />
      </head>
      <body>
        {message ? (
          <div className="flash">{message}</div>
        ) : null}
        <Outlet />
        <Scripts />
      </body>
    </html>
  );
}
```

#### `session.get()`

Accesses a session value from a previous request:

```js
session.get("name");
```

#### `session.unset()`

Removes a value from the session.

```js
session.unset("name");
```

<docs-info>When using cookieSessionStorage, you must commit the session whenever you `unset`</docs-info>

```js
export async function loader({ request }) {
  // ...

  return json(data, {
    headers: {
      "Set-Cookie": await commitSession(session),
    },
  });
}
```

[cookies]: #cookies
[sessions]: #sessions
[constraints]: ../guides/constraints