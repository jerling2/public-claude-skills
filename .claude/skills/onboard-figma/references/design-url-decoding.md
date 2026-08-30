# Reference: Figma design URL decoding

Field-by-field breakdown of every Figma URL shape, with an example of each field.

Verified against Figma's docs on 2026-08-30.

## Standard file URL

```
https://www.figma.com/design/abttA1F3GxZdZlkI9t5s99/Example-Figma-file?node-id=32-9
```

| Position | Encodes | Example | Description |
| --- | --- | --- | --- |
| 1 | scheme | `https` | Always HTTPS. A plain HTTP request returns `403` rather than redirecting. |
| 2 | host | `www.figma.com` | `www` normally; `embed.figma.com` for an embedded view; the bare apex also resolves. Does not affect how any other field is read. `api.figma.com` is the REST base, not a file URL. |
| 3 | file type | `design` | Which Figma product the key belongs to. Values enumerated below. |
| 4 | file key | `abttA1F3GxZdZlkI9t5s99` | The durable id, and the value every REST endpoint takes. Case-sensitive. Typically 22 alphanumeric characters, but the length is not guaranteed — read to the next `/` rather than validating a length. Survives renames and folder moves. |
| 5 | file name | `Example-Figma-file` | Slug of the display name, punctuation flattened to hyphens (`[NEW] Marketing Site` → `-NEW--Marketing-Site`). Figma: "The file name doesn't impact the functionality of the URL." Rewritten on rename, lossy, and absent on short links. Never parse it. **If this segment reads `branch`, this is a branch URL** and the key you want sits further along — see below. |
| — | query string | `?node-id=32-9` | Optional. Parameters enumerated below. |

## Branch file URL

```
https://www.figma.com/design/abttA1F3GxZdZlkI9t5s99/branch/9RmMkc4uy0BXlpVkOL6vsG/Example-Figma-file
```

| Position | Encodes | Example |
| --- | --- | --- |
| 1 | scheme | `https` |
| 2 | host | `www.figma.com` |
| 3 | file type | `design` |
| 4 | main file key | `abttA1F3GxZdZlkI9t5s99` |
| 5 | branch marker | `branch` |
| 6 | branch key | `9RmMkc4uy0BXlpVkOL6vsG` |
| 7 | file name | `Example-Figma-file` |
| — | query string | `?node-id=1-2` |

**Send position 6 to the API, not position 4.** File endpoints accept "a file key or branch key" in
the same slot, so a request built from position 4 succeeds and returns the *main* file — wrong
content, no error. `GET /v1/files/:key?branch_data=true` returns `mainFileKey`, which identifies a
branch key after the fact.

## File type values (position 3)

| Value | Encodes | Description |
| --- | --- | --- |
| `design` | Figma Design file | The normal case. |
| `proto` | Design file, prototype view | Same underlying file and same key as `design` — not a reason to request a different link. |
| `board` | FigJam | |
| `slides` | Figma Slides | |
| `deck` | Figma Slides, presentation view | |
| `site` | Figma Sites | |
| `buzz` | Figma Buzz | |
| `make` | Figma Make | |
| `file` | Design file, legacy prefix | Redirects to `/design/`. The key is unchanged. |

## Query parameters

Keyed, not ordered — read them by name, never by position.

| Parameter | Encodes | Example | Description |
| --- | --- | --- | --- |
| `node-id` | the selected layer | `node-id=32-9` | The second value you need. May address a page (`CANVAS`/`PAGE`) rather than a frame, so check the node type before relying on it. Absent means the link names a file only and opens on its first page. |
| `page-id` | the page | `page-id=0-1` | A node id addressing a `CANVAS`. |
| `m` | editor mode | `m=dev` | Opens the file in Dev Mode. A view preference, not part of the file's identity. |
| `embed-host` | the embedding app | `embed-host=example-product` | Required on embed URLs. |
| `t` | sharing-attribution token | `t=AbC123xyz-0` | Undocumented; appended by Share. Strip it before storing or comparing URLs, or one link reads as two. |

## Node id forms

Percent-decode before reading. Converting between the two contexts is a character swap on the
value — `-` → `:` — applied to the whole string, including the instance forms.

| Form | Encodes | Example | Description |
| --- | --- | --- | --- |
| URL, simple | one layer | `32-9` | What `node-id=` carries in a browser URL. |
| URL, inside an instance | one layer within a component instance | `I422-10713;1082-2236` | `I` prefix plus semicolon-separated pairs describing the path down through the instance. Figma does not document this format — treat it as opaque and pass it through. |
| API, simple | one layer | `32:9` | Figma: "if a Figma file URL has the node id `1-3`, you must convert it to `1:3`." |
| API, inside an instance | one layer within a component instance | `I422:10713;1082:2236` | The same swap applied to the opaque form. |
| API, multiple | a set of layers | `ids=1:2,1:3` | Comma-separated. |

## Developer Documentation

- [Guide to files and folders](https://help.figma.com/hc/en-us/articles/1500005554982-Guide-to-files-and-folders) —
  URL anatomy and the full list of file-type prefixes; re-check when a new Figma product ships
- [Plugin API — node `id`](https://developers.figma.com/docs/plugins/api/properties/nodes-id/) —
  where Figma states the hyphen-to-colon rule
- [REST API — File endpoints](https://developers.figma.com/docs/rest-api/file-endpoints/) —
  "a file key or branch key", `branch_data`, and `mainFileKey`
- [Embeds](https://developers.figma.com/docs/embeds/resources) — embed URL parameters and the
  supported path prefixes
