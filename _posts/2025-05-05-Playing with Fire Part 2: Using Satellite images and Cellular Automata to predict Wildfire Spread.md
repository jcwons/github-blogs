---
title: "Playing with Fire Part 2: Using Satellite images and Cellular Automata to predict Wildfire Spread"
date: 2025-05-05
---

<!-- Load MathJax with support for $...$ inline math -->
<script>
window.MathJax = {
  tex: {
    inlineMath: [['$', '$'], ['\\(', '\\)']]
  }
};
</script>
<script type="text/javascript" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>


In the second part of this series on wildfires with remote sensing, I will continue with the analysis of the massive wildfire in Tasmania, Australia. [In the first part](https://jcwons.github.io/github-blogs/2025/04/24/Playing-with-Fire-Part-1-Using-Remote-Sensing-to-calculate-Wildfire-damages.html), we collected the satellite images off the fire and calculated the area of the fire. Now, I will show you how to build a model that simulates how the fire spreads using the NDVI, elevation and wind. 
<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/3D_landscape_layers.png" alt="Simplified image of cellular automaton state" width="600" height="400">
</figure>

## Cellular Automaton for Modelling Wildfires
We will be using **cellular automaton (CA)** model, which is a **semi-empirical** method. That basically means that we develop a model with parameters that are being determined by the **actual data** from part 1. If our model captures all important effects of the fire spread, then we will be able to match the data accurately.

Cellular Automata are actually easy to understand. We divide the landscape into a grid of square **cells**. Each cell can be in one of these 4 states.

- **Flammable** (e.g., green vegetation)
- **Burning**
- **Burnt**
- **Nonflammable** (e.g., water bodies or bare ground)

Below there is a quick illustration of the fire cells, just that in reality, the size of the cells is much smaller than what is shown.
<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/cellular_automaton_illustration.png" alt="Simplified image of cellular automaton state" width="600px">
    <figcaption> Simplified image of a cellular automaton state with red representing burning cells, green flammable cells, blue nonflammable cells and black burnt cells.</figcaption>
</figure>

###  The rules of fire spread

Next, we define the rules of how these cells change in time.

1. Burning cells (red) turn into burnt cells (black) after 1 time step
2. Nonflammable cells (blue) can not catch fire, thus, they remain unchanged
3. Burnt cells remain burnt cells
4. All flammable cells (green) that are adjacent to burning cells can catch fire in the next time step with a probability $p_\text{burn}$.

The rules are quite simple, the tricky part is now to define a burn probability $p_\text{burn}$ that reflects the actual spread of the fire. I decided on using 3 effects, the NDVI, the slope and the wind. Together, I combine them into

### The burn probability

We will work with the following probability that the fire spreads:

$$p_\text{burn} = p_0 \cdot f_\text{NDVI} \cdot f_\text{slope} \cdot f_\text{wind},$$

Where:  
- $p_0$ is a base ignition probability,  
- $f_\text{NDVI}$, $f_\text{slope}$ , and $f_\text{wind}$ are multiplicative factors derived from the cell’s attributes.

For the NDVI, I decided to divide the flammable cells into normal and densely vegetated cells, each having its own value, i.e., $f_\text{NDVI,low}$ and $f_\text{NDVI,dense}$.

The slope compares the altitude of the flammable cell to the burning cell and then calculates the factor in the following way

$$f_\text{slope} = e^{c_\text{slope} \cdot (\text{elevation flammable cell - elevation burning cell})}.$$

If the flammable cell is higher than the burning cell, the chance of catching fire is increased. The parameter $c_\text{slope}$ controls how strong this effect is.

Similarly, for the wind we have
$$f_\text{wind} = e^{c_\text{wind} \cdot (\text{wind speed})},$$
where again $c_\text{wind}$ controls how strong this effect is. For the wind speed, we also need to consider the direction. The wind speed in the equation is actually the wind speed in the direction of the fire spread. If wind is blowing from the burning cell towards the flammable cell, the burn probability is increased, if the wind is blowing the other way, the probability is reduced.

### Finding the right choice for the parameters

Now that we have set up the "physics" of how the fire is spreading, we need to determine the values for the parameters. Let's summarise all the parameters that we have:
- $p_0$: the base ignition probability  
- $f_\text{NDVI,low}$, $f_\text{NDVI,mid}$, $f_\text{NDVI,dense}$: the effects of the vegetation density on the fire spread.
- $c_\text{wind}$, $c_\text{slope}$: the effects of wind and slope on the fire spread.

Now, to evaluate the results of our simulation, we compare it against the actual data of the fire. We can use a **confusion matrix** to evaluate performance:

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

Our goal is now to vary the model parameters to **maximise the F1 score**. This will give us the parameters that describe the underlying fire the most accurately.

During my PhD, I worked extensively on building optimisation routines, so I am excited for this step because I have been wanting to try the Python package 'Optuna' for quite a while. It uses Bayesian Optimisation to search for the best parameter set. Which is an algorithm  to search for the maximum
### The Result
<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/fire_animation_final.gif" alt="Animation of the fire spread" width="300">
</figure>

A surprising result from the best-fit parameters is that the probability for fire to spread towards cells with dense vegetation is lower than for less dense vegetation. I was expecting that the more vegetation, the higher the possibility of fire spreading. Turns out that a high NDVI corresponds to healthy, lush vegetation that contains a lot of water which reduces flammability. A lower NDVI, such as can indicate moisture stress, i.e, vegetation that has been dried out and is easily flammable. 

Let's compare the results of this simulation to the actual affected area. The image below shows the **spatial comparison** between the **burnt area predicted by the simulation** and the **observed fire scar** from satellite data.

<figure>
    <img src="{{ site.baseurl }}/docs/assets/bushfire/evaluation_result.png" alt="F1 Score visualised" width="200">
</figure>

- ✅ **Green cells** represent areas that both the simulation and the satellite data marked as burnt — the **true positives**.
- ❌ **Red cells** are **false positives** — locations where the simulation predicted fire, but there was none in the satellite data.
- 🔵 **Blue cells** show **false negatives** — areas that burnt in reality but were missed by the simulation.
- **White cells** are **true positive** - no fire in the simulation or the satellite images.

Some parts of the fire were predicted quite accurately. The simulated fire stopped roughly where it stopped in reality. However, other parts were not matched properly. This was to be expected. The real fire had several (2 or 3) ignition points, and we did not include important effects such as rain, humidity and temperature. Nonetheless, this is a pretty strong result considering the simplicity of the model and the limited number of parameters.

I would call this a success. We only considered 3 effects, and the results look decent.

### Possible Improvements to the Model

While the results were quite good, there are a million ways to improve on this simple model. First would be to include more/better effects into the model. Here are several factors that could be included

- **Humidity**
- **Temperature**
- **Precipitation**
- **Vegetation moisture stress**
- **Vegetation type**

In the model, fires mostly stopped due to **unfavourable wind conditions** and **high NDVI** (dense and likely moist vegetation). Incorporating moisture-related factors explicitly would likely improve realism even further. In the comparison, there are a couple of patches that didn't burn in the actual fire but almost always burnt in the simulation. That means there is some effect that was not captured.

The speed of the fire spread, i.e., how much distance the fire covers per time step, is something that needs more consideration. Currently, the model spreads 1 cell (250m) per hour. We could use the VIIRS fire detection data to measure how long it took the fire to spread from the ignition point to the outer edge of the fire scar and measure the distance. From this, we could get an indicator of how fast the fire spreads.

Another thing to consider is for how long the fire burns (this probably depends on the NDVI). Currently, the fire always turns off after 1 time step (1 hour). In a more realistic model, a cell would burn for longer than 1 time step with a smaller overall spread possibility. This would mean fire spreads faster if it is supported by the wind. I could spread every time step. While against the wind or without wind, it might still spread, but at a slower rate, maybe every 2-3 time steps.

There are many more ways to improve, but I wanted to highlight these points above.

### Recap and Possible Application
The final goal of such simulations would be to predict the spread of fires either while their burning or to better understand high-risk areas. To get this model to this level of robustness, we need to train it on many many wildfires. To do so, a  **catalogue of fires** — including ignition dates, burnt area masks, and fire durations — would need to be created. The results of the first blog post can be used to build a pipeline that creates this catalogue, given the ignition date and approximate burn area.

At the moment, our model is good at predicting this particular fire, but it will very likely perform much worse on other fires. To get a model that can accurately predict the spread of any fire, we need to train it on many fires. We can follow the usual machine learning methodology.:
0. **Split** the dataset into a train and test dataset.
1. **Train** the model on many fires.
2. **Evaluate** on a separate test set of fires.
3. **Validate** whether the model generalises well to new fire events.

If successful, the model could be used to:
- Study how **climate change** affects wildfire behaviour (e.g., under hotter, drier conditions).
- Predict **fire paths** for use in **preventative burns** or emergency response planning.

Overall, I am really happy with the result of this little project. The progress was really slow in the beginning when gathering data and getting used to the different APIs. Once the data was ready, the rest went quite smoothly without any major hiccups.
