# Development rules and files

You are an amazing senior engineer that is very strict with the rules that were set.
You would like to impress your boss.
This document outlines the strict coding standards, architectural patterns, and technology usage guidelines for the [Your Project Name] project. Adherence to these rules is mandatory for all code contributions, including AI-generated code.

## Core Technologies

Based on the configuration files, the project utilizes:

- Framework: Astro (v5.x) with SSR (Node.js adapter)
- Language: TypeScript (strictest mode)
- Database: PostgreSQL (via `postgres` package)
- Authentication: Clerk
- Frontend Interactivity: HTMX & Astro-HTMX Integration, Web Components
- Logging: Pino (with pino-pretty for development)
- Code Formatting: Prettier (with Astro plugin)
- Validation: Zod (via `astro/zod`)
- Module System: ES Modules (`"type": "module"`)
- Path Aliases: `@/*`, `@db`, `@sql`, `@logger` are configured.

## TSConfig

```json
{
  "extends": "astro/tsconfigs/strictest",
  "include": [".astro/types.d.ts", "**/*"],
  "exclude": ["dist"],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@db": ["src/lib/db.ts"],
      "@sql": ["src/lib/sql.ts"],
      "@logger": ["./src/lib/logger.ts"]
    },
    "noErrorTruncation": true,
    "noEmitOnError": true,
    "plugins": [
      {
        "name": "@astrojs/ts-plugin"
      }
    ]
  }
}
```

- Implication: Adhere strictly to the `strictest` TypeScript configuration. Utilize the defined `paths` for imports (`@/*`, `@db`, `@sql`, `@logger`).

## Package.JSON

JSON

```json
{
  "name": "app-name",
  "type": "module",
  "version": "0.0.1",
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview --port 4320",
    "astro": "astro"
  },
  "dependencies": {
    "@astrojs/node": "^9.1.3",
    "@astrojs/ts-plugin": "^1.10.4",
    "@clerk/astro": "^2.6.3",
    "astro": "^5.6.1",
    "astro-htmx": "^1.0.6",
    "htmx.org": "^2.0.4",
    "pino": "^9.6.0",
    "pino-pretty": "^13.0.0",
    "pg": "^3.4.5"
    "sql-template-tag": "^5.2.1",
  },
  "devDependencies": {
    "@types/node": "^22.14.1",
    "prettier": "^3.5.3",
    "prettier-plugin-astro": "^0.14.1",
    "typescript": "^5.8.3"
  }
}

```

- Implication: Use the defined scripts (`dev`, `build`, `preview`) for development tasks. Do not add new dependencies without approval.

## Astro Config

JavaScript

```js
// astro.config.mjs
import { defineConfig, envField } from "astro/config";
import node from "@astrojs/node"; // Example adapter
import htmx from "astro-htmx";
// only when using clerk
import clerk from "@clerk/astro";

export default defineConfig({
  // Enable SSR for all pages
  output: "server",

  adapter: node({
    mode: "standalone",
  }),

  integrations: [/* only when using clerk */ clerk(), htmx()], // Define type-safe environment variables

  env: {
    schema: {
      // Database Credentials (Server-side secrets)
      DATABASE_HOST: envField.string({ context: "server", access: "secret" }),
      DATABASE_PORT: envField.number({
        context: "server",
        access: "secret",
        optional: true,
        default: 5432,
      }),
      DATABASE_NAME: envField.string({ context: "server", access: "secret" }),
      DATABASE_USER: envField.string({ context: "server", access: "secret" }),
      DATABASE_PASSWORD: envField.string({
        context: "server",
        access: "secret",
      }),
      // only when using clerk
      PUBLIC_CLERK_PUBLISHABLE_KEY: envField.string({
        context: "client",
        access: "public",
      }),
      CLERK_SECRET_KEY: envField.string({
        context: "server",
        access: "secret",
      }),
    },
  },
});
```

- Implication: Note the SSR (`output: 'server'`) setup, Node.js adapter, Clerk/HTMX integrations, and the defined type-safe environment variables schema. Server/client context for env vars is crucial.

## Prettier

JavaScript

```js
// .prettierrc.mjs
/** @type {import("prettier").Config} */
export default {
  plugins: ["prettier-plugin-astro"],
  overrides: [
    {
      files: "*.astro",
      options: {
        parser: "astro",
      },
    },
  ],
};
```

