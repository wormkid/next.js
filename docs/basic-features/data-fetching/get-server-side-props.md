---
description: Fetch data on each request with `getServerSideProps`.
---

# getServerSideProps

만약 당신이 `getServerSideProps` (Server-Side Rendering) 라는 함수를 page에서 export 하면, Next.js 가 `getServerSideProps` 에서 리턴되는 데이터를 이용해서 각각의 리퀘스트마다 이 페이지를 pre-render할 것입니다.

```js
export async function getServerSideProps(context) {
  return {
    props: {}, // will be passed to the page component as props
  }
}
```

> 페이지로 전달되는 어떠한 `props` 라도 client-side의 초기 HTML에서 보여질 수 있으니 주의하세요. 이것은 이 페이지가 적절하게 [hydrated](https://react.dev/reference/react-dom/hydrate) 되도록 하기 위한 것입니다. 따라서 `props` 에 민감한 정보가 포함되지 않도록 주의하세요.

## getServerSideProps 는 언제 실행되는가

`getServerSideProps` 는 오직 서버에서만 실행되고 브라우저에서는 실행되지 않습니다. 만약 어느 페이지가 `getServerSideProps` 를 사용하면,:

- When you request this page directly, `getServerSideProps` runs at request time, and this page will be pre-rendered with the returned props
- 이 페이지를 직접 요청하면, `getServerSideProps` 가 요청시에 실행되고, 이 페이지는 리턴된 props를 이용해 pre-render 될 것입니다.
- When you request this page on client-side page transitions through [`next/link`](/docs/api-reference/next/link.md) or [`next/router`](/docs/api-reference/next/router.md), Next.js sends an API request to the server, which runs `getServerSideProps`
- 만약 이 페이지를 클라이언트에서 `next/link`나 `next/router`를 이용해서 페이지 전환을 하면 Next.js는 서버로 API 요청을 해서 `getServerSideProps`를 실행하게 됩니다.

`getServerSideProps` returns JSON which will be used to render the page. All this work will be handled automatically by Next.js, so you don’t need to do anything extra as long as you have `getServerSideProps` defined.
`getServerSideProps`는 페이지 렌더에 사용되는 JSON을 리턴합니다. 이 모든 과정은 Next.js에 의해서 자동으로 이뤄지기 때문에, 당신은 `getServerSideProps`를 정의하는 것 외에 다른 작업을 하지 않아도 됩니다.

You can use the [next-code-elimination tool](https://next-code-elimination.vercel.app/) to verify what Next.js eliminates from the client-side bundle.

`getServerSideProps` can only be exported from a **page**. You can’t export it from non-page files.

Note that you must export `getServerSideProps` as a standalone function — it will **not** work if you add `getServerSideProps` as a property of the page component.

The [`getServerSideProps` API reference](/docs/api-reference/data-fetching/get-server-side-props.md) covers all parameters and props that can be used with `getServerSideProps`.

## When should I use getServerSideProps

You should use `getServerSideProps` only if you need to render a page whose data must be fetched at request time. This could be due to the nature of the data or properties of the request (such as `authorization` headers or geo location). Pages using `getServerSideProps` will be server side rendered at request time and only be cached if [cache-control headers are configured](/docs/going-to-production#caching).

If you do not need to render the data during the request, then you should consider fetching data on the [client side](#fetching-data-on-the-client-side) or [`getStaticProps`](/docs/basic-features/data-fetching/get-static-props).

### getServerSideProps or API Routes

It can be tempting to reach for an [API Route](/docs/api-routes/introduction.md) when you want to fetch data from the server, then call that API route from `getServerSideProps`. This is an unnecessary and inefficient approach, as it will cause an extra request to be made due to both `getServerSideProps` and API Routes running on the server.

Take the following example. An API route is used to fetch some data from a CMS. That API route is then called directly from `getServerSideProps`. This produces an additional call, reducing performance. Instead, directly import the logic used inside your API Route into `getServerSideProps`. This could mean calling a CMS, database, or other API directly from inside `getServerSideProps`.

### getServerSideProps with Edge API Routes

`getServerSideProps` can be used with both Serverless and Edge Runtimes, and you can set props in both. However, currently in Edge Runtime, you do not have access to the response object. This means that you cannot — for example — add cookies in `getServerSideProps`. To have access to the response object, you should **continue to use the Node.js runtime**, which is the default runtime.

You can explicitly set the runtime on a [per-page basis](https://nextjs.org/docs/advanced-features/react-18/switchable-runtime#page-runtime-option) by modifying the `config`, for example:

```js
export const config = {
  runtime: 'nodejs',
}
```

## Fetching data on the client side

If your page contains frequently updating data, and you don’t need to pre-render the data, you can fetch the data on the [client side](/docs/basic-features/data-fetching/client-side.md). An example of this is user-specific data:

- First, immediately show the page without data. Parts of the page can be pre-rendered using Static Generation. You can show loading states for missing data
- Then, fetch the data on the client side and display it when ready

This approach works well for user dashboard pages, for example. Because a dashboard is a private, user-specific page, SEO is not relevant and the page doesn’t need to be pre-rendered. The data is frequently updated, which requires request-time data fetching.

## Using getServerSideProps to fetch data at request time

The following example shows how to fetch data at request time and pre-render the result.

```jsx
function Page({ data }) {
  // Render data...
}

// This gets called on every request
export async function getServerSideProps() {
  // Fetch data from external API
  const res = await fetch(`https://.../data`)
  const data = await res.json()

  // Pass data to the page via props
  return { props: { data } }
}

export default Page
```

## Caching with Server-Side Rendering (SSR)

You can use caching headers (`Cache-Control`) inside `getServerSideProps` to cache dynamic responses. For example, using [`stale-while-revalidate`](https://web.dev/stale-while-revalidate/).

```jsx
// This value is considered fresh for ten seconds (s-maxage=10).
// If a request is repeated within the next 10 seconds, the previously
// cached value will still be fresh. If the request is repeated before 59 seconds,
// the cached value will be stale but still render (stale-while-revalidate=59).
//
// In the background, a revalidation request will be made to populate the cache
// with a fresh value. If you refresh the page, you will see the new value.
export async function getServerSideProps({ req, res }) {
  res.setHeader(
    'Cache-Control',
    'public, s-maxage=10, stale-while-revalidate=59'
  )

  return {
    props: {},
  }
}
```

Learn more about [caching](/docs/going-to-production.md).

## Does getServerSideProps render an error page

If an error is thrown inside `getServerSideProps`, it will show the `pages/500.js` file. Check out the documentation for [500 page](/docs/advanced-features/custom-error-page#500-page) to learn more on how to create it. During development this file will not be used and the dev overlay will be shown instead.

## Related

For more information on what to do next, we recommend the following sections:

<div class="card">
  <a href="/docs/api-reference/data-fetching/get-server-side-props.md">
    <b>getServerSideProps API Reference</b>
    <small>Read the API Reference for getServerSideProps</small>
  </a>
</div>
