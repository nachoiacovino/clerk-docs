---
sidebar_position: 1
---

# Next.js

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

Learn to install and initialize Clerk in a new Next.js application.

## 1. Create a new application

### 1.1 Run create command

Start by creating a new Next.js application.

```bash
npx create-next-app
# or
yarn create next-app
```

For more information about these commands, please reference the Next.js documentation.

### 1.2 Install clerk-react

Install Clerk's NPM package for React applications.

```bash
# From your application's root directory
cd yourapp

npm install @clerk/clerk-react
# or
yarn add @clerk/clerk-react
```

### 1.3 Add environment variables

Create a file named **.env.local** in your application root. Any variables inside this file with the **NEXT_PUBLIC** prefix will be accessible in your frontend via `process.env.NEXT_PUBLIC_X`.

:::info

Make sure you update these variables with the Client Host found in your dashboard.

:::

```bash
NEXT_PUBLIC_CLERK_FRONTEND_API=clerk.abc123.lcl.dev
NEXT_PUBLIC_CLERK_SIGN_IN=https://accounts.abc123.lcl.dev/sign-in
```

### 1.4 Start the dev server

```bash
npm run dev
# or
yarn dev
```

## 2. Configure pages/\_app.js

Clerk requires your application to be wrapped in the `<ClerkProvider>` context. In Next.js, we add this in **pages/\_app.js**.

:::info

In the code below, we've also added a sign in requirement for every page, unless the page is listed in the **publicPages** variable.

This is just one example of how authentication can be handled with Clerk. Feel free to adjust the strategy to better serve your use case!

:::

```jsx title="pages/_app.js"
import "../styles/globals.css";
import { ClerkProvider, SignedIn, SignedOut } from "@clerk/clerk-react";
import { useRouter } from "next/router";
import { useEffect } from "react";

// Retrieve Clerk settings from the environment
const clerkFrontendApi = process.env.NEXT_PUBLIC_CLERK_FRONTEND_API;
const clerkSignInURL = process.env.NEXT_PUBLIC_CLERK_SIGN_IN;

/**
 * List pages you want to be publicly accessible, or leave empty if
 * every page requires authentication. Use this naming strategy:
 *  "/"              for pages/index.js
 *  "/foo"           for pages/foo/index.js
 *  "/foo/bar"       for pages/foo/bar.js
 *  "/foo/[...bar]"  for pages/foo/[...bar].js
 */
const publicPages = [];

function MyApp({ Component, pageProps }) {
  const router = useRouter();
  /**
   * If the current route is listed as public, render it directly.
   * Otherwise, use Clerk to require authentication.
   */
  return (
    <ClerkProvider
      frontendApi={clerkFrontendApi}
      navigate={(to) => router.push(to)}
    >
      {publicPages.includes(router.pathname) ? (
        <Component {...pageProps} />
      ) : (
        <>
          <SignedIn>
            <Component {...pageProps} />
          </SignedIn>
          <SignedOut>
            <RedirectToSignIn />
          </SignedOut>
        </>
      )}
    </ClerkProvider>
  );
}

function RedirectToSignIn() {
  useEffect(() => {
    window.location = clerkSignInURL;
  });
  return null;
}

export default MyApp;
```

## 3. Hello, world!

That's all you need to configure with Clerk. Now you can say hello to your user:

```jsx title="pages/ìndex.js"
import { UserButton, useUser } from "@clerk/clerk-react";

export default function Home() {
  const { firstName } = useUser();
  return (
    <>
      <header>
        <UserButton />
      </header>
      <main>Hello, {firstName}!</main>
    </>
  );
}
```

Visit [https://localhost:3000](https://localhost:3000) to see your page. If you haven't signed in yet, you will be redirected to the sign in page.

## 4. Make a backend request

### 4.1 Add a fetcher utility

We've documented three strategies here, but you can use any strategy that meets the [request requirements for making backend requests](https://docs.clerk.dev/frontend/react/making-backend-requests).

<Tabs
defaultValue="swr"
values={[
{label: 'SWR', value: 'swr'},
{label: 'React Query', value: 'react-query'},
]}>

<TabItem value="swr">

**Install swr**

```bash
npm install swr
# or
yarn add swr
```

**Restart the application**

```bash
npm run dev
# or
yarn dev
```

**Create utils/fetchers.js**
Here, we wrap **useSWR** to add the active session ID to the request's query parameters. This way, one user will never retrieve data from another user's cache.

```jsx title="utils/fetchers.js"
import { useClerk } from "@clerk/clerk-react";
import useSWR from "swr";

export const useClerkSWR = (url, fetcher = null) => {
  const { session } = useClerk();
  if (!session) {
    throw new Error("Cannot useClerkSWR when there is no session.");
  }
  const sessionId = session.id;

  // The fetcher is not included as part of useSWR's cache key,
  // so we must append clerk session ID directly to the URL
  const urlWithSession = new URL(url, window.location.href);
  urlWithSession.searchParams.set("_clerk_session_id", sessionId);

  // The default fetcher includes credentials and returns json
  if (!fetcher) {
    fetcher = (request, options) => {
      return fetch(request, { ...options, credentials: "include" }).then((r) =>
        r.json(),
      );
    };
  }
  return useSWR(urlWithSession, fetcher);
};
```

</TabItem>

<TabItem value="react-query">

**Install react-query**

```bash
npm install react-query
# or
yarn add react-query
```

**Restart the application**

```bash
npm run dev
# or
yarn dev
```

**Create utils/fetchers.js**
Here we create a `<ClerkQueryClientProvider>` that switches query clients depending on the active session. This way, one user will never retrieve data from another user's cache.

```jsx title="utils/fetchers.js"
import { QueryClient, QueryClientProvider } from "react-query";
import { useContext } from "react";
import { ClerkContext } from "@clerk/clerk-react";

const clerkQueryClients = {};

// Use the queryKey as the URL and append _clerk_session_id
const clerkQueryFn = (sessionId) => {
  return ({ queryKey }) => {
    const urlWithSession = new URL(queryKey, window.location.href);
    urlWithSession.searchParams.set("_clerk_session_id", sessionId);
    return fetch(urlWithSession, { credentials: "include" }).then((r) =>
      r.json(),
    );
  };
};

export const ClerkQueryClientProvider = ({ children }) => {
  const { clerk } = useContext(ClerkContext);
  if (!clerk.session) {
    return children;
  }

  const { session } = clerk;

  if (!(session.id in clerkQueryClients)) {
    clerkQueryClients[session.id] = new QueryClient({
      defaultOptions: {
        queries: {
          queryFn: clerkQueryFn(session.id),
        },
      },
    });
  }

  return (
    <QueryClientProvider client={clerkQueryClients[session.id]}>
      {children}
    </QueryClientProvider>
  );
};
```

**Update pages/\_app.js**

First, add an import for the newly-created **ClerkQueryCleintProvider** to the top of the file.

```jsx title="pages/_app.js"
import { ClerkQueryClientProvider } from "../utils/fetchers";
```

Then, add `<ClerkQueryCleintProvider>` to **MyApp**'s return statement. You should nest it directly inside `<ClerkProvider>`:

```jsx title="pages/_app.js"
<ClerkProvider
  frontendApi={clerkFrontendApi}
  navigate={(to) => router.push(to)}
>
  <ClerkQueryClientProvider>// ...</ClerkQueryClientProvider>
</ClerkProvider>
```

</TabItem>
</Tabs>
