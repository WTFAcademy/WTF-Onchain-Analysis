# 23 Creating an Application Using Dune API

## Project Overview

On April 25th, Dune officially opened up API access to all user levels, including free users. Now, even free users can access the Dune API. This tutorial provides a demo that demonstrates how to develop an application around the Dune API.

The demo application is deployed here: [Watcher Demo](https://dev.lentics.xyz/).

The demo application was completed in early March but due to various reasons, the tutorial has been delayed until now. We apologize for the delay. The project code was collaboratively completed by our colleagues George, Ken, and Benny. We extend our thanks to them.

Please note that the above demo application may not work continuously due to API Key limitations. We recommend forking the project and deploying it yourself using your own API Key to further explore.

The user interface of the project is shown in the following image:

![](img/ch23_watcher01.jpg)

## Introduction to Dune API Usage

Dune API is executed and results are obtained based on the Query ID. Each saved Query can automatically be converted into an API access endpoint. In the latest version of the Query Editor interface, you simply need to write the query, test its functionality, save the query, and then click the "API Endpoint" button to obtain the API endpoint URL to access the query results.

```
https://api.dune.com/api/v1/query/2408388/results?api_key=<api_key>
```

This is the simplest way to access the result set of a saved query through the API.

![](img/ch23_watcher02.jpg)

The execution result of a Query is cached by default. If you do not actively execute the Query again, the API endpoint obtained above will provide the cached result of the last execution. Typically, our applications need to actively execute queries to obtain the latest data that meets the conditions, rather than repeatedly obtaining cached result sets. This is especially true for monitoring-type applications. Therefore, we also need to access the "Execute" endpoint to execute the query and the "Status" endpoint to check the status of query execution. After receiving the information that the query has been executed, we can access the "Results" endpoint to obtain the data.

A complete API call process includes executing the query, checking the query execution status, and obtaining the query results. Detailed explanations about Dune API can be found in the API documentation: [Dune API](https://docs.dune.com/api-reference/overview/introductionapi-reference/#latest-results-endpoint).

## Project Requirements

In order to demonstrate the complete process of developing a project using Dune API as comprehensively as possible, the following main requirements points have been organized.

The main function of this application is to provide a pure frontend application for monitoring the creation of new liquidity pools based on the Dune API, specifically for Uniswap V3. It uses a database to save the user's choice of monitored pool addresses and implements caching to avoid making API requests for exactly the same data.

The application includes three main functionalities:

1. **New Pools**
   
    Returns a list of newly created liquidity pools that meet the selected blockchain and date range criteria. When calling the API, the user selects the blockchain name (in all lowercase) and the date parameters (YYYY-MM-DD, converted to specific dates based on user choices).

1. **Latest Swaps**

    Displays the latest 100 swap exchange records for a pool selected from the pool list in a new interface. The user enters this interface by clicking the "Latest Swap" link next to a specific pool from the previous step. When calling the API, the user provides the blockchain name, pool address, and the current hour value (YYYY-MM-DD HH:MI:SS, converted to specific date and hour values, e.g. "2023-02-27 09:00:00"). The API call returns the results, which are displayed in a list format.

3. **Large Swap Alerts**

    Allows the user to input a pool address (prompting the user to copy it from the pool list), and set a threshold for large swaps (e.g.1000USD). The application then calls the API every 5 minutes and generates an on-site alert if there are swap records that meet the specified condition. The user can set the pool address to monitor and the minimum swap amount in USD (currently providing three choices: 1000, 10000, and 100000). If the API returns data, it will be added to the on-site alert. A red badge in the navigation bar will indicate the number of unread alerts, and clicking it will display the list. After clicking on a single alert, it will be marked as read.

## Development environment

Use Node.js 22.12 or newer. The commands below scaffold a Next.js 16 project with
the App Router and Tailwind, then install the Prisma 7 SQLite adapter and the
libraries used in the watcher.

```sh
npx create-next-app@16.4.6 watcher --ts --tailwind --app --use-npm
cd watcher
npm install --save-exact @prisma/client@7.10.1 @prisma/adapter-better-sqlite3@7.10.1 dotenv axios dexie dexie-react-hooks
npm install --save-dev --save-exact prisma@7.10.1
```

## Cache schema

Set `DATABASE_URL="file:./dev.db"` in the project's `.env`. Define the config
and schema as follows to use one cache row per normalized query and parameter set.

```ts
// prisma.config.ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  datasource: { url: env("DATABASE_URL") },
});
```

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "sqlite"
}

model DuneQuery {
  id          String   @id
  executionId String   @map("execution_id")
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

```sh
npx prisma migrate dev --name init
npx prisma generate
npm run dev
```

Initialize the generated Prisma Client on the server using the SQLite adapter.

```ts
// lib/db.ts
import { PrismaBetterSqlite3 } from "@prisma/adapter-better-sqlite3";
import { PrismaClient } from "../generated/prisma/client";

const url = process.env.DATABASE_URL;
if (!url) throw new Error("DATABASE_URL is required");

export const prisma = new PrismaClient({ adapter: new PrismaBetterSqlite3({ url }) });
```

## Query execution and display

Use a server route handler under `app/api/` to call Dune with the server-side
`DUNE_API_KEY`. On a cache miss, use the saved query ID and its parameters to
start an execution, store the returned execution ID, and poll its status.
Fetch results when the execution completes, and stop polling on failure,
cancellation, or timeout. The frontend requests the server route and displays
the returned rows.

When deploying, run `npx prisma migrate deploy` against the persistent
database and `npx prisma generate` before `npm run build`. The SQLite adapter
needs a Node.js runtime and persistent storage.

### Writing SQL Queries for API

API calls and their corresponding query information:

- New Pools: https://dune.com/queries/2056212
- Latest Swap: https://dune.com/queries/2056310
- Alerts: https://dune.com/queries/2056547

### Important Functionality Points

1. Dune API requires executing `Execute Query ID` to obtain its `execution_id` before performing `status/results`. Handle cache expiration properly.
2. The front-end needs to make recursive calls to the system API to retrieve results.

## Dune API Documentation

- [Dune API reference](https://docs.dune.com/api-reference/overview/introduction)
- [Prisma SQLite adapter](https://www.prisma.io/docs/orm/v7/core-concepts/supported-databases/sqlite)

## Project Code Repository

The source code of the project is available here: [Uniswap New Pools Watcher](https://github.com/codingtalent/watcher)

## About Us

`Sixdegree` is a professional onchain data analysis team Our mission is to provide users with accurate onchain data charts, analysis, and insights. We are committed to popularizing onchain data analysis. By building a community and writing tutorials, among other initiatives, we train onchain data analysts, output valuable analysis content, promote the community to build the data layer of the blockchain, and cultivate talents for the broad future of blockchain data applications. Welcome to the community exchange!

- Website: [sixdegree.xyz](https://sixdegree.xyz)
- Email: [contact@sixdegree.xyz](mailto:contact@sixdegree.xyz)
- Twitter: [twitter.com/SixdegreeLab](https://twitter.com/SixdegreeLab)
- Dune: [dune.com/sixdegree](https://dune.com/sixdegree)
- Github: [https://github.com/SixdegreeLab](https://github.com/SixdegreeLab)
