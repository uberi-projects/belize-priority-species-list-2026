# Belize Priority Species List

This is a forked repository from uberi-projects/belize-priority-species-list, with results populated for the Belize Priority Species List 2026, published in the National Biodiversity Monitoring Programme. .gitignore has been updated in this fork to allow results to be commited to the repository. The repository is otherwise untouched. Birdlife range maps, basemaps, and user credentials for ICUN and GBIF will still have to be acquired independently by users of this repository as they cannot be committed. The original  README.md continues from here:

This codebase creates output tables presenting the IUCN redlist status, Belizean regional endangerment status, and Belizean national responsibility (calculated as range share; alpha-hull method) for all Belizean species of focus taxa, including mammals, birds, reptiles, amphibians, sharks and rays, bony fish, insects, mollusks, corals, fungi, and plants. See METHODS.md for details on methodology and supporting literature for the approach used. This codebase also supports the customizable generation of output tables for any input species, provided they exist on GBIF.

## Table of Contents
- [Belize Priority Species List](#belize-priority-species-list)
  - [Table of Contents](#table-of-contents)
  - [Using this Repository](#using-this-repository)
  - [Running the Pipeline](#running-the-pipeline)
    - [Pipeline 1: Full Species Priority List](#pipeline-1-full-species-priority-list)
      - [Quick Run](#quick-run)
      - [Manual Run](#manual-run)
    - [Pipeline 2: Custom Species](#pipeline-2-custom-species)
  - [Redoing Work](#redoing-work)
  - [Coordinate Caching \& Replay Mode](#coordinate-caching--replay-mode)
  - [Repository Structure](#repository-structure)
  - [Data](#data)
    - [Acquiring Data](#acquiring-data)
    - [Pre-Included Data](#pre-included-data)
  - [AI Disclaimer](#ai-disclaimer)
  - [Literature Cited](#literature-cited)

## Using this Repository

Two pipelines are supported in this app. Pipeline 1 generates the entire priority species list for Belize, with some allowable customization, including by-group results for each taxa both filtered and unfiltered to priority species list criteria. Pipeline 2 generates non-filtered results for specific user-provided species.

Before starting, some important **SETUP IS REQUIRED**:
1. Create `.Renviron` in the root and populate it according to `.Renviron.example`.
2. Place required user-supplied files into their appropriate folders. See section "Acquiring Data" below for details.
3. Run `setup.r`
4. Run your preferred pipeline. See "Running the Pipeline" below for details.

## Running the Pipeline

### Pipeline 1: Full Species Priority List

#### Quick Run

`Rscript run_pipeline.r` automatically generates the entire species priority list and GBIF DOI in accordance with METHODS.md. This includes building the candidate pool, loading national and IUCN redlists, building the Belize boundary, computing range shares, retrying transient GBIF fetch failures, building BirdLife-derived range shares for birds ONLY IF BirdLife maps are provided, exports results, and generates a custom GBIF DOI that can be used to cite the GBIF datasets used to calculate the range shares.

If you do not want to create a DOI, set `run_citation_download <- FALSE` near the top of `run_pipeline.r` to skip it.

If you want to generate results for ALL Belizean species, not just IUCN-assessed ones, you may opt in to this step by setting `run_extended_list <- TRUE` near the top of `run_pipeline.r`.

Results may be found in `outputs/results/primary/by_group_all` for unfiltered by-taxon tables, `outputs/results/primary/by_group_filtered` for filtered by-taxon tables, and `outputs/results/extended/by_group` for the extended taxon tables including non-IUCN assessed species of focus taxa.

If you want to further customize your run, or redo a specific step, see "Manual Run," below.

#### Manual Run

Below are scripts that pertain to each major action of the pipeline, as well as any built-in customization options.

**Build the candidate pool and load redlists**: Construct list of candidates for the species priority list and load IUCN and national redlists.
```r
source("R/load_packages.r")
source("R/load_redlist.r")
source("R/load_national_lists.r")
```

**Build the Belize boundary**: Construct the Belize boundary using provided shapefiles.
```r
source("R/build_belize_boundary.r")
```

**Compute alpha-hull range shares**: Calculate range shares in Belize using alpha-hull method. There are two options for this step. Sequential is simpler, and parallel faster for larger taxa:

Sequential, one taxon at a time, can run all taxa in one run. Run:
```r
source("R/load_packages.r")
source("R/load_redlist.r")
source("R/build_belize_boundary.r")
source("R/calculate_w_alphahull_national_lists.r")
```
By default this fetches live GBIF data for every species. To replay from cached coordinates
instead, set `use_cached_coords <- TRUE` near the top of `calculate_w_alphahull_national_lists.r`
before running - see "Coordinate Caching & Replay Mode" below.

Parallel, for large taxa. No need to run the `source(...)` lines above first.
1. Pick a taxon. Valid values: `"Amphibians"`, `"Birds"`, `"Corals"`, `"Fish"`, `"Fungi"`, 
    `"Insects"`, `"Mammals"`, `"Mollusks"`, `"Plants"`, `"Reptiles"`, `"Sharks & Rays"`.
2. Decide how many workers to split the work across (recommendation is 3), and open that many separate
   terminal windows at the repo root.
3. In each terminal, run one line below. Start each one without waiting for the previous to
   finish (e.g. open terminal 1 and run its line, then without closing it open terminal 2 and run
   its line, and so on). The middle number is that terminal's own worker ID (1, 2, 3, ...), and
   the last number is the total worker count, which must be identical in every terminal:
```
Rscript R/calculate_w_alphahull_parallel_worker.r "Birds" 1 3
Rscript R/calculate_w_alphahull_parallel_worker.r "Birds" 2 3
Rscript R/calculate_w_alphahull_parallel_worker.r "Birds" 3 3
```
Add a 4th argument, `TRUE`, to any of these to replay from cached coordinates instead of fetching
live - see "Coordinate Caching & Replay Mode" below.

4. Once all workers for that taxon have finished, merge their results into the taxon's final CSV:
```r
source("R/merge_alphahull_partitions.r")
merge_taxon("Birds")
```
5. Repeat steps 1-4 for any other taxon you want to run in parallel. Any taxon you don't run this
   way still needs to go through the sequential option above - it automatically skips taxa whose
   final CSV already exists, so it's safe to run after parallel runs to pick up the rest.

**(Optional) Retry transient GBIF fetch failures.**: Retry failed fetches from GBIF. To automatically loop:
```r
source("R/load_packages.r")
source("R/retry_gbif_fetch_failures_loop.r")
```

For manual control instead, first build retry input directly:
```r
library(dplyr)
groups <- c("reptiles", "fungi", "amphibians", "mollusks", "corals", "sharks_rays", "mammals", "insects", "birds", "fish", "plants")
failed <- bind_rows(lapply(groups, function(g) {
    read.csv(file.path("outputs/results/primary/by_group_all", paste0(g, ".csv")), colClasses = c(gbif_id = "character")) %>%
        filter(note == "GBIF fetch failed after retries") %>%
        transmute(gbif_id, species, clip, taxon = g)
}))
saveRDS(failed, "outputs/intermediates/primary/by_group/fetch_failed_for_retry.rds")
```
Then retry - sequential (simpler) or parallel (faster for a large retry set):

Sequential:
```
Rscript R/retry_gbif_fetch_failures.r 1 1
```

Parallel, in separate terminal windows:
```
Rscript R/retry_gbif_fetch_failures.r 1 3
Rscript R/retry_gbif_fetch_failures.r 2 3
Rscript R/retry_gbif_fetch_failures.r 3 3
```

Then merge - required after either option above, sequential or parallel:
```r
source("R/load_packages.r")
source("R/merge_fetch_retry_results.r")
```

**(Optional) BirdLife comparison weights for birds**: Add BirdLife comparison weights, as a second methodology for calculating range share. Needs `birdlife_ranges/BOTW_2025.gpkg`
acquired (see "Acquiring Data"):
```r
source("R/load_packages.r")
source("R/load_redlist.r")
source("R/calculate_w_birdlife.r")
```

**Export the final CSVs**: This exports a clean, usable set of csvs as results. Results may be found in `outputs/results/primary/by_group_all` for unfiltered by-taxon tables, and `outputs/results/primary/by_group_filtered` for filtered by-taxon tables.
```r
source("R/load_packages.r")
source("R/export_alpha_results_by_group.r")
```

**(Optional) Submit a GBIF citation download.**: Produces a citable GBIF download DOI
covering the datasets used.
```r
source("R/build_citation_taxon_keys.r")
source("R/load_packages.r")
source("R/submit_gbif_citation_download.r")
```
The download key is saved to `outputs/results/citation/gbif_citation_download_key.rds`. Re-running this script submits a brand new download
(there's no cache/skip check).

**(Optional) Discover species missing from the IUCN-based candidate pool.**: This generates range share results for every species GBIF has Belize
records for across the focus taxa (Fish excluded - it already draws its own checklist from FishBase, not IUCN) including non-IUCN-assessed species. Alpha-hull range share is only computed for species that clear a record-share screen first (`screen_gbif_checklist_gaps.r`), to save on computation. `outputs/intermediates/extended/gbif_checklist_screen_results.rds` shows all results and whether they pass screening. `outputs/results/extended/by_group` shows range share results for species that pass screening.

```r
source("R/load_packages.r")
source("R/load_redlist.r")
source("R/check_gbif_checklist_gaps.r") # size gap
source("R/screen_gbif_checklist_gaps.r") # screen missing species
source("R/retry_gbif_checklist_screen.r") # if applicable, retry failed species
source("R/calculate_w_alphahull_checklist_gaps.r") # compute range shares
source("R/export_alpha_extended_by_group.r") # export results
```

### Pipeline 2: Custom Species

`R/calculate_w_alphahull_custom_species.r` computes results for a user-supplied species, or a short list of species, that you specify directly. It otherwise functions like Pipeline 1, just filtered to only the supplied species.

First, edit the `custom_species` list near the top of the script according to inline usage notes at the variable. Then run:
```r
source("R/load_packages.r")
source("R/build_belize_boundary.r")
source("R/calculate_w_alphahull_custom_species.r")
```
Results are written to `outputs/results/custom/custom_species_results.csv`, one row per species,
with `iucn_category` (global) and `belize_ranking`. A second, formatted version is written alongside it to
`outputs/results/custom/custom_species_results_formatted.csv`.

By default this also submits a GBIF download covering just this run's species and gets a citable
DOI for the data used. Set `request_citation_doi <- FALSE` before running to skip this step. Saves download key to
`outputs/results/citation/custom_species_citation_download_key.rds`.

## Redoing Work

Most scripts cache their results to `.rds` files in `outputs/` and automatically reuse them
instead of recomputing. This makes the pipeline resumable after a stop or crash, but it also
means re-running a script after changing an input won't redo work that's already cached. To force
a step to redo, delete the relevant cache file(s) first, then re-run the step.

<table>
  <thead>
    <tr>
      <th>To Redo...</th>
      <th>Delete...</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>The IUCN Belize redlist fetch</td>
      <td><code>outputs/intermediates/redlist/belize_redlist_noDD.rds</code></td>
    </tr>
    <tr>
      <td>Taxonomy resolution in <code>load_redlist.r</code></td>
      <td><code>outputs/intermediates/redlist/batches/</code> (all files, or just the batch(es) covering the species you want re-resolved)</td>
    </tr>
    <tr>
      <td>FishBase data</td>
      <td><code>outputs/intermediates/fishbase/fb_countries.rds</code>, <code>fb_species.rds</code>, <code>fb_belize_species.rds</code></td>
    </tr>
    <tr>
      <td>GBIF taxon-key resolution in <code>load_national_lists.r</code></td>
      <td><code>outputs/intermediates/national_lists/national_lists_taxonomy.rds</code> and <code>national_lists_taxonomy_fallback.rds</code></td>
    </tr>
    <tr>
      <td>A taxon's alpha-hull computation, entirely</td>
      <td><code>outputs/results/primary/by_group_all/&lt;slug&gt;.csv</code> and, in <code>outputs/intermediates/primary/by_group/</code>, <code>&lt;slug&gt;_cache.rds</code> and any <code>&lt;slug&gt;_cache_part*.rds</code></td>
    </tr>
    <tr>
      <td>...and also force a live GBIF re-fetch rather than a coordinate replay</td>
      <td>additionally delete <code>outputs/intermediates/primary/raw_coordinates/&lt;slug&gt;.rds</code> and any <code>&lt;slug&gt;_part*.rds</code></td>
    </tr>
    <tr>
      <td>Just some species within a taxon (keep the rest cached)</td>
      <td>remove those species' rows from <code>&lt;slug&gt;_cache.rds</code> / <code>&lt;slug&gt;_cache_part*.rds</code></td>
    </tr>
    <tr>
      <td>GBIF retry results</td>
      <td><code>fetch_retry_part*.rds</code> in <code>outputs/intermediates/primary/by_group/</code></td>
    </tr>
    <tr>
      <td>Common-name lookups</td>
      <td><code>outputs/intermediates/primary/vernacular_names_cache.rds</code></td>
    </tr>
    <tr>
      <td>Extended-list missing-species checklist</td>
      <td><code>outputs/intermediates/extended/gbif_checklist_missing_species.rds</code></td>
    </tr>
    <tr>
      <td>Extended-list screening results</td>
      <td><code>outputs/intermediates/extended/gbif_checklist_screen_results.rds</code> (and <code>outputs/intermediates/extended/gbif_checklist_retry_results.rds</code>, if a retry is in progress)</td>
    </tr>
    <tr>
      <td>Extended-list alpha-hull weights</td>
      <td><code>outputs/intermediates/extended/gbif_checklist_gap_weights_alpha.rds</code></td>
    </tr>
    <tr>
      <td>A custom species' result (or all of them)</td>
      <td>remove its row(s) (or the whole file) at <code>outputs/results/custom/custom_species_results.csv</code></td>
    </tr>
    <tr>
      <td>...and also force a live GBIF re-fetch for custom species rather than a coordinate replay</td>
      <td>additionally delete <code>outputs/intermediates/primary/raw_coordinates/custom_species.rds</code> (shared across all custom species, not per-taxon)</td>
    </tr>
  </tbody>
</table>

`<slug>` is the taxon name, lowercased with spaces/symbols replaced by underscores (e.g. "Sharks &
Rays" -> `sharks_rays`).

## Coordinate Caching & Replay Mode

Raw GBIF coordinate points are cached upon fetch for each species, separately from the
summary results, in `outputs/intermediates/primary/raw_coordinates/` (`<slug>.rds` for the sequential
run, `<slug>_part<id>.rds` per parallel worker until `merge_taxon()` combines them). This backs
two run modes:

- **Fetch mode** (default): calls GBIF live for every species that isn't already done. Numbers
  may drift slightly from a previous run as GBIF's database grows.
- **Replay mode**: for any species with cached coordinates, skips the network call and reuses them
  directly - faster, and gives an exact, network-free reproduction of a previous run's numbers.
  Useful for testing a late-stage parameter change (an alpha-hull setting or clip rule) without
  re-fetching GBIF, or for verifying published numbers without relying on GBIF's live, growing
  database. Turn it on with `use_cached_coords <- TRUE` - for sequential, in `R/calculate_w_alphahull_national_lists.r`; for parallel, a 4th CLI argument, `TRUE`. This is also supported for the extended GBIF run beyond IUCN-assessed species, and for Pipeline 2 (custom species) - though custom species cache to their own shared `raw_coordinates/custom_species.rds` rather than a per-taxon file, since they aren't necessarily one of the pipeline's known taxa.

**Caveat**: both caches only track *whether* a species has been processed, not *what parameters*
it was processed under. If you change an alpha-hull parameter or clip rule (in
`define_alphahull_helpers.r`) and want that reflected, delete the relevant cache file(s) first (see
"Redoing Work" above) - otherwise a re-run, in either mode, just replays or skips the old result.

## Repository Structure

**`basemap/`** is an empty folder which is intended to hold user-provided shapefiles.

**`birdlife_ranges/` is an empty folder which is intended to hold user-provided BirdLife species ranges data.

**`data/`** contains reference data, such as the table of nationally threated species

**`outputs/`** is gitignored, like `basemap/` and `birdlife_ranges/` - only the folder structure
(via `.gitkeep`) is tracked, not the generated data itself. It's where everything the scripts
produce lands, split into two trees:
- **`outputs/intermediates/`** - working objects (`.rds` caches/checkpoints) organized by pipeline
  stage: `redlist/`, `fishbase/`, `national_lists/`, `primary/` (main alpha-hull computation,
  including `raw_coordinates/` and per-taxon `by_group/` checkpoints), `extended/`. Machinery for
  resuming or rebuilding a run, not meant to be read directly.
- **`outputs/results/`** - the human-facing deliverables, organized by category:
  `primary/by_group_all` (unfiltered per-taxon CSVs), `primary/by_group_filtered` (the final
  priority list), `extended/` (the extended-list companion CSVs), `custom/` (Pipeline 2 output),
  `citation/` (taxon-key list and download-key files for the citable DOIs). Note
  `primary/by_group_all` is read back in by a few later steps (export, retry, citation key
  building) as well as being a result in its own right.

**`R/`** contains R code to create the Belize Priority Species List
1. `load_packages.r` installs and attaches required packages.
2. `load_redlist.r` fetches RedList data from IUCN, fetches taxonomic data, links data to GBIF species, and filters to desired taxonomic groups, using FishBase for fish taxonomic data. Saves results in batches to `outputs/`.
3. `load_national_lists.r` merges the national/global priority-species source lists (`data/`) with a direct IUCN pull into one candidate table with resolved GBIF taxon keys.
4. `build_belize_boundary.r` builds the combined Belize political and maritime boundary and the equal-area projection (mollweide_crs).
5. `define_alphahull_helpers.r` holds shared candidate-pool and per-species alpha-hull logic used by the two runners below. Not meant to be run directly.
6. `calculate_w_alphahull_national_lists.r` runs the alpha-hull range-share calculation taxon by taxon, single-threaded, checkpointed.
7. `calculate_w_alphahull_parallel_worker.r` is the same calculation split across parallel processes, for the largest taxa.
8. `merge_alphahull_partitions.r` combines parallel workers' results into each taxon's final CSV.
9. `retry_gbif_fetch_failures.r` retries species that hit a transient GBIF fetch failure. See `retry_gbif_fetch_failures_loop.r` below for the version actively used, as this is a single pass worker.
10. `merge_fetch_retry_results.r` merges one retry pass's results back into the per-taxon CSVs.
11. `retry_gbif_fetch_failures_loop.r` loops the retry-and-merge cycle above until either nothing is left flagged as a fetch failure, or a pass recovers nothing new (some species genuinely have no usable GBIF data - that's a stop condition, not a bug). This is the one `run_pipeline.r` actually calls.
12. `calculate_w_birdlife.r` computes a second, independent range-share weight for birds from BirdLife's range maps. See "Acquiring Data" for how to get `BOTW_2025.gpkg`. Range is the union of every BirdLife polygon for a species where presence is "Extant" or "Probably Extant," origin is "Native" or "Reintroduced," and seasonal is "Resident," "Breeding," or "Non-breeding."
13. `export_alpha_results_by_group.r` builds the final, reader-facing per-taxon CSVs.
14. `build_citation_taxon_keys.r` builds the deduplicated GBIF taxon-key list, across all 11 taxa, that the citation download covers.
15. `submit_gbif_citation_download.r` submits a GBIF occurrence download covering the species pool used, for a citable DOI, and polls until it's ready.
16. `check_gbif_checklist_gaps.r` sizes how many Belize-occurring species (per GBIF) are missing from the IUCN-based candidate pool, for the focus taxa (excludes Fish). Diagnostic only.
17. `screen_gbif_checklist_gaps.r` builds the full missing-species list and screens each by Belize record share, to flag which are worth a full range-share computation.
18. `retry_gbif_checklist_screen.r` retries missing species that hit a count-probe failure during screening.
19. `calculate_w_alphahull_checklist_gaps.r` computes alpha-hull range shares for missing species that cleared the screen.
20. `export_alpha_extended_by_group.r` exports the extended-list results as a companion set of per-taxon CSVs, plus a combined file of genuine >=20% discoveries.
21. `calculate_w_alphahull_custom_species.r` computes alpha-hull range share, global IUCN status, and Belize national status for a user-specified species or short list of species - works standalone on a fresh clone, without the IUCN-based candidate pool. See "Pipeline 2: Custom Species" above.

**`renv/`** manages pinned package versions (`renv.lock`).

**`/`** (repo root) holds `README.md`, `setup.r` (installs/restores packages), `run_pipeline.r` (runs the standard pipeline on default settings), `DESCRIPTION` (declares dependencies for `renv`), `renv.lock`, `.Renviron.example`, and `LICENSE.txt`.


## Data

### Acquiring Data

Several datasets need to be acquired and placed in the appropriate repository folders before the scripts will work.

1. **`Belize_Basemap.shp`** (+ `.dbf`, `.prj`, `.shx`, `.shp.xml` sidecars) district-level
   shapefile (Meerman, 2013). http://www.biodiversity.bz. Place in `basemap/`.

2. **`land_belize.txt`** national land-boundary polygon (ArcGIS_DemoBz, 2023a). https://services.arcgis.com/lkHhAsiwtcKnwSeA/ArcGIS/rest/services/Belize_Country/FeatureServer/1/query?where=1=1&outFields=*&f=geojson. Place in `basemap/`.

3. **`maritime_belize.txt`** Belize's declared maritime jurisdiction, Territorial Sea +
   Exclusive Economic Zone (ArcGIS_DemoBz, 2023b). https://services.arcgis.com/lkHhAsiwtcKnwSeA/ArcGIS/rest/services/Belize_Nationalwaters/FeatureServer/0/query?where=1=1&outFields=*&f=geojson. Place in `basemap/`.

4. **`BOTW_2025.gpkg`** bird species distribution maps, used for BirdLife-based range-share
   weights (BirdLife International, 2025). Submit a data request at 
   http://datazone.birdlife.org/species/requestdis. Place in `birdlife_ranges/`.


### Pre-Included Data

1. **`Belize_Threatened_Species_Table.csv`**  list of species previously assessed as vulnerable  in Belize from Belize Forest Department (Belize Forest Department, 2025; Belize Forest Department Wildlife Programme, 2020).


## AI Disclaimer

The AI model Sonnet 5.5 (Anthropic) was used as a tool through the Claude Code for VS Code extension to assist in code development, bugtesting, and documentation of this codebase. However, the authors of this repository take full responsibility for the quality and accuracy of the codebase, and do not defer reponsibility of errors to the AI model in any way.


## Literature Cited

ArcGIS_DemoBz. (2023a). *Belize_Country [Feature layer]* [Dataset]. ArcGIS Online. https://www.arcgis.com/home/item.html?id=09608485ef21491d9ea15a9a5f83fe20

ArcGIS_DemoBz. (2023b). *Belize_Nationalwaters [Feature layer]* [Dataset]. ArcGIS Online.

Belize Forest Department. (2025). *Belize National Red List of Threatened Species: Mammals, Birds, Reptiles and Amphibians*.

Belize Forest Department Wildlife Programme. (2020). *National IUCN Red List for Threatened Avian Species—Belize*.

BirdLife International and Handbook of the Birds of the World. (2025). *Bird species distribution maps of the world* (Version 2025.2) [Dataset]. http://datazone.birdlife.org/species/requestdis

Meerman, J. C. (2013). *Belize Basemap* [Dataset]. Biodiversity and Environmental Resource Data System of Belize. http://www.biodiversity.bz/
