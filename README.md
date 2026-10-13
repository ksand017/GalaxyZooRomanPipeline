
# Galaxy Zoo: Roman Pipeline
A pipeline for creating a Zooniverse project from Roman survey images. The pipeline is structured in four stages.
1. Step 1 - Select-Sources: "Scrape" Roman L3 (images) and L4 (catalogs) data products for an input program and filter combination to compile a master catalog of detected sources.
2. Step 2 - Create-Cutouts: Create source cutouts (single band and 3 color composites) with various brightness scalings (arcsinh and MTF) applied.
3. Step 3 - Cutouts-to-Subjects: Create clean multipanel figures for scaled color images and single band images.
4. Step 4 - Subject-Upload: Locally connect to Zooniverse account, create new Zooniverse project and subject set, and batch upload subjects to Zooniverse.

### Step 1 - Select-Sources:
"Scrape" Roman L3 (images) and L4 (catalogs) data products for an input program and filter combination to compile a master catalog of detected sources. 

Searches the Roman AWS L3 file system for L3 data products that match input search criteria. Required inputs include the program number (e.g, 00342 for the SOC HLWAS simulated data) and at least one filter to match on. Additional optional inputs include the plan, pass, and segment number of the observation. See [here](https://roman-docs.stsci.edu/raug/astronomers-proposal-tool-apt/apt-program-elements) for more details on these APT program elements. 

The user can also optionally define the L4 data product (catalog) column names to include in the master catalog. Default columns include ```['ra_centroid', 'dec_centroid','kron_abmag', 'kron_radius', 'fluxfrac_radius_50', 'segment_area', 'is_extended', 'semimajor', 'semiminor']``` for each input filter. E.g., if input filters are ```['f106', 'f129']```, then default columns include ```['ra_centroid_f106', 'dec_centroid_f106','kron_abmag_f106'... 'is_extended_f129', 'semimajor_f129', 'semiminor_f129']```

The list of L3 files identified on the Roman AWS L3 file system are then parsed to extract source catalogs. This stage uses the primary filter (```in_filter```) to search against. For each ```in_filter``` L3 file, the associated L4 catalog and the skycell that the file corresponds to are indentified. The skycell information is used to identify spatially overlapping images for the other input filters (the match filters). If filter-matched files are not available for the given skycell, the filter is marked as 'MISSING.' 

After at least one filter-skycell file is identified, the source metadata is added to the master catalog. In cases where more than one filter returns skycell matches, the sources in the match filters are spatially cross-matched with the sources in the ```in_filter``` L4 catalog before merging them with the master catalog. 

While parsing through the ```in_filter``` L3 files, a tracking list is updated to log which ```in_filter``` L3 files have been crossmatched and scraped for sources. 

An option to re-search for the 'MISSING' L3 files is available. At this stage, if new files are found then an updated master catalog is created and saved.  

### Step 2 - Create-Cutouts:
Create source cutouts (single band and 3 color composites) with various brightness scalings (arcsinh and MTF) applied. 

Filter master catalog in step 1 for sources to make cutouts for. Currently takes first 200 subjects from master catalog. For each selected source, determines an appropriate cutout size based on a balance between source brightness/faintness and concentration. A minimum cutout size of 64×64 $\text{pixels}^2$ (~50 $\text{arcsec}^2$) is imposed. 

For the image cutouts, 2 brightness scalings are applied to individual filter images before creating composite 3 color images. The first scaling is a simple Arcsinh ([Lupton+2004](https://iopscience.iop.org/article/10.1086/382245/pdf)) stretch. The second scaling uses a Midtones Transfer Function (MTF) which works to brighten fainter features by remapping the fainter pixel values to the midpoint of the output image. Some details on the effects of an MTF scaling can be found [here](https://pixinsight.com/doc/legacy/LE/16_histograms/histogram_manipulation/histogram_manipulation.html). The images below provide examples of these scalings. The left show a source with Arcsinh scaling applied, while the right shows a source with MTF scaling applied to its images. 

<img width="256" height="256" alt="4_arcsinh_desaturated" src="https://github.com/user-attachments/assets/708b9087-5274-45d9-8f1f-3ea52ab5f018" />
<img width="256" height="256" alt="4_mtf_desaturated" src="https://github.com/user-attachments/assets/131f52bd-512b-4b0a-af9c-8bdb69295547" />


After applying these two different scalings, two 3 color composite images are created and saved with extensions ```'_arcsinh.png'``` and ```'_mtf.png'```

Desaturated versions of the 3 color composite images are also created to reduce the effects of relatively bright pixels dominating a single band. The desaturated image is given by ```RGBim * wt + a*(1-wt) + a*(1-wt)**2 * RGBim```, where RGBim is the original three color image, a is the mean pixel value across all bands, and wt is a 2D array with values between 0 (dominated by one band) and 1 (not dominated). In simple terms, the new desaturated three color image is ```new color = original color + gray version (mean) + contrast restoration```. Specifically, the terms have these "physical" meanings:

- RGBim * wt: downweight the saturated pixels (should have wt=0)
- a*(1-wt): shifts saturated pixels back towards the average pixel value across all filters (i.e., downweight, but then add the downweighted value to something close to the mean
- a*(1-wt)**2 * RGBim: add back some information about contrast between colors (contained in RGBim) for pixels that are close to/are saturated, with the amount of contrast determined relative to how close to saturated the pixel color is

In addition to the 3 color composite images, single band images are also created during this stage. 

### Step 3 - Cutouts-to-Subjects:
Create clean multipanel figures for scaled color images and single band images.

Individual PNG images are combined into multi-panel presentations. The two 3 color composite images are presented on the same figure (top), and the individual single band images are presented on the same figure (bottom). 
<img width="1559" height="806" alt="4-composite" src="https://github.com/user-attachments/assets/6fb23f25-31cb-461c-b875-85fb85267389" />

<img width="1559" height="1559" alt="4-single_band" src="https://github.com/user-attachments/assets/31612ee1-0939-46e2-9fc3-679d01c60e18" />

### Step 4 - Subject-Upload:
Locally connect to Zooniverse account, create new Zooniverse project and subject set, and batch upload subjects to Zooniverse. An example notebook for a simple upload of subjects from the RRN (or from an externally hosted site) can be found [here](https://github.com/ksand017/RRN-to-Zooniverse-Project-Tutorial).

From the filtered master catalog, creates a catalog of asdf file locations, source catalog metadata (AB mags and kron sizes), PNG file locations+name, and external links. This catalog is used to provide image locations for, and attach metadata to, subject uplaods. 

Users can optionally edit the list of columns to be included in the manifest metadata. The default columns include ```['image1_path', 'image2_path', '!RA', '!DEC', '!desi_link', '!ned_link', '!esasky_link', '#id_str']```

At this stage, the user should have their Zooniverse account login information ready to authenticate with Zooniverse. Required inputs include a project id for the project the user would like to add subjects to, and a name for the new subject set to be created or an existing subject set to be edited. Initiating a subject set upload will prompt the user for their Zooniverse login credentials once more before uploading the subjects in batches.


### What's next?
After uploading a new subject set to your Zooniverse project, you can login to Zooniverse and create a project workflow to link the subjects with. More information on building a Zooniverse workflow can be found [here](https://help.zooniverse.org/getting-started/#workflow-details)
