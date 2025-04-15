---
title: "Bushfire Simulations with real data from Tasmania"
date: 2025-04-20
---


## Introduction 

## From Space to Simulation: Modeling a Wildfire in Tasmania 🔥

Wildfires or bushfires are becoming more frequent and intense and even in countries where you would not expect them (such as Germany), they occur more frequently. In this project, I am using satellite data to calculate the affected area of fire and then I build a simulation that recreates the fire spread.

For the fire of interest, I have decided on one of the biggest fires in 2025 (so far). In February almost 100.000 ha were affected by a bushfire in Tasmania, Australia. The choice for this fire is that there was a press release with an image showing the affected area. Furthermore, the size and ignition date of the fire. As a plus, the fire was in a remote area of Tasmania, so there were no efforts made to stop the fire (making it easier to model the spread later).

I am still amazed that all the satellite data are freely available. The first step will be downloading the data. From the news article (https://www.abc.net.au/news/2025-02-18/tasmania-remote-west-bushfires-95000-hectares-burnt/104945100), we know that the fires started on the 3.02.2025. The area of interest is lat, lon. 

There are different websites that host the satellite data. They generally have APIs to download the data directly from Python. At first, I used the Earth data by NASA (https://www.earthdata.nasa.gov/data/catalog), but then settled for Google Earth Engine (link) because they have the data from ESA's Sentinel 2 satellite. Sentinel 2 is a great option for this project because it passes by Australia frequently and records radiation across 12 frequency bands with a resolution of 50m. So, now we "just" need to download the data.  Just. Having only worked with satellites that are facing away from Earth, most of my time on this project was spent trying to collect the correct images using a Python script, installing packages and solving dependencies. I probably learnt roughly 10 new file formats. The learning curve with new technology usually goes through these 3 steps:
1. Why are they using .hdf-files and not just .txt-files
2. Oh .hdf-files are actually quite handy. Makes sense to use them
3. Why is this not saved as .hdf, why would you use .txt for this

 However, once everything was set up, I am now able to get my satellite data of choice in a matter of seconds. It was quite a steep learning curve.

 ### How to map burn scares using remote sensing

 Now that we have all the satellite data at our dispense, we can start with the project. Using remote sensing with multi-band satellites, we will use the data in 2 ways.
 - Explain a bit more about remote sensing
 We either use the data directly. For example, when we combine the channels of red, blue and green light, we get a true colour image. The other method is to calculate the difference between 2 bands. Different kind of vegetation or soil reflects light in a specific way. A very commonly used combination is the normalized difference vegetation index which makes use of the fact that healthy vegetation areas reflect near-infrared (NIR) radiation much better than unhealthily vegetated areas. As shown in the image below, we can then take the difference between red light band and the NIR-band to get information about the vegetation. In similar manner, the normalized burn ratio (NBR) uses the fact that healthy vegetation emits very little shortwave infrared (SWIR) radiation while burnt regions emit strongly in these regions.

-Double Image of NDVI and NBR

The game plan is now to calculate the NBR of the fire area shortly before the fire and then just after the fire. Comparing the two images and their NBR, we can identify the area that has burned in this fire. In the first image below, we can see the area of interest just before the fire on the left and after the burn on the right. There are a couple of white spots coming from clouds that were cut out. Quite funny because in Cosmology and astrophysics, you use satellites to avoid clouds and the Earth's atmosphere while for remote sensing this is a bigger issue. To minimise the effect of clouds, we collect multiple pictures taken at different times and ideally by combining them, we get a cloud-free image. The results we got here are not perfect, but good enough for this project.

Now, we can compare the NBR of the before and after images. We also removed all the pixels that were capturing water. Sentinel-2 data comes with a Szene Classification Layer (SLC) assigning each pixel to water, land, cloud, shadow and more. The NBR of water can vary strongly and can easily be confused as a burnt area, so it is best to remove these pixels in advance. Below, I plotted the NBR before (left) and after (right) the burn.

- double image NBR

It is already quite clear to see the burnt areas. Taking the difference between the and calculating the dNBR, we can see the burnt areas more clearly. Most of the area is fairly white, which means that the NBR did not show significant changes between the two images. Everything that is brown to red highlights the burnt areas as shown in the image below on the left. In the last step, we decide on a dNBR value above which we declare an area as burnt. I went with 0.15, but changing it to 0.2 or even 0.3 didn't have any visual impact. Finally, let's "clean" up the area a bit. I removed tiny patches, filled in tiny holes in the fire, and smoothed out the outline. 

- dNBR

Going back to the image from the news article, I am able to come to similar conclusions about the size and shape of the fire. The total burnt area can be calculated by counting the burnt pixels and multiplying it with the resolution. I get pretty much the same area as in the article (taking into consideration that I did not include the smaller fire on the right).


## Creating a simulation that models the fire

Now on to the second big headache during this project - Coordinate reference systems. As not all satellites record images from the same direction and the projection of the sphere onto a flat surface (laptop screen) is not always unambiguous, satellite data also comes in different references systems. Here is where the new data file types come in handy. They are basically like Python dictionaries that besides the actual data also contain information about the reference system used and more. The main preprocessing for this step is converting the data from different sources into the same reference system. I want to have my simulation setup as simple as possible, so I want my reference system to be in cartesian coordinates meaning with an x and y-axis measured in meters. The preprocessing is mainly bringing all the required data into the right shape.

Speaking of data and the simulation, how are we actually going to model the fire? There are several approaches used in the real world, but the one that sparked my interest the most was Cellular Automata. Not because of the cool-sounding name, but because it is a semi-empirical approach. Semi-empirical approaches are great because we can come up with some ideas on how to model the physics/dynamics of the fire and then use the data to derive the strength of different interactions. As I am no expert on the exact mechanism of how bushfires spread this seems like a good approach. I am much better at coming up with (somewhat) logical-sounding mechanism and put them into mathematical equations.

So back to how cellular automata work. In this approach, the area of interest is divided into cells, in our case, they are squares, of length l. Each cell is either burning, burnt, flammable or nonflammable. Burning cells have a probability p_burn to set neighbouring cells on fire that are flammable. After a cell is burnt, the cell becomes a burnt cell. A simplified set up shown below. Green cells are flammable, red cells are burning, black cells are burnt and blue cells are nonflammable (water for example). The trick is now to choose a good p_burn.
- Simple image of CA
To model a realistic fire, several environmental factors need to be included. I have picked 3 factors that I thought to be the most impactful ones (without really researching). These are the wind speed and direction, the elevation/slope and the vegetation density/NDVI. Slope and NDVI are not time-dependent while the wind is time-dependent. Each cell of the cellular automaton model will have wind, elevation and NDVI assigned to it. Then the general logic is that fire spreads faster in the direction of the wind (and slower opposite to the wind). Fire spreads faster uphill and slower downhill, and higher NDVI means faster spread of the fire (We will see that the last point might not necessarily be true). After running the model and doing a bit more research into the topic, I would also include humidity, temperature, vegetation type and moisture stress in the simulation. But my 3 factors led to a satisfying result. The probability of fire to spread from one cell to another is thus:

p_burn = p_0 * (1 + p_NDVI) * (1 + p_slope) * (1 + p_wind)

The exact details of the implementation can be found in the excellently commented jupyter notebook - link - . Just to highlight the complications with slope and wind is that they change depending on the direction of the fire spread. Let's assume, we have a burning cell that can spread to the right and left, if the wind is blowing to the right, then p_wind on the right is positive, while it is negative on the left. This made the implementation a bit less straightforward.

The biggest surprise was the NDVI. My idea was to split the NDVI into different groups, very dense vegetation, "normal" vegetation and sparse vegetation. I assigned dense vegetation the highest increase in burn probability, normal vegetation a small increase and sparse vegetation has no increase. Doing this led to the fire spreading without stopping as the outer area of the simulation area is all very dense vegetation. I played around a bit with the different parameters to match the burn area, but then decided to let the machines do the work. If I can define a measure of what a good simulation is and what a bad simulation is, then I can simply run a regression model to find the best set of parameters. The measure is simple, I compare the fire area at the end of the simulation to the actual fire area. The more pixel match, the better. So the algorithm will change the parameters such that it will optimise the number of matched pixels.

The parameters to adjust are p_wind, p_slope and how strong each of the different NDVI groups affect the probability. You can choose your favourite algorithm for the optimisation of these parameters. I am choosing Bayesian Optimisation because that's what I used during my PhD and it should work pretty well in this case. I ran the optimisation over the lunch break and the results were quite surprising. The final simulation is shown below.

- simulation

The big surprise was that the burn probability for dense vegetation was slightly negative. This means that the fire very dense regions are the least likely to catch fire (besides nonflammable cells). Looking at the map of the fire, we see that the NDVI in the burnt area is lower than 0.7, so not extremely dense. Outside the burnt area, the vegetation is very high. This tells us that the NDVI might not be the best factor to use. Better would be to use the vegetation type. In Australia, the eucalyptus tree is very common and very flammable while other trees might be less flammable. Also, healthy vegetation with high water content is less flammable than dry vegetation. So, also the humidity plays are more crucial role than I anticipated beforehand.

## What are the learning from this
Despite the suboptimal choice of parameters, we were able to model the data. But how we say in theoretical physics, you can always cook a model that matches the data. Especially because we are only trying to match 1 fire and have 6 parameters to play with. It would be interesting to see, if we choose a different fire around Australia, if the model with the same parameters would also be able to model the fire accurately. I am assuming it will not.

So how can we make this model more general so that it can eventually even predict the spread of future fires? In machine learning terms, we need to train the model on more fires. So, we have to take more fires, let's say 100 fires, we choose a set of our hyperparameters and see how it performs on these 100 fires. Then, we change the parameters until we find the set of parameters that returns the best result across all 100 fires. We take another 20 (that were not included in the 100 fires) and see how well it performs against these 20 fires. This is the basic machine learning pipeline to train models. If we catch all of the major drivers of the fire, then this model should be able to predict fires accurately. 

