---
title: 'Meatrocket'
description: 'The really cursed method behind the madness'
pubDate: 'Apr 24 2025'
heroImage: ''
---





# Preamble

I wanted to hack this together to share some detail behind what I did for meatrocket temperature predictions, but also to maybe help defend+explain some of the really cursed things I did. 

**This is not the quickest/most-efficient way to get this answer. This was a nice real-world test case of some tools I have been developing to run CFD at home, for hobby uses, on my personal machine.**

Regardless- if you found your way here, I hope you find something in here interesting and/or insightful :)

![pretty picture 1](meat_pretty_1.png)



# What are we trying to do here?
We are trying to predict the temperature of a lump of copper (nosecone tip), as it is being subjected to aero-thermal heating on a rocket flight trajectory. 

To solve the temperature question, we first need to know the aero-heating environment that the nosecone tip going to be subject to. When looking at heating on something like this- the first thing we turn to is our canonical, simple shapes/flows that are well studied. At the nosecone tip, you could get estimates of the stagnation point heating from a littany of sources (Van Driest, Detra, Tauber, Fay-Riddell). More downstream of the nosecone tip, we could use something like flat-plate boundary layer heating relationships/correlations. But we would have to bridge between these two regimes somehow to get the full picture. 

Anyways, I didn't do any of that, and instead just ran some really (computationally) expensive Computational Fluid Dynamics (CFD) to determine the severity of the aero-heating applied to the nosecone. (*I did use the above methods to sanity check my results*)



# CF(D:)
## Background
I am going to assume the reader has a little bit of familiarity with fluid mechanics and CFD for the rest of this... results are below if you want to skip the nerd talk.

Running this sort of high-speed, viscous CFD requires a specific mesh to capture the physics happening far away from the body, as well as the aero-heating/shearing that is happening *really* close to the body (within the "boundary layer"). We typically use an "extruded" mesh to capture the near-body/boundary-layer stuff, and a tetrahedral (in the simplest case) mesh to capture the far-field stuff.

![Hybrid mesh example from cfd_meshman](mesh_red.png)

As I am running this at home, on my personal machine, for "fun", I don't have the really nice+expensive software to make such meshes easily. 

