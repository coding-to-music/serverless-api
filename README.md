# Build a serverless API for your frontend using Cloudflare Workers

---
pcx-content-type: tutorial
---
https://developers.cloudflare.com/pages/tutorials/build-an-api-with-workers

https://github.com/cloudflare/cloudflare-docs/blob/production/products/pages/src/content/tutorials/build-an-api-with-workers/index.md


## Introduction

In this tutorial, you will build an API on [Cloudflare Workers](https://www.cloudflare.com/learning/serverless/glossary/serverless-and-cloudflare-workers/) that can be used by your Pages application. Workers serve as a great companion to your front-end applications on Cloudflare Pages. In this tutorial, you will build a simple JSON API that returns blog posts that can be retrieved and rendered in a frontend application.

This tutorial contains two, separate applications: the backend, a [serverless](https://www.cloudflare.com/learning/serverless/what-is-serverless/) API deployed on Cloudflare Workers, and the front end, built with React and deployed using Cloudflare Pages.

If you are interested in a more comprehensive approach to building applications like this, you may benefit from moving from a serverless API for your content to a headless CMS — refer to the [headless CMS tutorial] to learn more.

## Deploying a serverless API

### Generating a new project

Begin by creating a new Cloudflare Workers project. If you have not used Cloudflare Workers, or installed [Wrangler](https://developers.cloudflare.com/workers/cli-wrangler/install-update), the command-line tool for managing and publishing Workers projects, refer to the [Get started guide](https://developers.cloudflare.com/workers/get-started/guide) in the Workers documentation. Once you have configured Wrangler and authenticated it with your Cloudflare account, return here to generate your API codebase.

You will use the Workers TypeScript template to generate our project. Don't worry if you do not know TypeScript — you will not be writing any complicated types, and if you are using VS Code or another editor with TypeScript support, your code will be validated and checked by the editor as you build your application. Run `wrangler generate` in your terminal to create a new project using the template:

```sh
---
header: Creating a new Workers project with Wrangler
---
wrangler generate serverless-api https://github.com/cloudflare/worker-typescript-template
```

### Adding a router

Your Workers application will serve as a backend API to return blog post data using JSON to our static application. This means that it should have two routes: `/api/posts`, which will return a list of blog posts, and `/api/posts/:id`, which will be used to retrieve a specific blog post based on ID.

While you can manually parse incoming URLs, it is much easier to use a routing library to consistently handle incoming requests. Install and use [itty-router](https://github.com/kwhitley/itty-router), an open-source routing library with support for Cloudflare Workers by running the following command in your terminal:

```sh
---
header: Installing itty-router
---
npm install itty-router
---
```

With `itty-router` installed, you can open up `handler.ts` and set up your router. Begin by importing the `itty-router` package and setting up a new instance of the `Router` class:

```ts
---
filename: "src/handler.ts"
---
import { Router } from 'itty-router'

import Posts from './handlers/posts'
import Post from './handlers/post'

const router = Router()

router
  .get('/api/posts', Posts)
  .get('/api/posts/:id', Post)
  .get('*', () => new Response("Not found", { status: 404 }))

export const handleRequest = request => router.handle(request)
```

The above routing configuration defines three routes: `/api/posts`, `/api/posts/:id`, and a wildcard route, which will be called if the incoming request does not match the first two routes.

Create two files, `handlers/posts.ts` and `handlers/post.ts`, which will contain the handler code for our two API routes.

### Implementing API routes

In `handlers/posts.ts`, you will return an array of posts as a JSON array. If you have never worked with JSON APIs before, all you need to know is that a serverless API returning JSON needs to define a `Content-type` header, which should be set to `application/json`. Any JSON data coming back as part of the response body should be turned into a JSON string using `JSON.stringify`.

In the below sample, you will stub out the `posts` array, and later, you will come back and fill it in with real data:

```ts
---
filename: "src/handlers/posts.ts"
---
const posts = []

const Posts = () => {
  const body = JSON.stringify(posts)
  const headers = { 'Content-type': 'application/json' }
  return new Response(body, { headers })
}

export default Posts
```

`Posts` is a simple function with no arguments that returns a JSON response. When an application makes a GET request to `/api/posts`, they will receive a JSON-encoded array back, which they can use to render a list of blog posts.

You will define something similar for `handlers/post.ts`, which returns a single blog post. Importantly, you will use the request argument that `itty-router` passes to handlers, using it to retrieve the `:id` parameter that we defined in `handler.ts`. Again, you will stub out the `post` data, but you can already begin to see in the below sample how to retrieve the `:id` param inside of a handler:

```ts
---
filename: "src/handlers/post.ts"
---
const post = {}

const Post = request => {
  // This will be used soon to retrieve a post
  const postId = request.params.id

  const body = JSON.stringify(post)
  const headers = { 'Content-type': 'application/json' }
  return new Response(body, { headers })
}

export default Post
```

### Defining a static data class

Until now, you have used empty stub data to return data in our API routes. To make this application meaningful, define a static data class, called `PostsStore`, to simulate how you might retrieve data from a database or other data source. In `posts_store.ts`:

```ts
---
filename: "src/posts_store.ts"
---
const _posts = [
  {
    id: 1,
    title: "My first blog post",
    text: "Hello world! This is my first blog post on my new Cloudflare Workers + Pages blog.",
    published_at: new Date("2020-10-23")
  },
  {
    id: 2,
    title: "Updating my blog",
    text: "It's my second blog post! I'm still writing and publishing using Cloudflare Workers + Pages :)",
    published_at: new Date("2020-10-26")
  }
]

export default class PostsStore {
  async all() {
    return _posts
  }

  async find(id: number) {
    return _posts.find(post => post.id.toString() === id.toString())
  }
}
```

You are still referring to static content, but by building a `PostsStore` class, you can abstract the retrieval of posts and easily imagine a future where this class talks to a database, key-value store, or however you prefer to store your data.

With `PostsStore` set up, you can import it and use it in our handlers:

```ts
---
filename: "src/handlers/posts.ts"
highlight: [1, 4, 5]
---
import Store from '../posts_store'

const Posts = async () => {
  const posts = new Store()
  const body = JSON.stringify(await posts.all())
  const headers = { 'Content-type': 'application/json' }
  return new Response(body, { headers })
}

export default Posts
```

```ts
---
filename: "src/handlers/post.ts"
highlight: [1, 4, 7]
---
import Store from '../posts_store'

const Post = async request => {
  const posts = new Store()
  const postId = request.params.id

  const body = JSON.stringify(await posts.find(postId))
  const headers = { 'Content-type': 'application/json' }
  return new Response(body, { headers })
}

export default Post
```

wrangler dev
```java
wrangler dev
```

Output
```java
🌀  Running npm install && npm run build

up to date, audited 593 packages in 2s

52 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

> worker-typescript-template@1.0.0 build
> webpack

asset worker.js 6.04 KiB [emitted] (name: main) 1 related asset
modules by path ./src/*.ts 1.62 KiB
  ./src/index.ts 223 bytes [built] [code generated]
  ./src/handler.ts 706 bytes [built] [code generated] [1 error]
  ./src/posts_store.ts 725 bytes [built] [code generated]
modules by path ./src/handlers/*.ts 1.11 KiB
  ./src/handlers/posts.ts 541 bytes [built] [code generated]
  ./src/handlers/post.ts 591 bytes [built] [code generated] [1 error]
./node_modules/itty-router/dist/itty-router.min.js 546 bytes [built] [code generated]

ERROR in /mnt/ap/ap/serverless-api/src/handler.ts
./src/handler.ts 13:29-36
[tsl] ERROR in /mnt/ap/ap/serverless-api/src/handler.ts(13,30)
      TS7006: Parameter 'request' implicitly has an 'any' type.
ts-loader-default_e3b0c44298fc1c14
 @ ./src/index.ts 3:18-38

ERROR in /mnt/ap/ap/serverless-api/src/handlers/post.ts
./src/handlers/post.ts 3:19-26
[tsl] ERROR in /mnt/ap/ap/serverless-api/src/handlers/post.ts(3,20)
      TS7006: Parameter 'request' implicitly has an 'any' type.
ts-loader-default_e3b0c44298fc1c14
 @ ./src/handler.ts 9:31-57
 @ ./src/index.ts 3:18-38

webpack 5.38.1 compiled with 2 errors in 3135 ms
Error: Build failed! Status Code: 1
```

### Adding CORS headers

Before you are ready to deploy, you will make one more change to our handlers, adding [CORS headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) to allow your frontend application to make requests to the API. In your handlers, you will update the `headers` variable accordingly:

```ts
---
filename: "src/handlers/posts.ts"
highlight: [6, 7, 8, 9]
---
import Store from '../posts_store'

const Posts = async () => {
  const posts = new Store()
  const body = JSON.stringify(await posts.all())
  const headers = {
    'Access-Control-Allow-Origin': '*',
    'Content-type': 'application/json'
  }
  return new Response(body, { headers })
}

export default Posts
```

```ts
---
filename: "src/handlers/post.ts"
highlight: [8, 9, 10, 11]
---
import Store from '../posts_store'

const Post = async request => {
  const posts = new Store()
  const postId = request.params.id

  const body = JSON.stringify(await posts.find(postId))
  const headers = {
    'Access-Control-Allow-Origin': '*',
    'Content-type': 'application/json'
  }
  return new Response(body, { headers })
}

export default Post
```

wrangler dev
```java
wrangler dev
```

Output
```java
🌀  Running npm install && npm run build

up to date, audited 593 packages in 2s

52 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

> worker-typescript-template@1.0.0 build
> webpack

asset worker.js 6.15 KiB [emitted] (name: main) 1 related asset
modules by path ./src/*.ts 1.62 KiB
  ./src/index.ts 223 bytes [built] [code generated]
  ./src/handler.ts 706 bytes [built] [code generated] [1 error]
  ./src/posts_store.ts 725 bytes [built] [code generated]
modules by path ./src/handlers/*.ts 1.21 KiB
  ./src/handlers/posts.ts 597 bytes [built] [code generated]
  ./src/handlers/post.ts 647 bytes [built] [code generated] [1 error]
./node_modules/itty-router/dist/itty-router.min.js 546 bytes [built] [code generated]

ERROR in /mnt/ap/ap/serverless-api/src/handler.ts
./src/handler.ts 13:29-36
[tsl] ERROR in /mnt/ap/ap/serverless-api/src/handler.ts(13,30)
      TS7006: Parameter 'request' implicitly has an 'any' type.
ts-loader-default_e3b0c44298fc1c14
 @ ./src/index.ts 3:18-38

ERROR in /mnt/ap/ap/serverless-api/src/handlers/post.ts
./src/handlers/post.ts 3:19-26
[tsl] ERROR in /mnt/ap/ap/serverless-api/src/handlers/post.ts(3,20)
      TS7006: Parameter 'request' implicitly has an 'any' type.
ts-loader-default_e3b0c44298fc1c14
 @ ./src/handler.ts 9:31-57
 @ ./src/index.ts 3:18-38

webpack 5.38.1 compiled with 2 errors in 1348 ms
Error: Build failed! Status Code: 1
```

### fixing the errors

My offending line is:

There is this line: 

```java
export const handleRequest = request => router.handle(request)
```

Per this:

https://stackoverflow.com/questions/43064221/typescript-ts7006-parameter-xxx-implicitly-has-an-any-type


Change this line:

```java
let user = Users.find(user => user.id === query);
```

to this:

```java
let user = Users.find((user: any) => user.id === query); 
// use "any" or some other interface to type this argument
```

### In src/handler.ts

Change from:

```java
export const handleRequest = request => router.handle(request)
```

to

```java
export const handleRequest = (request: any) => router.handle(request)
```

### In src/handlers/post.ts

Change from:

```java
const Post = async request => {
```

to

```java
const Post = async (request: any) => {
```


### Publishing the API

With your API configured, you are ready to publish. Run `wrangler publish` in your terminal, and when you have successfully deployed your application, you should be able to make requests to your API to see data returned in the console:

wrangler publish
```java
wrangler publish
```

Output
```java
🌀  Running npm install && npm run build

up to date, audited 593 packages in 2s

52 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities

> worker-typescript-template@1.0.0 build
> webpack

asset worker.js 6.15 KiB [compared for emit] (name: main) 1 related asset
modules by path ./src/*.ts 1.62 KiB
  ./src/index.ts 223 bytes [built] [code generated]
  ./src/handler.ts 708 bytes [built] [code generated]
  ./src/posts_store.ts 725 bytes [built] [code generated]
modules by path ./src/handlers/*.ts 1.21 KiB
  ./src/handlers/posts.ts 597 bytes [built] [code generated]
  ./src/handlers/post.ts 647 bytes [built] [code generated]
./node_modules/itty-router/dist/itty-router.min.js 546 bytes [built] [code generated]
webpack 5.38.1 compiled successfully in 2648 ms
✨  Build completed successfully!
✨  Successfully published your script to
 https://serverless-api.coding-to-music.workers.dev
```

### Testing the API

```java
---
header: "Testing the API"
---
curl serverless-api.coding-to-music.workers.dev/api/posts
curl serverless-api.coding-to-music.workers.dev/api/posts/1
curl serverless-api.coding-to-music.workers.dev/api/posts/2

# or 

curl serverless-api.coding-to-music.workers.dev/api/posts | jq
curl serverless-api.coding-to-music.workers.dev/api/posts/1 | jq
curl serverless-api.coding-to-music.workers.dev/api/posts/2 | jq
```

call api/posts
```java
curl https://serverless-api.coding-to-music.workers.dev/api/posts | jq
```

```java
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   355  100   355    0     0   6016      0 --:--:-- --:--:-- --:--:--  6016
[
  {
    "id": 1,
    "title": "My first blog post",
    "text": "Hello world! This is my first blog post on my new Cloudflare Workers + Pages blog.",
    "published_at": "2020-10-23T00:00:00.000Z"
  },
  {
    "id": 2,
    "title": "Updating my blog",
    "text": "It's my second blog post! I'm still writing and publishing using Cloudflare Workers + Pages :)",
    "published_at": "2020-10-26T00:00:00.000Z"
  }
]
```

call api/posts/1
```java
curl https://serverless-api.coding-to-music.workers.dev/api/posts/1 | jq
```

```java
| jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   171  100   171    0     0   3109      0 --:--:-- --:--:-- --:--:--  3109
{
  "id": 1,
  "title": "My first blog post",
  "text": "Hello world! This is my first blog post on my new Cloudflare Workers + Pages blog.",
  "published_at": "2020-10-23T00:00:00.000Z"
}
```

## Deploying a new React application to Pages

With your serverless API deployed, we can now build the frontend of our application with React. First, you will generate the application, and then you will define the functionality by adding routing, and rendering blog posts from the API. Once you are happy with the implementation locally, you will use Cloudflare Pages to deploy it in just a matter of minutes.

### Generating a new React application

To start, create a new React app using `create-react-app` in your terminal, and then navigate into the directory and start a local development server:

```java
---
header: "Creating a new React application"
---
npx create-react-app blog-frontend
```

Output
```java
Done in 13.00s.

Created git commit.

Success! Created blog-frontend at /mnt/ap/ap/blog-frontend
Inside that directory, you can run several commands:

  yarn start
    Starts the development server.

  yarn build
    Bundles the app into static files for production.

  yarn test
    Starts the test runner.

  yarn eject
    Removes this tool and copies build dependencies, configuration files
    and scripts into the app directory. If you do this, you can’t go back!

We suggest that you begin by typing:

  cd blog-frontend
  yarn start

Happy hacking!
```

npm start
```java
cd blog-frontend
npm start
```

Start up the app locally, and clear out the contents of `App.js`:

```js
---
filename: "src/App.js"
---
function App() {
  return (
    <div>
      <span>Hello world</span>
    </div>
  );
}

export default App;
```

### Adding routing and consuming blog posts

Add `@reach/router`:

```sh
---
header: "Adding @reach/router"
---
yarn add @reach/router
```

Import it into `App.js`, and set up a new router with two routes:

```js
---
filename: "src/App.js"
---
import { Router } from "@reach/router";

import Posts from './components/posts'
import Post from './components/post'

function App() {
  return (
    <Router>
      <Posts path="/" />
      <Post path="/posts/:id" />
    </Router>
  );
}

export default App;
```

Create a new folder called `components`, and inside of it, create two files: `posts.js`, and `post.js`. These components will load the blog posts from our API, and render them. Begin with `posts.js`:

```java
mkdir src/components
touch src/components/posts.js
touch src/components/post.js
```


```java
---
filename: "src/components/posts.js"
---
import React, { useEffect, useState } from "react";
import { Link } from "@reach/router";

const Posts = () => {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    const getPosts = async () => {
      const resp = await fetch(
        "https://serverless-api.coding-to-music.workers.dev/api/posts"
      );
      const postsResp = await resp.json();
      setPosts(postsResp);
    };

    getPosts();
  }, []);

  return (
    <div>
      <h1>Posts</h1>
      {posts.map((post) => (
        <div key={post.id}>
          <h2>
            <Link to={`/posts/${post.id}`}>{post.title}</Link>
          </h2>
        </div>
      ))}
    </div>
  );
};

export default Posts;
```

Next, add the component for individual blog posts, in `src/components/post.js`:

```js
---
filename: "src/components/post.js"
---
import React, { useEffect, useState } from "react";
import { Link } from "@reach/router";

const Post = ({ id }) => {
  const [post, setPost] = useState({});

  useEffect(() => {
    const getPost = async () => {
      const resp = await fetch(
        `https://serverless-api.coding-to-music.workers.dev/api/posts/${id}`
      );
      const postResp = await resp.json();
      setPost(postResp);
    };

    getPost();
  }, [id]);

  if (!Object.keys(post).length) return <div />;

  return (
    <div>
      <h1>{post.title}</h1>
      <p>{post.text}</p>
      <p>
        <em>Published {new Date(post.published_at).toLocaleString()}</em>
      </p>
      <p>
        <Link to="/">Go back</Link>
      </p>
    </div>
  );
};

