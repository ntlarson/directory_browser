# Problem
The problem before us today is how to allow a user a safe and secure way to browse files that have been uploaded to a remote directory. 

# Proposed Solution
The solution will take shape in 2 distinct parts, a node/express based backend and a react/react-router based frontend

### Backend

###### Technology:
The backend will be written in NodeJs with Typescript and Express

###### API Design:

I am a fan of versioned api's to allow for better deprecation, organization, and controlled support. For the basic nature of the application I am designing, I intend to support the following behaviors:

1) Login
    - route: /api/v1/login
    - method: POST
    - payload: {username: string, password: string}
    - response: 200 if successful, 401 if not

2) Logout
    - route: /api/v1/logout
    - method: GET
    - payload: nothing
    - response: 200 on successful clearing of the associated session/cookie

3) list-dir
    - route: /api/v1/lsdir
    - method: GET
    - query params: I will support a `prefix` query param to allow drilling down into sub-directories via the api
    - response: a JSON string representation of the directory contents

###### Security:

A system such as this needs to be locked down to prevent unauthorized access. 

I intend to develop a session based authentication system. For ease and scope in this project, I will not be setting up a DB. I will use in-memory and files to manage static data such as password hash and session tokens. A key aspect of this design will be to account for, and mitigate, risk of common attack vectors. The two most common types of issues we will see here are XSS attacks and CSRF attacks. XSS is a vector largely to be resolved in the UI piece of the solution so I will address plans to mitigate that there. CSRF is a problem for the API itself to manage. I will employ a CSRF token in my session in order to allow for verification that requests are coming from the correct place. 

I intend to use the following packages for token managagement:
-- express-session - manage session security and cookies
-- csurf - manage csrf tokens

My cookie will be configured/secured with the following configuration settings

```
{
    secure: true,
    httpOnly: true,
    domain: 'localhost',
    path: '/',
    expires: someDateValue
}
```

Since I intend to use the `secure` setting, this means I will need to setup a self-signed cert and enable https for my server.

###### Application design:

I intend to use a modular nested design where things are broken out into common directories. To manage complexity I particularly intend to break all api route handling into separate files. I envision a structure similar to below.

```

application-root/
    node_modules/
    package.json
    config.ts
    client-src/
    server-src/
        index.ts
        router.ts
        middlewares/
            middleware_0.ts
            ...
            middleware_n.ts
        utils/
            util_file_0.ts
            ...
            util_file_n.ts
        controllers/
            index.ts
            controller_0.ts
            ...
            controller_n.ts
    user-dirs/
        username0/
        username1/
        username2/
```

As this is a minimal application I will use node for all server needs. Any path that does not begin with `/api` will be returned the index.html file for the React SPA.

For the specifics of the task at hand I will assume a basic structure on the server that each user would have a directory associated with their username and that is the "root" directory structure from the api perspective. The ability to do a dir listing from Node is fairly straight forward using the `path` and `fs` modules. I will use the JSON structure laid out in the task description as a model/starting point to build my payload for the lsdir api endpoint. However, I intend to handle drilling down a little differently and therefore the payload will have a slightly different shape. 

```
const directorystats = {
  path: 'user1/'
  items: [
    {
      name: "images",
      sizeKb: 0,
      type: "dir"
    },
    {
      name: "documents",
      sizeKb: 0,
      type: "dir"
    },
    {
      name: "README.md",
      sizeKb: 4340,
      type: "file",
    },
  ],
};
```

My reasoning here is to balance performance and memory. In the application here, scale will not likely be an issue, but as I have worked in similar systems (in my previous job I built a few directory browsers based on Amazon s3 bucket contents) and trying to get a full listing of all nested directories (prefixes) became a very expensive operation both in terms of api processing and client side memory overhead. To me having an api endpoint that gives the immediate listing and then on interaction with the UI if a sub-directory is selected the api takes that value and gives the next layer of results is a better system than recursively collecting n+1 layers nested directories and files into a single large payload to load into the browser memory.

In this example the base response would be something like above and were the user to select `images` from the ui list, the request/response would look something like the following:

`https://localhost/api/v1/lsdir?prefix="images"`

```
const directorystats = {
  path: 'user1/images/'
  items: [
    {
      name: "icons",
      sizeKb: 0,
      type: "dir"
    },
    {
      name: "screenshots",
      sizeKb: 0,
      type: "dir"
    },
    {
      name: "other_image_0.png",
      sizeKb: 4340,
      type: "file",
    },
    {
      name: "..."
    },
    {
      name: "other_image_n.png",
      sizeKb: 98832,
      type: "file"
    }
  ],
};
```

### Frontend

###### Technology:
The frontend will be written using React and Typescript. Supporting modules and packages will be: 

`create-react-app` to bootstrap and manage the basic build for this application

`react-router` to manage client side routing

`fetch` native browser http client

`react-feather` for icon support

###### Application Design:
Similar to the backend, I want a clean, modular folder structure that makes it easy to find and maintain files. After a many years of folder structures and various framework interactions I am not sure there really is a solid, clean answer on the question of how to organize components, however, this is what I have come up with 

```
application-root/
    node_modules/
    package.json/
    config.ts
    server-src/
    client-src/
        index.tsx
        index.css
        react-app-env.d.ts
        reportWebVitals.ts
        setupTests.ts
        components/
            composites/
                composite_0.tsx
                ...
                composite_n.tsx
            widgets/
                widget_0.tsx
                ...
                widget_n.tsx
            elements/
                element_0.tsx
                ...
                element_n.tsx
``` 

I break up my components into these categories as I feel it provides a better idea of categorization. My definitions are as follows:

`composites`: composites are full view components. These should map to routes and paths in the application router (at broad levels). So in this case the base application will have a `/login` and a `/browse` path. I will therefore start with two composites `login.tsx` and `browse.tsx`. Composites will generally consume `widgets` and `elements` and are not likely to be re-used anywhere on the site.

`widget`: widgets are components of medium complexity. These would be more complex structures, such as a search widget that would be composed of a text input, a button, and have its own style internal state and logic. This search widget would have a lot of reuse in other widgets or composites.

`elements`: Elements are the basic component type. These are the most basic building blocks (e.g. a button component, an input component, etc) of the components and are generally only consumed.

I think creating a structure like this allows for easier discovery of existing components and allows for better hierarchical organization as apps grow in size and complexity, I am confident I am not the only one that has seen a mile long list of components in a flat `components` directory.

###### Basic UX

My basic UX philosphy is influence by the Bauhaus design philosophy, form should follow function. I believe in minimizing clicks, clean clear designs, and intuitive layouts. For this challenge, the application interface is pretty simple and straight forward. 

1) If a user accesses the application and there is no active user/session I will redirect to a login page.

2) Once a user successfully logs in I will direct them to the be main directory browser view

3) An authenticated user will have a top applicaiton bar. This will be the main logo/title and navigation

4) As a user drills down the route will update and a breadcrumb will be populated

5) A user will be able to bookmark any level of drilldown and any sub-path below `/browse` will be parsed into the breadcrumb

6) A user will be able to sort the view on any column 

7) A user will be able to search/filter the current directory for specific files/dirs, file types, and by size breakpoints (e.g. all files with size greater than X, or all files with size less than Y, or all files between size A and size B)

8) if an entry is a `dir` then the user will be able to click on that row and it will initiate an updated query to the api to get the next layer of the directory structure.