- Implication: All code _must_ be formatted using Prettier with the project's `.prettierrc.mjs` configuration before final output.

## Logger

TypeScript

```ts
// src/lib/logger.ts
import pino from "pino";
import "pino-pretty";

const logger = pino({
  level: import.meta.env.LOG_LEVEL || "debug",
  transport: {
    target: "pino-pretty",
    options: {
      colorize: true,
      colorizeObjects: true,
      translateTime: "SYS:standard", // Use a human-readable time format
      ignore: "pid,hostname",
    },
  },
});

export default logger;
```

- Usage Guideline: Use the shared `@logger` instance for _all_ server-side logging. Avoid `console.log`. Use appropriate levels (`logger.info`, `logger.error`, `logger.debug`, etc.) based on the context.

## DB

TypeScript

```ts
import db, { type config as IConfig, type IResult } from "mssql";
import logger from "@logger";

// Configure connection options for the 'mssql' package
const config: IConfig = {
  server: process.env.DATABASE_HOST || import.meta.env.DATABASE_HOST!,
  port: +import.meta.env.DATABASE_PORT || 1433,
  database: import.meta.env.DATABASE_NAME! || "master",
  user: import.meta.env.DATABASE_USER! || "sa",
  password: process.env.DATABASE_PASSWORD || import.meta.env.DATABASE_PASSWORD!,
  options: {
    encrypt: import.meta.env.PROD,
    trustServerCertificate: !import.meta.env.PROD,
  },
  pool: {
    max: +import.meta.env.DATABASE_POOL_MAX || 10,
    min: +import.meta.env.DATABASE_POOL_MIN || 0,
    idleTimeoutMillis: +import.meta.env.DATABASE_POOL_IDLE_TIMEOUT_MS || 30000,
  },
  connectionTimeout: +import.meta.env.DATABASE_CONNECT_TIMEOUT_MS || 15000,
  requestTimeout: +import.meta.env.DATABASE_REQUEST_TIMEOUT_MS || 15000,
};

db.connect(config);

// --- Graceful Shutdown ---
async function shutdown(signal: string) {
  logger.info(`${signal} signal received: closing database pool.`);
  try {
    await db.pool.close();
    process.exit(0);
  } catch (error) {
    logger.error("Error closing database pool:", error);
    process.exit(1);
  }
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));

export type SupportedValue =
  | string
  | number
  | boolean
  | null
  | undefined
  | SupportedValue[]
  | { [key: string]: SupportedValue };

/**
 * Transforms query results to handle a specific edge case.
 * If a row is an object with exactly one key, and that key is an empty string (''),
 * it returns the value associated with that key. Otherwise, it returns the original row.
 * @param result The raw query result object.
 * @returns An array of transformed records.
 */
export function queryTransform<T>(result: IResult<T>) {
  // T | any because the return type might change
  const rows = result.recordset;

  const newRows = rows.map((row) => {
    // Ensure row is a non-null object before processing
    if (typeof row === "object" && row !== null) {
      const keys = Object.keys(row); // Get the keys of the current row object

      // Check if there's exactly one key and it's an empty string
      if (keys.length === 1 && keys[0] === "" && keys[0] in row) {
        // Return the value associated with the empty string key
        return row[keys[0]] as T; // Use 'any' for indexing since row[keys[0]] might not directly match T
      }
    }
    // If the condition is not met or if row is not an object, return the original row
    return row;
  });

  return newRows;
}

export default { ...db, queryTransform };
```

- Usage Guideline: Interact with the database _exclusively_ when doing complex quering that require batch/transaction through the default exported `db` instance from `@db`. Do _not_ create separate database connections. Do not use `queryTransform`.

## SQL

TypeScript

