# EIA Ingestion Call Schematic

Call tree for `src/ingestion/eia_api_ingestion.py`.

Legend: `──▶` calls (in order, top to bottom), `┄┄▶` conditional call, `⟳` loop.

## `view_eia_data`

```
view_eia_data
│
├──▶ 1. _check_valid_api_key
│        ├──▶ _build_view_url                      (base URL, no routes)
│        └──▶ _make_url_request
│
├──▶ 2. _get_valid_route_ids
│        ├┄┄▶ return early if route_ids == []
│        ├──▶ _build_view_url                      (full route lineage)
│        ├──▶ _make_url_request                    (fast path: success returns here)
│        └┄┄▶ ⟳ validation walk, only if the fast path failed
│               ├──▶ _build_view_url               (route_ids[:n+1])
│               └──▶ _make_url_request             (stops at first invalid ID)
│
├──▶ 3. _extract_response                          (branches on valid/terminal flags)
│        │   (invalid ID present → uses only the valid sublist of route IDs)
│        │
│        ├┄┄▶ no terminal route
│        │      └──▶ _get_subroutes                   (returns {"subroutes": df})
│        │             ├──▶ _build_view_url
│        │             └──▶ _make_url_request
│        │
│        └┄┄▶ terminal route found
│               └──▶ _extract_metadata                (returns {"base", "facets", "response"})
│                      ├──▶ _build_view_url         (base metadata)
│                      ├──▶ _make_url_request
│                      ├──▶ _build_view_url         (view_data_size=True → row count)
│                      ├──▶ _make_url_request
│                      ├──▶ _build_route_lineage
│                      │
│                      └┄┄▶ if get_subfacet_metadata
│                             └──▶ _extract_facet_metadata   (falls back to all facets if none valid)
│                                    └──▶ ⟳ per valid facet ID
│                                           ├──▶ _build_view_url   (view_facet_metadata=True)
│                                           └──▶ _make_url_request
│
└──▶ 4. display step                               (branches on the response dict)
         ├┄┄▶ "subroutes" in response
         │      └──▶ display(response["subroutes"])
         │
         └┄┄▶ otherwise
                └──▶ _display_metadata
                       ├──▶ _side_by_side  ×4 (general, dates+defaults, frequency, data+facets)
                       └┄┄▶ if facet metadata present
                              ├──▶ _side_by_side  ×1 (subfacet counts)
                              └┄┄▶ if specific facet IDs were requested
                                     └──▶ _side_by_side  ×N (head(5) sample per facet)

(returns the response dict if return_response=True, else None)
```

## `get_eia_data`

```
get_eia_data
│
├──▶ 1. view_eia_data(return_response=True)        (the whole tree above; subfacet metadata NOT requested)
│        └┄┄▶ "subroutes" in result → return None  (invalid or non-terminal route)
│
├──▶ 2. _validate_data_request_params              (pure checks against the metadata dict; no calls)
│
├──▶ 3. _build_data_request_params                 (dict of query params; no calls)
│
├──▶ 4. _build_base_url                            (route path + "/data")
│
├──▶ 5. _make_url_request                          (initial data request, in try/except; reads "total" row count;
│                                                   warns if 0 rows)
│
├──▶ 6. batch decision
│        ├┄┄▶ n_rows <= 5000 or subset_n_rows <= 5000
│        │      └──▶ build the df straight from the initial response
│        │
│        └┄┄▶ else
│               └──▶ _batch_data_request
│                      └──▶ ⟳ per batch (ceil(n_rows / 5000))
│                             └──▶ _retrieve_data
│                                    ├──▶ _build_base_url
│                                    ├──▶ _build_data_request_params
│                                    └──▶ _make_url_request
│                      └──▶ pd.concat of the batch dfs   (batch params = {subset: 5000, offset: 5000 * batch_num})
│
└──▶ 7. output
         ├┄┄▶ write_to_filepath → write_dataframe_to_csv (from common.py)
         └┄┄▶ else → return data_df
```

## Notes

- **Shared leaf utilities:** `_build_view_url` and `_make_url_request` sit at the bottom of nearly every branch. `_build_base_url` and `_build_data_request_params` play the same role on the data side.
- **`_build_get_url`:** it's marked `# DEPRECATE` and nothing calls it any more.
- **Batching ignores user subset/offset:** `_batch_data_request` is passed `data_params` (without `subset`/`offset`), so a user `offset_n_rows` is dropped when batching. Likewise the user's `subset_n_rows` only matters on the non-batched path.
- **Facet validation gap:** `_validate_data_request_params` reads `bmr["facets"]["subfacets"]` when `facets` are given, but `get_eia_data` calls `view_eia_data` without `get_subfacet_metadata`, so `bmr["facets"]` is `None` there. Facet-filtered calls would hit a `TypeError`. The facet ID/value checks are also still `pass` stubs. `bmr_facets` is only defined when `facets` is given, so `sort_by` without `facets` raises a `NameError`.
- **`view_eia_data` return:** the `"subroutes" in response.keys()` display check runs on the response dict, which is why `get_eia_data` can re-use that same dict.
- **Redundant request:** `_extract_metadata` fetches the same base-metadata URL that `_get_valid_route_ids` already fetched on its fast path. Each is a separate HTTP request. This is where a cached response could save a call.
- **Extra `_make_url_request` in `get_eia_data`:** the initial data request in step 5 is not reused when batching. `_batch_data_request` re-requests batch 1 from offset 0.