export default Post;
```

### Publishing with Cloudflare Pages

Publishing your project with Cloudflare Pages is an easy, two-step process: first, push your project to GitHub, and then in the Cloudflare Pages dashboard, set up a new project based on that GitHub repository. 

Pages will deploy a new version of your site each time you publish and will set up preview deployments whenever you open a new pull request.

To push your project to GitHub, [create a new repository](https://repo.new), and follow the instructions to push your local Git repository to GitHub.


```java
gh repo create blog-frontend
```

Output
```java
gh repo create blog-frontend
? Visibility Public
? Would you like to add a .gitignore? No
? Would you like to add a license? No
? This will create the "blog-frontend" repository on GitHub. Continue? Yes
✓ Created repository coding-to-music/blog-frontend on GitHub
? Create a local project directory for "coding-to-music/blog-frontend"? No
```

Go to the new repo on github and copy the following:
```java
git remote add origin git@github.com:coding-to-music/blog-frontend.git
git branch -M main
git push -u origin main
```


Deploy your site to Pages by logging into the [Cloudflare dashboard](https://dash.cloudflare.com/) > **Account Home** > **Pages** and selecting **Create a project**. Select the new GitHub repository that you created and, in the **Set up builds and deployments** section, choose **React** -- Pages will automatically apply the correct build settings for you.

When your site has been deployed, you will receive a unique URL to view it in production.

You have now deployed your own blog, powered by Cloudflare Workers and Cloudflare Pages.

## Custom domains

Cloudflare Workers and Pages both provide first-class support for custom domains. This means that you can deploy your Pages application to a custom domain, as you would for any normal website, and also deploy your API (`/api/posts` and `/api/posts/:id`) "over" your Pages application, by specifying a route in your `wrangler.toml` file.

For example, given the example domain `myblog.com`, you could set up your API application's `wrangler.toml` with the following config:

```toml
---
filename: wrangler.toml
---
workers_dev = false
route = "*myblog.com/api*"
zone_id = "$zoneId"
```

Now, when you run `wrangler publish`, your API will be published and served on `myblog.com`, but only when you visit a route matching `/api*` -- this means that your basic routes (`/`, `/posts/:id`, etc.) will be sent to your front-end Pages application, but any API routes will be intercepted and handled by your Workers application.

## Conclusion

In this tutorial, you built a full blog application by combining a front end deployed with Cloudflare Pages, and a serverless API built with Cloudflare Workers. You can find the source code for both codebases on GitHub:

- Blog frontend: https://github.com/coding-to-music/blog-frontend
- Serverless API: https://github.com/coding-to-music/serverless-api

If you enjoyed this tutorial, refer to the [headless CMS tutorial] to learn how to build a blog using Nuxt.js and Sanity.io.

[headless CMS tutorial]:/tutorials/build-a-blog-using-nuxt-and-sanity