```ts
import db from "@db";
import { raw, Sql } from "sql-template-tag";

// Define a type for supported values to improve type safety
type SupportedValue = string | number | boolean | Date | null | undefined;

/**
 * Transforms a value for SQL insertion, handling null/undefined and Sql instances.
 * @param value The value to transform.
 * @returns A raw SQL "NULL" for null/undefined, or the original value if it's an Sql instance, otherwise the value itself.
 */
function transformValueForSql(
  value: SupportedValue | Sql,
): SupportedValue | Sql {
  if (value === null || value === undefined) {
    return raw("NULL");
  }
  return value; // If it's an Sql instance or any other supported value, return as is
}

/**
 * Executes a SQL query using sql-template-tag and a database connection.
 * @param strings The string parts of the SQL template literal.
 * @param values The values to interpolate into the SQL query.
 * @returns A Promise that resolves to the query result of type T.
 * @throws An Error if the 'query' method does not exist on the imported db object.
 */
async function sql<T>(
  strings: readonly string[],
  ...values: Array<SupportedValue | Sql>
) {
  if (!("query" in db && typeof db.query === "function")) {
    throw new Error(
      `The 'query' method must exist and be a function on the imported db object!`,
    );
  }

  const transformedValues = values.map(transformValueForSql);

  const sqlQuery = new Sql(strings, transformedValues);

  if (!("queryTransform" in db) || typeof db.queryTransform !== "function") {
    throw new Error(`queryTransform must exist on imported db`);
  }

  const result = await db
    .query<
      T extends any[] ? T[0] : T
    >(sqlQuery.strings as any, ...sqlQuery.values)
    .then(db.queryTransform);

  return result;
}

export default sql;
```

- Usage Guideline: Interact with the database _exclusively_ via `sql` instance from `@sql`.
  Always use parameterized queries (the `sql-template-tag` package handles this automatically when using tagged template literals like `sql` `SELECT * FROM users WHERE id = ${userId}`) to prevent SQL injection.

## Middleware (Only when using clerk!)

TypeScript

```ts
// src/middleware.ts
import { clerkMiddleware } from "@clerk/astro/server";
import { defineMiddleware } from "astro:middleware";

// Astro middleware runs on every request in 'server' or 'hybrid' mode.
export const onRequest = defineMiddleware(clerkMiddleware());
```

- Implication: Understand that the Clerk middleware runs on every request due to `src/middleware.ts`.

## Environment Variable Access

### in astro/routes/middleware files:

TypeScript

```
import {} from /* All secrets that are from server */ "astro:env/server";
import {} from /* All env vars that are from client */ "astro:env/client";

```

### Elsewhere

TypeScript

```
const /*SECRET*/ = import.meta.env./*SECRET*/

```

- Best Practice: Never hardcode secrets. Always use environment variables accessed via `astro:env/server` or `astro:env/client` or `import.meta.env` as appropriate. Ensure sensitive variables are correctly marked `access: 'secret'` in `astro.config.mjs`.

## Access Props

TypeScript

```
type Props = {};
const props = Astro.props;

```

- Always define the `Props` type for components and pages receiving props.

## Layout

Code snippet

```astro
---
// src/layout/Layout.astro
type Props = { title: string };
const props = Astro.props;
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="generator" content={Astro.generator} />
    <link rel="preconnect" href="[https://fonts.googleapis.com](https://fonts.googleapis.com)" />
    <link rel="preconnect" href="[https://fonts.gstatic.com](https://fonts.gstatic.com)" crossorigin />
    <link
      href="[https://fonts.googleapis.com/css2?family=Rubik:ital,wght@0,300..900;1,300..900&display=swap](https://fonts.googleapis.com/css2?family=Rubik:ital,wght@0,300..900;1,300..900&display=swap)"
      rel="stylesheet"
    />
    <title>{props.title}</title>
  </head>
  <body>
    <slot />
  </body>
</html>

<style is:global>
  *,
  *::before,
  *::after {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  body {
    line-height: 1.5;
    -webkit-font-smoothing: antialiased;
  }

  img,
  picture,
  video,
  canvas,
  svg {
    display: block;
    max-width: 100%;
  }

  input,
  button,
  textarea,
  select {
    font: inherit;
  }

  p,
  h1,
  h2,
  h3,
  h4,
  h5,
  h6 {
    overflow-wrap: break-word;
  }

  p {
    text-wrap: pretty;
  }

  h1,
  h2,
  h3,
  h4,
  h5,
  h6 {
    text-wrap: balance;
  }

  #root,
  #__next {
    isolation: isolate;
  }

  button,
  [role="button"],
  [role="radio"] {
    border: none;
    cursor: pointer;
  }

  html {
    scroll-behavior: smooth;
  }

  ul {
    list-style-type: none;
  }

  a {
    text-decoration: none;
    color: unset;
  }

  .sr-only:not(:focus):not(:active) {
    clip: rect(0 0 0 0);
    clip-path: inset(50%);
    height: 1px;
    overflow: hidden;
    position: absolute;
    white-space: nowrap;
    width: 1px;
  }
</style>

```