What I do have, is a [GMSH](https://gmsh.info/), [NASA mesh_tools](https://software.nasa.gov/software/MSC-26648-1), vscode, and a fever dream. If you combine those, you get [cfd_meshman](https://github.com/elliottmckee/cfd-meshman) - a free tool that can make hybrid boundary-layer + far-field meshes useful for running the type of CFD you'd want for this sort of thing. It is what I used to make the above image.

I will post in more detail about that workflow at some point. If interested, follow me on [Twitter/X](https://twitter.com/mckee_elliott)

There are a lot of limitations with this tool-chain though. The main one here being that I cannot make 2D meshes nor utilize symmetry planes to take advantage of the very simple axisymmetric nosecone tip. **So I am simulating this in full 3D, which is a tremendous waste of time and computational resources.**


## Geometry
Thankfully though, we only have to simulate a small section of the nosecone tip, and not the full rocket.

This is due to the nature of super-sonic flows (which is the regime of interest, as it is where aero-heating occurs), in which information can generally only travel in one direction*. In other words, what is happening downstream on the rocket (for example, further downstream on the nosecone or body), has no impact on the flow near the tip. (*the caveat here is that information can travel upstream within the **subsonic** boundary layer near the body)

So I ended up simulating something like this:

![Meatrocket nosecone tip geometry](meat_geo.png)

This looks pretty whacky... 

I would do this way differently if I had a more-competent meshing software, but basically- I didn't want to chop the geometry right at the aft/base of the tip, as that would cause a really big separation right near a region I care about. So I added a bit of an aft-body, and gave it a gradual taper so we aren't separating the flow until later on- and when we do, we are doing so more gradually, due to the large aft-radius, which causes less computational shenanigans within the CFD solver (really strong/sharp angle changes like this, without adequete mesh resolution, can lead to instabilities).

## Mesh
Using the aformentioned [cfd_meshman](https://github.com/elliottmckee/cfd-meshman). I generated this:

![cursed mesh](meat_mesh.png)

This is NOT a good mesh. It is a chiefly a product of meshing limitations, and trying to use it to ham-fistedly capture physics I want.

You need really small cells near the surface to resolve the sharp gradients of heating in the boundary layer near the surface (*for those of us in the biz*, this is often more restrictive than the y+<1 criteria often used for resolving momentum components of turbulent boundary layers).

Additionally, you want to have high resolution in the vicinity of shocks. If you have a really big cell that is not aligned with the shock, it tends to non-physically smear the shock. The extruded mesh shown here isn't perfect, as it is aligned to the body, not the shock, but it is certainly better than the tetrahedra, which have random relative orientations.

So, for lack of better options, I simply extruded my boundary layer mesh far out enough that it captured most of the shock that I cared about with adequate resolution, and then just kinda blew out the resolution when we switch to the tetrahedral far field. This sort of cell-volume jump is normally sub-optimal. 

Here's an example showing the shock captured within this extruded boundary layer mesh:
![cursed mesh solution](cursed_mesh2.png)


There are smarter ways to do this, which normally involve a thinner extruded mesh to just capture the boundary layer, and then either fitting the mesh to the shock (*which NASA mesh_tools can do*), grid adaptation, etc.






## Conditions/Trajectory
Now that we have a mesh- we need to know what conditions to simulate. 

We have an expected trajectory (i.e. OpenRocket, or RASAeroII), and we want to run CFD at a minimal number of points (to save on computation time), such that we resolve the heating pulse that we're gonna see in flight. 

You could just run at bunch of different mach numbers, but I am a big ole nerd, so I spaced my simulation conditions out by calculating the Chapman heating*, finding the peak, and then grabbing equal-heating increments on either side of that peak, as shown. (*this is a coarse approximation for stagnation point heating... somewhat of a heuristic really. I think using turbulent flat plate heating would have been a bit better here though)

![cfd conditions along the trajectory](meat_conditions.png)

I also assumed standard atmosphere conditions at each altitude for all these runs.


## Post-processing
I am going to skip most of the CFD-solver/setup specific shenanigans. Partially because I believe in you, but mostly because I am running low on time to write this. Feel free to reach out to me with questions on one of my linked socials.

But I ran these simulations with "cold-wall" boundary conditions. As in, the temperature of the nosecone tip was always 300 Kelvin in the sims. This will come up later. And I also ran everything turbulent.  

In Paraview, I surface-area-weighted-averaged the heating across the actual tip (not including aft-body), to get the following average heating at each of the mach points I simulated, to get something like:

|mach	|qdot avg watts/m^2|
| -------- | ------- |
|0	    |0|
|0.634	|2714|
|1.503	|20188|
|1.883	|40474|
|2.16	|60704|
|2.387	|80637|
|2.583	|100427|
|2.787	|120160|
|2.66	|99788|
|2.493	|79325|
|2.293	|58704|
|2.038	|37660|
|1.674	|16646|
|0.788	|-2866|

<br>



# Temperature Predictions

Ok, that was a lot of work to just get some heating **environments**. How are we translate this into knowing how hot it actually gets?

Answer: MCEN 3012 - Thermodynamics 1

We assume the nosecone tip is a lumped mass. It has some thermal mass (m*Cp). Q = mCp∆T your way to freedom.

By taking the derivative of Q = mCp∆T, assuming a finite time step, and re-arranging, we arrive at 

![how to integrate forward in time](math_1.png)

* T_n+1 is the temperature at the next time step
* T_n is the temperature at the current time step
* mCp is the thermal mass of the nosecone tip
* ∆t is some small time step
* Qdot_tot is your total heat flux [Watts] into the nosecone tip

We can then take this, given an initial temperature, and step forward in time to solve for the temperature along the trajectory. 


But unfortunately, Qdot_tot needs a bit more explaining. 

You could, as a simple cut, just take the "qdot avg" from the above table above, multiply it by the surface area of the tip, and just jam it in there. But to be a bit more accurate, you'd want to perform an extra step of turning it into a heat-transfer coefficient. Remember how we ran the CFD simulations as a "cold-wall"? Doing this step basically allows us to compensate for the fact that the copper tip is going to heat up during flight, causing us to shlorp less heat in from our environment (since heat transfer is proportional to the temperature gradient). This also allows us to properly model the aero-cooling that will happen as you slow back down and the atmosphere becomes colder w/ altitude, relative to the hot copper tip.

I calculated the heat transfer coefficient, h from the above table data using:

![how to heat tranfer-ificate](math_2.png)

* q_avg,cfd is the avg heating from cfd
* Tr is the [recovery temperature](https://www.thermopedia.com/content/291/) of the flow 
* T_cfd is 300K (from the CFD sim setup)


We can now sub in our solved-for temperature as we step along the trajectory, to get the heating at a given timestep

![how to get back to heatflux again](math_3.png)


Additionally, you may want to consider the heat coming out of the nosecone tip via [radiation](https://en.wikipedia.org/wiki/Stefan%E2%80%93Boltzmann_law). 

And you also need to multiply this all by the surface area of the nosecone tip. Here's the final equation, roughly:

![total heating](math_4.png)
* A_tip is the tip external surface area
* qdot_avg_n is the average convective heating at time step n
* sigma and epsilon are the stefan-boltzman constant, and surface emissivity
* T_n is the tip temperature at time step n
* T_atm is the atmospheric temperature at time step n (using as a coarse approximation for radiation sink temperature)



# Results
I am so over-time on how long I thought this would take to write, so just gonna drop the results here. 
![pretty picture 2](meat_pretty_2.png)


## Pre-flight
I assumed a hot Mojave afternoon initial condition.
![pre-flight prediction](flight_vs_preflight_raw.png)



## Post-flight
The same as above, except I changed the initial condition of the simulation to match the Thermocouple initial temperature.
![post-flight prediction](flight_vs_preflight_tuned.png)




