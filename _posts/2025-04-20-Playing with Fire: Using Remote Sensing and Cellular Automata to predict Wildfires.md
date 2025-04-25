---
title: "Playing with Fire: Using Remote Sensing and Cellular Automata to predict Wildfires"
date: 2025-04-20
---


Wildfires are becoming increasingly frequent and severe — not just in traditionally fire-prone regions, but even in places like my home country Germany. Understanding how wildfires behave is crucial for mitigation and response planning. In this project, I use remote sensing data from the Sentinel-2 satellite to estimate the area affected by a wildfire, and then build a simulation that models how the fire might have spread across the landscape.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/fire_animation_good.gif" alt="Burnt area fire mask" width="500">
</figure>

### Fire Selection and Data Collection

For this project, I chose one of the largest wildfires of 2025 (so far): a February bushfire in remote western Tasmania, Australia, which affected nearly 100,000 hectares. I picked this fire because it was covered in a [press release](https://www.abc.net.au/news/2025-02-18/tasmania-remote-west-bushfires-95000-hectares-burnt/104945100) that included a map of the burned area, along with key metadata such as the ignition date and total area impacted. Another reason this fire was ideal is that, due to its remote location, there were no suppression efforts — making the fire spread easier to model without having to account for human intervention.

The ignition date was reported as February 3rd, 2025, and I confirmed this using VIIRS fire detection data from the Suomi NPP satellite. The first ignition point occurred at 145.05249° longitude and -41.57241° latitude.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/ABC_firemask.png" alt="Burnt area fire mask" width="300">
    <figcaption>Burnt area of the bushfire in Tasmania from February 2025. Image from <a href="https://www.abc.net.au/news/2025-02-18/tasmania-remote-west-bushfires-95000-hectares-burnt/104945100" target="_blank">ABC News Australia</a>.</figcaption>
</figure>

There are several platforms that host satellite data, many of which offer APIs to download data programmatically. I initially explored [NASA's Earthdata catalog](https://www.earthdata.nasa.gov/data/catalog), but ultimately settled on [Google Earth Engine](https://developers.google.com/earth-engine/datasets/catalog), as I specifically needed imagery from ESA’s Sentinel-2 satellite.

Sentinel-2 is ideal for this type of analysis: it has a high revisit frequency over Australia and provides data in 12 spectral bands at a resolution of up to 10–50 meters, which is excellent for burn scar mapping.

So now, it was time to “just” download the data.

*Just.*

During my PhD, I worked extensively with another ESA satellite — the Planck Surveyor — which looked away from Earth to measure cosmic radiation. So, this Earth-facing satellite business should’ve been a breeze, right?

Well, the steepest learning curve of this entire project was not modeling the fire, but wrangling the satellite data. Every image I downloaded seemed to arrive in a different format, and most of my early time was spent installing packages, solving dependencies, and debugging scripts. I experienced what I like to call the *Three Stages of Facing New Technology*:

1. “Why are they using `.hdf` files and not just `.txt` files?”
2. “Oh... `.hdf` files are actually quite handy. Makes sense.”
3. “WHY is this not saved as `.hdf`?!”

Eventually, I managed to automate the download process via Google Earth Engine, apply cloud masks, and clean up the data pipeline. Having all the functions in place to quickly download satellite images that are ready to process, I could finally move on to the fun parts of this project.

## Remote Sensing and Spectral Indices

Now that we’ve got all the satellite data ready, we can dive into the core of the project: using **remote sensing** to detect and quantify fire damage.

Remote sensing involves gathering information about the Earth's surface from a distance—typically using satellites equipped with sensors that record reflected sunlight across various wavelengths. Multispectral satellites like **Sentinel-2** record data in multiple spectral bands, including visible light (red, green, blue) and invisible wavelengths like near-infrared (NIR) and shortwave infrared (SWIR). This opens up two powerful approaches for analyzing the land surface:

- **True-color imagery**, where red, green, and blue bands are combined to form an image similar to what we’d see with our own eyes.
- **Spectral indices**, which calculate the contrast between different wavelengths to reveal patterns invisible to the naked eye.

One popular example is the **Normalized Difference Vegetation Index (NDVI)**. It takes advantage of the fact that healthy vegetation reflects much more NIR radiation than red light. By comparing these two bands, we can map the density and health of vegetation. This is shown on the left.

Similarly, to detect areas affected by fire, we use the **Normalized Burn Ratio (NBR)**. This index compares NIR and SWIR reflectance. Burned vegetation reflects more SWIR and less NIR, making the NBR an effective tool for identifying burn scars. This is shown on the right below.

<div style="display: flex; align-items: flex-start; justify-content: space-between; gap: -30px;">
    <figure>
        <img src="{{ site.baseurl }}/docs/assets/bushfire/NDVI_example.png" alt="NDVI Example" width="400" height="290">
        <figcaption>Reflection of different vegetation types across the visible and near-infrared wavelengths. Image from <a href="https://midopt.com/solutions/color-imaging/ndvi/" target="_blank">midopt.com</a>.
        </figcaption>
    </figure>
    <figure>
        <img src="{{ site.baseurl }}/docs/assets/bushfire/NBR_example.jpg" alt="NBR Example" width="400" height="290">
        <figcaption>Comparison of burnt areas and healthy vegetation reflectance across the spectrum from visible to shortwave infrared. Image from <a href="https://un-spider.org/advisory-support/recommended-practices/recommended-practice-burn-severity/in-detail/normalized-burn-ratio" target="_blank">un-spider.org</a>.</figcaption>
    </figure>
</div>



### Calculating Burnt Area with dNBR

With the pre- and post-fire satellite data in hand, the next step is to calculate the **Normalized Burn Ratio (NBR)** before and after the fire. By comparing these two NBR images, we can pinpoint the areas affected by the fire.

In the first image below, you can see the NBR calculated just **before** the fire (left) and **after** the fire (right). You’ll notice some white patches—those are clouds. Kind of ironic: in cosmology, we launch satellites to get *above* the clouds and avoid atmospheric interference, but in remote sensing, clouds are one of our main obstacles. To reduce their impact, we try to combine multiple satellite captures taken over different days to stitch together a cloud-free view. It’s not always perfect, but for our purposes, this works just fine.

We also **excluded pixels classified as water**. That’s because water can have wildly varying NBR values and is easily mistaken for burned vegetation. Fortunately, Sentinel-2 provides a **Scene Classification Layer (SCL)**, which automatically tags each pixel as land, water, cloud, shadow, etc. We used that to clean the data before proceeding.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/NBR_double.png" alt="NBR of the fire area before and after the fire" width=500> 
    <figcaption>NBR values before (left) and after (right) the fire. Clouds were masked out.</figcaption>
</figure>

To better highlight the fire damage, we calculate the **difference in NBR (dNBR)** between the two images. In the image below, brighter areas (white) indicate no major change, while darker browns to reds show where the fire caused significant vegetation loss.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/dNBR.png" alt="NBR of the fire area before and after the fire" width=500>
    <figcaption>NBR values before (left) and after (right) the fire. Clouds were masked out.</figcaption>
</figure>


The final step is to classify pixels as "burned" based on a dNBR threshold. I went with a threshold of **0.15**, though even thresholds as high as 0.3 gave visually similar results. Then I cleaned the final fire area:
- Removed tiny isolated patches
- Filled in small internal holes
- Smoothed the boundaries for a more realistic shape

Using the cleaned-up dNBR map, we can now **estimate the total burnt area**. By counting the burned pixels and multiplying by the pixel area, we get that roughyl **80.000 ha** of burnt land which aligns well with what was reported in news articles—excluding a small fire area on the right that I didn’t include in my mapping.


# Creating a simulation that models the fire

## Dealing with Coordinate Reference Systems (CRS)

Now for the second major headache of the project: **coordinate reference systems** (CRS). Different satellites don’t always capture Earth’s surface from the same angle, and projecting a curved globe onto a flat screen isn’t as straightforward as it sounds. As a result, satellite datasets often come in **different projections and CRS**.

Thankfully, modern satellite data formats (like GeoTIFFs or NetCDFs) are pretty smart. They’re more than just images—they’re like Python dictionaries: each pixel value comes bundled with metadata that tells you what CRS it uses, what the spatial resolution is, and where on Earth it’s located.

To make everything work together, the first step in preprocessing was to **reproject all data into a common CRS and resolution**. I wrote a function that could take in data at any CRS and spatial resolution and reproject it to match my project’s target system.

In some cases, that meant **resampling**: scaling down higher-resolution imagery to match lower-resolution data, or interpolating larger pixels into smaller ones. For simplicity, I chose to reproject **everything to the coarsest resolution** available across my datasets. That way, I avoided upscaling (which adds no new information) and kept the simulation computationally light.

I also opted to work in a **projected coordinate system** (instead of geographic coordinates like latitude and longitude), so all my maps and arrays were in **Cartesian coordinates**, with units in meters. This made it easier to calculate distances, areas, and apply smoothing algorithms down the line.

In short, this CRS wrangling step was about **getting all geospatial layers into the same shape, resolution, and reference system**—so they could be used side by side without errors or distortions.


## Cellular Automaton for Modelling Wildfires

Now that we’ve got our data preprocessed and aligned, the next step is figuring out how to **simulate the spread of the fire**. There are several modelling approaches out there—ranging from physical to purely statistical—but the one that really caught my attention was the **cellular automaton (CA)** model.

Why CA? Not just because it sounds cool, but because it’s a **semi-empirical** method. That means we can describe fire behavior with some basic, physics-inspired rules, and then **tune the model using actual data**. Since I’m no expert in the detailed chemistry and fluid dynamics of bushfire propagation, this seemed like the perfect fit. I’m much better at coming up with (somewhat) logical rules and translating them into mathematical models.

Here’s how it works: in a cellular automaton, we divide the landscape into a grid of square **cells**, each with a fixed size \( l \). Every cell is in one of several states:

- **Flammable** (e.g., green vegetation)
- **Burning**
- **Burnt**
- **Nonflammable** (e.g., water bodies or bare ground)

At each time step, burning cells can **ignite adjacent flammable cells** with a certain probability $p_\text{burn}$. Once a cell has burned, it transitions into the burnt state and can no longer catch fire.

The key is to define a good $p_\text{burn}$ function—one that might depend not just on adjacency, but on factors like **vegetation type, wind, and moisture**, which we can estimate using our satellite data.

Below is a simplified schematic of the CA grid used in the simulation. Red cells are burning, green are flammable, black are already burnt, and blue are nonflammable (like water).

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/cellular_automaton.png" alt="Simplified image of cellular automaton state" width="400">
    <figcaption> Simplified image of a cellular automaton state with red representing burning cells, green flammable cells, blue nonflammable cells and black burnt cells.</figcaption>
</figure>

## Modelling Fire Spread with NDVI, Wind and Slope

To make the fire simulation more realistic, we need to incorporate **environmental factors** that influence how fires behave in the real world. I selected three variables that I assumed would be most impactful (without doing a deep dive into the literature): **wind**, **elevation/slope**, and **vegetation density**, which I approximated using **NDVI**.

- **Wind** is **time-dependent** and affects how fast and in which direction a fire spreads.  
- **Slope** influences spread by increasing fire speed uphill and slowing it downhill.  
- **NDVI** reflects vegetation density, with the assumption that **higher NDVI leads to faster spread**—a hypothesis we’ll revisit later.

Each cell in the cellular automaton model is assigned values for these three factors. The spread of fire between neighboring cells is then determined by:

$$p_\text{burn} = p_0 \cdot f_\text{NDVI} \cdot f_\text{slope} \cdot f_\text{wind}$$


Where:  
- `p_0` is a base ignition probability,  
- `f_NDVI`, `f_slope`, and `f_wind` are multiplicative factors derived from the cell’s attributes.

The full implementation, including how directional dependencies are handled (like uphill/downhill or wind-aligned spread), is available in the [well-commented Jupyter notebook](link).

### Wind, Slope and Directionality

Both **wind** and **slope** have **directional effects**. That means the probability of spread isn’t the same in all directions. For instance, suppose we have a burning cell that can spread fire either left or right. If the wind is blowing to the right, `f_wind` will be greater for the right neighbor and smaller (or even suppressive) for the left. The same applies to slope—fire spreads more easily uphill and is slowed down going downhill.

This introduced a layer of complexity in the implementation since `f_wind` and `f_slope` need to be computed **relative to the direction of potential spread** rather than as fixed values per cell.

### First Simulation Attempt

Using guessed values for all parameters, I ran the first simulation. The result? A fire that **spreads too easily**. Once ignited, it continues almost endlessly unless it encounters water (blue) or designated nonflammable zones (orange/yellow).

Still, this was a **solid first attempt**. The fire interacts with the spatial environment, and we can already see areas where vegetation and wind conditions accelerate the burn. With some parameter tuning and the addition of other factors like **humidity, temperature**, or **vegetation type**, the model could evolve into something much more realistic.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/fire_animation_bad.gif" alt="Simplified image of cellular automaton state" width="400">
    <figcaption> Fire animation with guessed parameters. Green cells represent flammable cells, darker green means higher vegetation. Blue is water and yellow and orange are nonflammable cells. The timestep is in hours. The wind directions is shown in the bottom left. </figcaption>
</figure>
## Optimising the model

The next step is to find better model parameters. The parameters are the following:

- `p_0`: Base ignition probability  
- `NDVI_low`, `NDVI_mid`, `NDVI_high`: Modifiers for ignition probability in areas with low, medium, and high NDVI  
- `c_slope`, `c_wind`: Coefficients for how strongly slope and wind affect the spread of fire

To optimise these parameters, we need to define what makes a **good simulation**. The perfect simulation reproduces the observed burn scar from satellite data, correctly predicting which pixels burned and which didn't.

### Confusion Matrix

We can use a **confusion matrix** to evaluate performance:

|                          | **Burned in fire data (ground truth)** | **Unburned in fire data** |
|--------------------------|----------------------------------------|----------------------------|
| **Burned in simulation** | True Positive (TP)                     | False Positive (FP)        |
| **Unburned in simulation** | False Negative (FN)                    | True Negative (TN)         |

- **True Positives (TP)**: Pixels burned both in the simulation and in the observed satellite data  
- **False Positives (FP)**: Pixels burned in the simulation but **not** in the satellite data  
- **False Negatives (FN)**: Pixels that burned in the satellite data but were missed by the simulation  
- **True Negatives (TN)**: Pixels that correctly remained unburned in both cases  

We can then calculate the **F1 score**:

$$
F1 = \frac{2 \cdot TP}{2 \cdot TP + FP + FN}
$$

- The **perfect score is 1**.
- The more incorrect predictions we make (either FP or FN), the lower the score.

By varying the model parameters to **maximise the F1 score**, we can find a set of parameters that best match the observed fire behavior.

---

There are heaps of different optimisation algorithms to choose from. I used my own version of **Bayesian Optimisation**. Running a single simulation takes around **2 minutes**, and finding optimal parameters may require **100–200 iterations**. So yes, this can take a while.

To speed things up, I considered:
- Vectorising the simulation steps  
- Updating only the cells near burning ones  

But in the end, I just let the whole thing run **overnight**. Worked like a charm. Let's see how the optimised parameters perform.


## Final Simulation Results

After running the optimisation, let’s take a look at the final model parameters and the resulting simulation.

One key observation is that the **overall base ignition probability (`p_0`) was reduced**. This means that, on average, fire spreads less aggressively across the landscape. The **influence of wind** on the fire spread was **significantly increased**, suggesting that the direction and strength of wind were crucial drivers of fire behavior in this case.

Most surprising to me was the result for **high NDVI areas**. The model assigned a **negative contribution** to ignition probability in these regions — meaning that **densely vegetated areas were less likely to burn** than sparsely or moderately vegetated ones. At first this seemed counterintuitive, but it actually makes sense: 

> Healthy, high-NDVI vegetation is typically moist and less flammable, while low to moderate NDVI may indicate **stressed, dry, or dead vegetation** — which is more prone to ignition.

In the final simulation shown below, the fire spread looks **much more organic**. There are **holes** and **irregularities** in the burned area, which correspond to **natural firebreaks** like rocky outcrops, water bodies, or nonflammable ground cover. Interestingly, the real fire scar from satellite data showed similar patterns before any post-processing or smoothing.

All in all, the optimised simulation captures the spatial dynamics of the fire remarkably well — a satisfying end to this modelling journey.


<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/fire_animation_good.gif" alt="Simplified image of cellular automaton state" width="400">
    <figcaption> Fire animation with optimised parameters. Green cells represent flammable cells, darker green means higher vegetation. Blue is water and yellow and orange are nonflammable cells. The timestep is in hours. The wind directions is shown in the bottom left. </figcaption>
</figure>

## Comparing Simulation to Reality

Now that we have the final simulation, let’s compare it directly with the actual satellite fire scar. The image below shows the **spatial comparison** between the **burnt area predicted by the simulation** and the **observed fire scar** from satellite data.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/evaluation_result.png" alt="Simplified image of cellular automaton state" width="200">
    <figcaption>Comparison of actual burnt area with simulated burnt area. Green represents True Positives, red False Positives, and blue False Negatives.</figcaption>
</figure>

- ✅ **Green cells** represent areas that both the simulation and the satellite data marked as burnt — the **true positives**.
- ❌ **Red cells** are **false positives** — locations where the simulation predicted fire but there was none in the satellite data.
- 🔵 **Blue cells** show **false negatives** — areas that burnt in reality but were missed by the simulation.

Overall, the simulation **overestimated the burnt area by about 15%**, meaning it predicted more spread than what actually occurred. However, this is a pretty strong result considering the simplicity of the model and the limited number of parameters.

I wasn’t entirely sure how well a **cellular automaton model** would perform, or whether the **three chosen environmental factors** (NDVI, slope, and wind) would be sufficient. But the outcome suggests that these are meaningful predictors. That said, **important additional variables** could further improve accuracy — such as:

- **Humidity**
- **Temperature**
- **Precipitation**
- **Vegetation moisture stress**
- **Vegetation type**

In this current version of the model, fires mostly stopped due to **unfavourable wind conditions** and **high NDVI** (dense and likely moist vegetation). Incorporating moisture-related factors explicitly would likely improve realism even further.




## Reflections and Next Steps

Cellular automata proved to be a simple yet effective approach to modeling wildfire spread — especially when detailed knowledge of combustion processes or vegetation behavior is limited. The core logic is easy to implement, and with only a few parameters, the model produced fairly realistic results.

### Improvements and Expansion

The next logical step is to incorporate more environmental data, such as:
- **Humidity**
- **Temperature**
- **Precipitation**
- **Vegetation type or moisture stress**

This would require additional remote sensing work and research into which parameters truly influence fire behavior, beyond NDVI alone. A more advanced model could then be tested against the same fire for comparison.

Ultimately, the goal would be to build a **catalog of fires** — including ignition dates, burnt area masks, and fire durations — which can serve as training data for a more generalizable model. At the moment, the model has only been tuned to one specific fire. As a result, it likely won't generalize well to others.

A more robust approach would involve using a machine learning pipeline:
1. **Train** the model on many fires.
2. **Evaluate** on a separate test set.
3. **Validate** whether the model generalizes well to new fire events.

If successful, the model could be used to:
- Study how **climate change** affects wildfire behavior (e.g., under hotter, drier conditions).
- Predict **fire paths** for use in **preventative burns** or emergency response planning.

### Final Thoughts

This project was a fantastic introduction to **remote sensing**. About 90% of the time was spent framing the problem and acquiring clean data. Once that was done, the fire simulation itself was implemented in just one evening.

While the GUIs of Google Earth Engine and NASA Earth Data were intuitive, working with their **Python APIs** was initially challenging — but worthwhile. I can now access remote sensing datasets directly from Python, which massively improves workflow efficiency.

The cellular automaton model may have worked a bit too well on this one dataset, but that’s also the beauty of semi-empirical models. I’m a big fan of combining simple, interpretable concepts — like wind, slope, and NDVI — and seeing how far they can take you with just one fitting step.

This project was a proof of concept — and a fun one at that.
