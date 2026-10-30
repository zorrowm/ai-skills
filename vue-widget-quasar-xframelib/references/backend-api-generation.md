# Backend API Generation and Calls

Use this workflow to connect a Vue/Quasar/xframelib frontend to backend HTTP services through generated TypeScript methods.

## Supply the Swagger Document

Open the [WebAPI generator](https://zorrowm.github.io/service/#/webapi). Supply the backend's Swagger/OpenAPI JSON using any one of these equivalent methods:

| Input method | Action |
| --- | --- |
| JSON URL | Enter the document URL in **Swagger JSON URL**, such as `http://127.0.0.1:5139/swagger/v1/swagger.json`. Use the actual backend address. |
| JSON file | Use the file picker beside **Load or drag a JSON file** (`加载或拖拽 JSON文件`), or drag one `swagger.json` file into the JSON editor panel. |
| JSON content | Paste the complete contents of `swagger.json` into the large JSON editor panel. |

Selecting or dropping a file fills the same editor used for pasted content. The editor content takes precedence over the URL; clear it before generating from a different URL. A valid document supplied through any one method is sufficient to download the client. If a browser cannot fetch a URL because of CORS, mixed content, or backend reachability, use a locally obtained JSON file or paste its contents instead.

## Configure and Download

1. Set **SysConfig API URL configuration** (`SysConfig API URL 配置`) to the project's runtime service URL expression. The default is `Global.Config.ServiceURL.DefaultWebAPI`. This is a TypeScript property path inserted into the generated file, not the Swagger document URL or a literal HTTP address. For another backend, use its existing `Global.Config.ServiceURL` key.
2. Select **New mode, return res** (`新方式，返回 res`). The page currently defaults to **Old mode, return res.data** (`旧方式，返回 res.data`), so explicitly switch to the new option before downloading.
3. Click **Download API file (.ts)** (`下载 API 文件 (.ts)`). The current generator derives the filename from the document's `info.title`, replaces filename-unsafe characters with underscores, and falls back to `WebAPI.ts` when the title is absent.
4. Save or copy the downloaded `.ts` file into the **frontend application's** `src/api` directory, creating that directory if needed. For example, a document titled `CatalogAPI` produces `src/api/CatalogAPI.ts`. Check the diff when refreshing an existing client and keep caller imports aligned with the final filename.
5. Inspect the generated imports and exports. Preserve the shared `xframelib` request helpers and match the backend's HTTP methods, paths, parameter order, body types, and response types. Use the actual generated method names and signatures rather than inventing them.

The generated module captures its `baseURL` from the configured expression at module initialization. Ensure the frontend loads `public/SysConfig.js` into `Global.Config` before importing the API client. Check the matching service URL and the existing request/authentication configuration before changing runtime settings.

## Call the Generated TypeScript Methods

Import generated exports through the project's existing alias, commonly `@/api/<module>`, in a Page, Widget, or Pinia store. Pass the generated parameters in their declared order, and use the generated request/response types where available. Keep loading state, business rules, and user-facing errors in the caller.

The current generator's **new mode** emits a result with this shape:

```ts
return { response, data: response?.data as GeneratedResponseType };
```

Here, `GeneratedResponseType` represents the response type inferred for that endpoint. The outer result is the generated wrapper: `res.data` is the typed endpoint response body, while `res.response` preserves the underlying request result. Do not assume the whole `res` is the business payload or an Axios response. If the backend body itself contains an envelope, follow that endpoint's schema before reading nested fields or interpreting success/error codes.

For example, **only if** the downloaded `CatalogAPI.ts` exports `getItems(pageIndex?: number, ...)` with an array response body, a caller can use:

```ts
import { getItems } from "@/api/CatalogAPI";

export async function loadItems() {
  const res = await getItems(1);
  return res.data;
}
```

Adapt this example to the downloaded exports and backend response schema. The old mode returns the endpoint body directly, so migrating an existing caller to the new mode requires checking its result handling. Preserve the shared request layer's authentication, headers, timeouts, and error behavior; avoid duplicating generated endpoints with inline `fetch`, standalone Axios clients, or handwritten URL strings in business components.

## Refresh and Verify

- When the backend contract changes, regenerate from the updated JSON using the same new mode and intended service URL expression. Keep custom business logic outside the generated file so regeneration preserves it.
- Review the generated file's imports, exported method names, parameter signatures, paths, and types, then update affected callers. Keep unrelated generated clients and `src/service` RPC output intact.
- Run the frontend project's `pnpm.cmd typecheck` and applicable formatting/lint checks. A skill-only documentation change does not require a frontend build.
- When the backend is available, exercise a suitable read-only endpoint and confirm the configured base URL, HTTP method, path/query/body mapping, authentication, and the `{ response, data }` result consumed by the UI. Check existing error/loading behavior as appropriate; report when runtime verification was unavailable.