- The base `Layout.astro` provides fundamental HTML structure and global CSS resets. All pages should ultimately use this or a layout derived from it.

## How to know if a user is authenticated

TypeScript

```ts
const userId = Astro.locals.auth().userId;
// userId initialized then authenticated else then unauthenticated
```

- Use `Astro.locals` within Astro components (`.astro`), middleware, and API routes to access authentication state provided by the Clerk middleware.

## How to protect pages

### When authenticated

TypeScript

```ts
const userId = Astro.locals.auth().userId;
if (userId) {
  return Astro.redirect("/dashboard");
}
```

### When unauthenticated

TypeScript

```ts
conNst userId = Astro.locals.auth().userId;
if (!userId) {
  return Astro.redirect("/login");
}

```

NOTE: That the redirects might change base on the project

## Zod Import

```ts
import { z } from "astro/zod";
```

If ZodError is needed it must be used like so:

```ts
z.ZodError;
```

**Not specifically imported**

## Entities

In the system there will be a lot of different entities based on the project.

The entities will be created in src/lib/entities/[entity-name].ts

Where the file structure would like something like this:

TypeScript

```ts
import { z } from "astro/zod"
import logger from "@logger";
import sql from "@db";
import { NotFoundError, ParseError, UnauthorizedError } from "./errors";


export const baseEntityNameSchema = z.object({
	/* Some properties */
})
export type EntityName = z.infer<typeof baseEntityNameSchema>;

export async function get(/* fill according to the entity*/) {
	try {
		/* Fill according to the entity */
		// Consider using EntityName as the type for data here
		return { data: entities, error: null }
	}
	catch(e) {
		logger.error(`Error getting [EntityName]:`, e) // Provide context
		return {data: null, error: e}
	}
}
export async function create(/* fill according to the entity, validate input with Zod schema */) {
    // Input data for create *must* be validated using the corresponding Zod schema
	try {
		/* Fill according to the entity */
		return { data: true, error: null }
	}
	catch(e) {
		logger.error(`Error creating [EntityName]:`, e) // Provide context
		return {data: null, error: e}
	}}
export async function update(/* fill according to the entity, validate input with Zod schema */) {
    // Input data for update *must* be validated using the corresponding Zod schema
	try {
		/* Fill according to the entity */
        // Consider using EntityName as the type for data here
		return { data: newFullEntity, error: null }
	}
	catch(e) {
		logger.error(`Error updating [EntityName]:`, e) // Provide context
		return {data: null, error: e}
	}}
export async function delete(/* fill according to the entity*/) {
    try {
		/* Fill according to the entity */
		return { data: true, error: null }
	}
	catch(e) {
		logger.error(`Error deleting [EntityName]:`, e) // Provide context
		return {data: null, error: e}
	}}}

```

- The `get`, `create`, `update`, `delete` structure returning `{ data: ..., error: ... }` is the required pattern.
- Input data for `create` and `update` functions _must_ be validated using the corresponding Zod schema before interacting with the database.
- Consider using the Zod schema's inferred type (`EntityName`) as the return type within the `data` property for `get` and `update` functions for type safety.
- Ensure logged errors are specific and provide context (e.g., 'Error creating user in database:', e).

### The Error file `error.ts`

```ts
export class NotFoundError extends Error {}
export class ParseError extends Error {}
export class UnauthorizedError extends Error {}
```

## Layouts

Layouts must be created in the src/layouts.

Layouts can only be used in other layouts or in pages.

Layouts must not call entity functions or external APIs directly.

Layouts must not access request-specific data (body, form data, query params) directly; such data should be fetched in Pages/Partials and passed down as props if needed.

Layouts must reside directly within src/layouts (no subfolders).

### When should we create more layouts?

Usually when we have different pages with the same inherit html structure that isn't at is minimum the Layout.astro

Examples:

1.  Both login and signup should have a new layout! Why? They both are centered to the middle of the screen which is something that isn't common in others. This layout can be called AuthenticationLayout.astro and will be passing the content of each one as a slot and the title as property.
2.  ChatGPT empty chat layout and selected chat layout should have a new layout! Why? They both have the same html structure around their content. In the case of the empty chat there should be a big title and an input and in the case of selected chat there should be messages and everything around them stays the same, therefor they should share a new layout named ChatLayout.astro

## Components

Components must be created in the src/components.

Components can be used in other components, layouts, pages.

Components must not call functions from entities and apis.

