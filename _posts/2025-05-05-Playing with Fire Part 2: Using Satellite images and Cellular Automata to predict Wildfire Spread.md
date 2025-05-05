---
title: "Playing with Fire Part 2: Using Satellite images and Cellular Automata to predict Wildfire Spread"
date: 2025-05-05
---

<!-- Load MathJax -->
<script type="text/javascript" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>


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