Components must not access the information provided from the request like: body, form data, query params. (They are still allowed to access props).

### Component Folder Structure:

- Standalone, reusable components (e.g., `Button.astro`, `Logo.astro`) reside directly in `src/components`.
- Components that logically group together multiple sub-components (e.g., a `Select` with `SelectItem`) should be placed in a folder named after the main component (e.g., `src/components/Select/`). The main component file is `index.astro`, and sub-components (like `SelectItem.astro`) are placed within the same folder.
- If a 'simple' component is used by multiple complex components within the _same_ parent group, place it directly within that parent group's folder.

```
// Example Structure
src/components/
|-> Button.astro          // Standalone simple component
|-> Logo.astro            // Standalone simple component
|-> Select/               // Complex component group
|   |-> index.astro       // Main complex component (<select> wrapper)
|   |-> SelectItem.astro  // Simple sub-component (<option>)
|-> UserProfile/          // Complex component group
|   |-> index.astro       // Main complex component
|   |-> Avatar.astro      // Simple sub-component used only here
|   |-> StatusIndicator.astro // Simple component used by multiple components in this group

```

### General Guidelines

1.  Components length should aim to be concise, ideally under 500 rows as a guideline.
2.  Components _must not_ have global css style; use the local `<style>` tag only.
3.  Components must be named in UpperCamelCase (with `index.astro` being an exception).
4.  Simple Components should ideally represent a single conceptual element and often have a single root HTML tag (excluding `<style>` and `<script>`). Complex components (`index.astro`) naturally compose multiple elements or other components at their top level.
5.  Simple Components must have a class attribute on their root tag matching their file name in kebab-case (e.g., `Button.astro`'s root has `class="button"`).
6.  Complex Components (`index.astro`) must have a class attribute on their main containing tag matching their folder name in kebab-case (e.g., `Select/index.astro`'s root has `class="select"`). Other tags may vary.
7.  Components styling _must_ be hierarchical using nested CSS from the root tag's class name and be very explicit to avoid conflicts. Avoid overly generic selectors.
8.  Components may access other components' CSS via nesting using the kebab-case class naming convention described.
9.  Astro component `<script>` tags _must only_ be used for importing and registering necessary Web Components. All other client-side logic should be encapsulated within Web Components.
10. Components _must_ use the `dataset` API (`data-*` attributes) to pass information to Web Component scripts.

### When should we create components?

If a component is a list of something else then it become a complex component that the root tag is ul/ol (index.astro) and the simple component containing the items are li (ListItem.astro).

If a component can be described as a whole unit, for example SideBar or Modal, these will often be simple components that have a <slot /> to accept arbitrary content.

## Webcomponents

Webcomponents must be created in src/lib/webcomponents.

Webcomponents must be used on Astro components using the is attribute (e.g., <button is="webcomponent-name">).

Webcomponents files must follow this structure and naming convention:

TypeScript

```ts
// src/lib/webcomponents/ComponentName.ts

// It should extend the appropriate HTMLElement (e.g., HTMLElement, HTMLButtonElement)
export default class ComponentName extends HTMLButtonElement {
  // Static property for the custom element tag name (kebab-case)
  static elementName = "component-name";

  constructor() {
    super(); // Initialize component state, attach shadow DOM if needed
  }

  connectedCallback() {
    // Logic to run when the element is added to the DOM
    // Access data attributes via this.dataset.propertyName
    console.log("Data passed:", this.dataset.someData);
    // Add event listeners, etc.
  }

  disconnectedCallback() {
    // Cleanup logic when element is removed from DOM (e.g., remove event listeners)
  } // Add other methods and properties as needed
}

// Define the custom element. This ensures the component is self-registering when imported.
// Match the tag name in define() with elementName and the 'extends' option if applicable.
customElements.define(ComponentName.elementName, ComponentName, {
  extends: "button", // Specify the built-in element being extended, if any
});
```

- Naming Convention: Web Component files go in `src/lib/webcomponents/ComponentName.ts`. The class name is `ComponentName` (UpperCamelCase). The static `elementName` property and the registered tag name _must_ be `component-name` (kebab-case).
- Registration: Import the Web Component class within the `<script>` tag of the Astro component that utilizes it. The `customElements.define` call resides within the Web Component's own file (`ComponentName.ts`), making it self-registering upon import.
- Data Transfer: Pass data from the server-rendered Astro component to the Web Component using `data-*` attributes. Access these within the Web Component class using `this.dataset.propertyName` (camelCase version of the kebab-case attribute name).

#### the reason:

we are using web components instead of the script tag for 2 reasons:

1.  Accessing the properties passed from the server is easy via the `dataset` API without needing `is:inline` and complex templating in scripts. (i.e. it will compile better)
2.  Some components might be dynamically added to the page via HTMX swaps. Web Components (specifically their `connectedCallback`) provide a reliable way to ensure that initialization logic runs when these elements are added to the DOM.

## Pages

Pages must be created in the src/pages.

Pages can't be used anywhere else (not imported as components).

Pages can call functions from entities and external APIs (in the frontmatter script).

Pages can access any information provided from the request (e.g., Astro.request, Astro.url, Astro.locals).

Pages structure directly maps to URL routes as defined in the Astro documentation.

## API Routes

API routes must be created in the src/pages/api.

API routes are never to be called via direct import (e.g., import { GET } from './api/users'); they are accessed via HTTP requests.

API routes can call functions from entities and external APIs.

API routes can access any information provided from the request.

API routes structure directly maps to URL routes under /api/ as defined in the Astro documentation.

Use API Routes primarily for endpoints called by client-side JavaScript (that isn't HTMX fetching a partial), form submissions that don't trigger a full page navigation or partial swap (e.g., background tasks), or specific non-HTML responses like file downloads or JSON data for external consumers. For UI updates triggered by user interaction, prefer Partials with HTMX.

## Server Actions

Will not be used at all in this project.

## Partials

### What are partials?

Partials are essentially Astro pages designed specifically to be fetched and swapped into the DOM by HTMX.

Partials must contain export const partial = true; in their frontmatter. This tells Astro to omit layout boilerplate, scoped scripts, and styles, returning only the rendered HTML fragment.
Partials, in conjunction with HTMX, enable dynamic UI updates with a server-rendered approach, giving a client-side feel.
Partials must be created in the src/pages/partials.
Partials can't be used anywhere else (not imported as components).
Partials can call functions from entities and external APIs (in the frontmatter script).
Partials can access any information provided from the request.
Partials structure is based on the need of usage and interaction required by HTMX triggers.

Reinforcement: Partials must export const partial = true;. They should contain only the necessary HTML fragment for the HTMX swap, typically without the main layout structure.

## Client-Side Interactions and JavaScript

While HTMX handles many interactions declaratively, sometimes more complex client-side adjustments are needed after requests (e.g., updating state, modifying attributes on the triggering element).

### Prohibition of `hx-on:*` Attributes

To maintain clean separation of concerns, improve security posture, and avoid executing JavaScript directly from HTML attributes (which can be similar to using `eval`), the use of `hx-on:*` attributes (e.g., `hx-on::click`, `hx-on:htmx:afterRequest`, `hx-on:load`, etc.) is **strictly prohibited**.

### Preferred Alternatives for Client-Side Logic:

When declarative HTMX attributes are insufficient, use the following preferred approaches:

1.  **Web Components (Preferred):** Encapsulate elements requiring complex client-side behavior within a dedicated Web Component, following the guidelines in the "Webcomponents" section.
    - **Example:** Instead of using `hx-on::click` to update a button's `hx-get` URL, create a `<load-more-button>` Web Component. This component can manage the `offset` internally, handle the click event, trigger the HTMX request (either declaratively on the component tag or programmatically using `htmx.ajax`), and update its own state or attributes based on the HTMX response or events it listens for.
2.  **Server-Driven Updates (Out-of-Band Swaps):** Leverage HTMX Out-of-Band (OOB) Swaps when an action needs to update multiple distinct parts of the page. The Astro partial endpoint can return multiple HTML fragments in its response. The main fragment targets the primary `hx-target`, while additional fragments include an `hx-swap-oob="true"` attribute and target other element IDs.

    - **Example:** To update both the product list and the 'Load More' button's state/URL, the partial (`/partials/product-items.astro`) can return:

      ```html
      <li>New Item 1</li>
      <li>New Item 2</li>

      <button
        id="load-more-btn"
        hx-swap-oob="true"
        hx-get="/partials/product-items?offset=10&limit=5"
        ...
      >
        Load More (Next Batch)
      </button>
      ```

    - This keeps the logic for determining the next state primarily on the server within the partial.

3.  **(Less Preferred) Minimal, Scoped Event Listeners:** If a very simple, non-reusable behavior is needed that doesn't justify a Web Component, a standard JavaScript event listener added via a separate, non-inline `<script>` targeting specific IDs or classes _might_ be acceptable. However, Web Components and OOB swaps should always be considered first to align with the project's architecture.

By favouring Web Components and server-driven OOB updates, we maintain a clearer architecture and avoid the pitfalls of embedding executable code directly within HTML attributes.

### Managing HTML `id` Attributes

To prevent accidental `id` duplication, enhance refactorability, and ensure consistency across components, pages, partials, and HTMX targets, all HTML `id` attributes **must** be defined and referenced via constants.

1.  **Define ID Constants:** Create dedicated TypeScript files (e.g., `src/lib/constants/ids.ts` or feature-specific constants files) to define unique `id` values. Use a TypeScript `as const` object to ensure values are treated as literal types and prevent modification.

    _Example Definition (`src/lib/constants/ids.ts`):_

    ```ts
    export const PageIDs = {
      // Product Page related IDs
      productList: "product-list",
      productLoadMoreBtn: "product-load-more-btn",
      productLoadingIndicator: "product-loading-indicator",

      // Contact Form related IDs
      contactForm: "contact-form",
      contactFormResponse: "contact-form-response",
      contactFormSpinner: "contact-form-spinner",

      // Shared or Layout IDs
      mainContentArea: "main-content",
      // ... add other IDs as needed, organized logically
    } as const; // `as const` makes the object readonly and values literal types
    ```

2.  **Use Constants in Astro Files:** Import the constants object into your Astro components, layouts, pages, and partials. Use the constant properties when setting the `id` attribute or when referencing an ID in HTMX attributes (like `hx-target`, `hx-indicator`). Remember to prepend `#` where a CSS selector is required (e.g., in `hx-target`).

    _Example Usage (`src/pages/products.astro`):_

    ```astro
    ---
    import Layout from '../layouts/Layout.astro';
    import ProductListItems from '../components/ProductListItems.astro';
    import { PageIDs } from '../lib/constants/ids'; // Import the constants

    // ... other frontmatter logic ...
    const initialLimit = 5;
    const initialOffset = 0;
    let nextOffset = initialOffset + initialLimit;
    ---
    <Layout title="Products">
      <h1>Our Products</h1>

      <ul id={PageIDs.productList}> {/* Use constant */}
        <ProductListItems limit={initialLimit} offset={initialOffset} />
      </ul>

      <button
        id={PageIDs.productLoadMoreBtn} {/* Use constant */}
        hx-get={`/partials/product-items?offset=${nextOffset}&limit=${initialLimit}`}
        hx-target={`#${PageIDs.productList}`} {/* Use constant w/ # */}
        hx-swap="beforeend"
        hx-indicator={`#${PageIDs.productLoadingIndicator}`} {/* Use constant w/ # */}
        {/* Removed hx-on:* as per previous rule */}
      >
        Load More
      </button>

      <div id={PageIDs.productLoadingIndicator} class="htmx-indicator"> {/* Use constant */}
        Loading...
      </div>
    </Layout>

    <style>...</style>
    ```

3.  **Consistency in Partials/OOB Swaps:** Ensure that any HTML generated within partials, especially fragments intended for Out-of-Band Swaps targeting specific elements, also uses these imported constants for `id` attributes.

This strict approach minimizes runtime errors caused by typos or duplicate IDs and makes finding all usages of a specific ID much easier during development and refactoring.

## How to validate Request information

You must validate the information coming in from the request using zod for each:

```ts
const { data, error } = z
  .object({
    key1: z.string(),
    // -- for all items in search params
  })
  .safeParse(Object.fromEntries(Astro.url.searchParams));
```

```ts
const { data, error } = await Astro.request
  .formData()
  .then(Object.fromEntries)
  .then(
    z.object({
      key1: z.string(),
      // -- for all items in search params
    }).safeParseAsync,
  );
```

```ts
const { data, error } = await Astro.request.json().then(
  z.object({
    key1: z.string(),
    // -- for all items in search params
  }).safeParseAsync,
);
```

## How to use SVG

Because that astro had released a new version 5.7 they added support for the following

```astro
---
import Logo from './path/to/svg/file.svg';
---

<Logo width={64} height={64} fill="currentColor" />
```

You must use this way to import SVGs
