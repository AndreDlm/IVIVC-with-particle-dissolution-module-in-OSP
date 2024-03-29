# IVIVC with the particle dissolution module implemented in OSP

## Introduction

This repository explains the technical steps for conducting an in vitro-in vivo correlation (IVIVC) for oral drug formulations with the Open Systems Pharmacology (OSP) software suite. Specifically, dissolution kinetics is described by a modified form of the Noyes-Whitney equation [[1](#References)] which accounts for different sizes of spherical particles with a predefined particle size distribution [[2,3](#References)]. In PK-Sim<sup>®</sup>, this dissolution function is readily available in the `Formulation` building block section as `Particle Dissolution` [[4](#References)]. This particle dissolution function has also been implemented in MoBi<sup>®</sup> to model dissolution in biorelevant media *in vitro* and the MoBi<sup>®</sup> file is distributed with this repository.

## Necessary files

To conduct a PBPK-based IVIVC according to the workflow described below, the folder `Input_files` available in this repository needs to be downloaded. This folder contains the following files which should be saved locally in the working directory:

* `particle_distribution_raw_data.csv`
* `psd calculation.R` 
* `in vitro dissolution model.mbp3`
* `DissolutionData.xlsx`
* `BB_Administration_ParticleDissolution_10Bins_toJSON.json`
* `BB_Formulation_ParticleDissolution_10Bins_toJSON.json`
* `JSON.R`



## Workflow

The workflow for establishing IVIVC with OSP comprises the following three consecutive steps:

1. Fitting a cumulative distribution function to a distribution of measured particle sizes using R
2. Fitting the particle dissolution function to *in vitro* dissolution profiles measured in biorelevant media using MoBi<sup>®</sup>
3. Transferring the particle size distribution from step 1 and parameters of the dissolution function from step 2 to PK-Sim<sup>®</sup> for predicting dissolution in the gastrointestinal tract *in vivo*

Step 1 will be exclusively conducted in R; step 2 exclusively in MoBi<sup>®</sup>; and step 3 in R and PK-Sim<sup>®</sup>.

The technical workflow for each of these steps will be explained in detail in the following sections.

### 1. Fitting a cumulative distribution function to measured particle sizes

The particle size of a drug substance or product, typically measured by laser diffraction spectroscopy, is an important physical attribute potentially affecting drug product performance. The measured sizes are generally discretized in specific size ranges or bins. Together with the relative amount of each size bin, they constitute the particle size distribution of a drug substance or product. The relative amount of a particle size bin is quantified in terms of the probability density (also termed q<sub>3</sub> in this context) or the quantile (also termed Q<sub>3</sub> in this context). The particle dissolution model implemented in MoBi<sup>®</sup> and PK-Sim<sup>®</sup> can handle up to 10 different particle size bins. 

In this step, different cumulative distribution functions are fitted to measured particle size data using R. The best fitting function (i.e. with the lowest squared error) is used to discretize the particle sizes in a pre-defined number of bins (per default 10 bins). Finally, the particle radius (i.e. size divided by 2) and the respective density for each radius (termed `rel_amountFactor`) are written in a Excel file which will be imported as `Paramater Start Values` in MoBi<sup>® </sup> in step 2 of this workflow.

The following files are used in this step:

* `particle_distribution_raw_data.csv`
* `psd calculation.R` 

#### Detailed descriptions and instructions:

`particle_distribution_raw_data.csv` is the input file containing the measured particle sizes in column `x_obs` and the respective cumulative distribution Q<sub>3</sub> [%] in column `q_obs` (i.e. the cumulative percentage of particles smaller than or equal to the given size). The values can be changed, but the column names should be kept unchanged. Here, the following hypothetical data were used:

| q_obs | x_obs |
| ----- | ----- |
| 0.23  | 0.7   |
| 0.96  | 0.9   |
| 1.85  | 1.1   |
| 3.13  | 1.3   |
| 4.52  | 1.5   |
| 8.96  | 1.8   |
| 13.92 | 2.2   |
| 21.04 | 2.6   |
| 28.73 | 3.1   |
| 39.02 | 3.7   |
| 46.62 | 4.3   |
| 57.50 | 5.0   |
| 68.48 | 6.0   |
| 78.15 | 7.5   |
| 86.35 | 9.0   |
| 91.35 | 10.5  |
| 95.44 | 12.5  |
| 97.26 | 15.0  |
| 98.91 | 18.0  |
| 99.54 | 21.0  |
| 100   | 25.0  |

`psd calculation.R` is an R script used to fit different cumulative distribution functions to the measured particle size data. The script contains the following sections:

* `FUNCTIONS`: defines the distribution functions fitted to the measured particle size data. Currently, these functions comprise the Weibull distribution, lognormal distribution, normal distribution and gamma distribution function. Additional functions can also be added; in this case the R script (especially the wrapper functions) needs to be adjusted.
* `MEASURED PARTICLE SIZE DATA`: defines the following items:
  * `workingDir` defines the working directory 
  * compound name, batch name and unit of the particle size (which are all used later during plotting)
  * file name containing the measured particle size data (given in `particle_distribution_raw_data.csv`)
  * number of bins (per default, the particle sizes are discretized in 10 bins)
  * the cumulative distribution functions that are being fitted to the measured data (per default, all functions listed above are used)
  * additional configuration settings used for plotting (per default, these settings are kept unchanged)
* `FITTING`: fits the cumulative distribution functions to the measured data, prints the results in the console and saves a figure of the measured data together with all fitted functions in the working folder (in this example: `FitComp_CompoundA_BatchX.png`).
* `DISCRETIZATION`: Using the cumulative distribution function with the best fit (i.e. lowest squared error), the particle sizes are discretized in bins. The number of bins is defined above. Bin borders are defined by equidistant binning between the minimum and maximum radius. Minimum and maximum radius are defined as the 0.001 and 0.999 quantile of the best fit distribution. The representative radius of each bin is the respective quantile of the mean of each bin's probability range. The fit results (name of best fitting function and fitted parameter values) and the discretized particle sizes are written in a csv file and saved in the working folder (in this example: `ParticleDistr_CompoundA_BatchX_lognormal.csv`).
* `PLOT BINS`: saves a figure showing the fitted density function together with the equidistant bin mean sizes and another figure showing the fitted cumulative density function together with the equidistant bin mean sizes and observed data in the working folder (in this example: `ProbDensBinned_CompoundA_BatchX.png` and `CumDistrBinned_CompoundA_BatchX.png`).
* `EXPORT RESULTS FOR MOBI`: saves the particle radius (= particle size/2) and relative amount factor (`rel_amountFactor`, i.e. the probability density normalized to sum up to 1.0) as Excel file that can be imported as as parameter start values into MoBi<sup>®</sup> in the working folder (in this example: `PSV_CompoundA_BatchX_lognormal.xlsx`).

To conduct the fitting, open the R-file and adjust relevant code in the section `MEASURED PARTICLE SIZE DATA` (see details above) and, if needed, define further distribution functions in the section `FUNCTIONS` (this is not a prerequisite). Execute the R script. 

### 2. Fitting the dissolution function to measured dissolution profiles in biorelevant media *in vitro*

In this step, the particle dissolution model describing *in vitro* dissolution profiles measured in biorelevant media (e.g. FaSSGF, FaSSIF, FeSSIF) is established in MoBi<sup>®</sup>. The structure of the particle dissolution model implemented in the MoBi<sup>®</sup> file distributed with this repository is schematically shown in the figure below.

<img width=512 src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/MoBi_DissolutionModelStructure.png" alt="Dissolution Model Structure MoBi" />

This dissolution model corresponds structurally to the particle dissolution model implemented in PK-Sim<sup>®</sup> [[4](#References)], but the parametrization is different as it accounts for the conditions of the *in vitro* dissolution experiment (e.g. volume and pH of the solvent). The particle size radii from the previous step (and their probability density) are incorporated in the dissolution model via import as `Paramater Start Values`. Unknown parameters of the model (typically, the aqueous diffusion coefficient and the thermodynamic solubility of the drug) are identified through the `Parameter Identification` module in MoBi<sup>®</sup>. Once a particle dissolution model has been successfully established, the parameter values of the dissolution model will be transferred to PK-Sim<sup>®</sup> in step 3 of this workflow.

The following files are used in this step:

* `in vitro dissolution model.mbp3`
* `DissolutionData.xlsx`

#### Detailed descriptions and instructions:

`DissolutionData.xlsx` contains the dissolution data in biorelevant media. In this example, the following hypothetical dissolution data of four different doses (20, 50, 100 and 200 mg) in FaSSIF have been used:

<img width="640" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/Conc~Time_all_2.png" alt="Concentrations versus Time" />

The observed data were loaded in the MoBi<sup>®</sup> file `in vitro dissolution model.mbp3`. 

This MoBi<sup>®</sup> file has the particle dissolution model implemented in the `Passive Transports` building block. Experimental conditions and physicochemical properties of the drug are defined in the `Paramater Start Values` building blocks for each of the experiments to be simulated. Each `Paramater Start Values` building block contains information on the drug dose, particle size distribution of the drug batch, volume and pH of the biorelevant medium and the drug's thermodynamic solubility in the medium. Consequently, each simulation in MoBi<sup>®</sup> generally requires its own `Paramater Start Values` building block. Additional `Paramater Start Values` building blocks can created be cloning an existing building block and modifying all relevant parameter start values manually (or via the import function in the context menu). 

The following parameter start values need to be manually adjusted:

- `Volume`: volume of the biorelevant medium
- `Density (drug)`: density of the drug used in the dissolution experiment
- `Dose`: mass of the drug used in the dissolution experiment
- `Molecular weight`: molar mass of the drug used in the dissolution experiment
- `PrecipitatedDrugSoluble`: Boolean function controlling whether the precipitated drug can (re-)dissolve (set value to `1`) or not (set value to `0`)
- `precipitationrate`: precipitation rate of the drug (only used if `PrecipitatedDrugSoluble` is set to `0`)
- `pH`: pH of the of biorelevant medium
- `Reference pH`: pH at which the `Solubility at reference pH` of the drug is measured
- `Solubility at reference pH`: thermodynamic solubility of the drug measured at `Reference pH`
- `Compound type {0...2}`: defines the drug's ionization state; the following values can be used: `-1`: acid; `0`: neutral; `+1`: base
- `pKa value {0...2}`: p*K*<sub>a</sub> value for functional group `{0...2}` in the drug

Furthermore, the following parameters can be changed for more advanced settings (although it is recommended to keep them set to the default values):

- `Thickness (unstirred water layer)`: unstirred water layers thickness
- `NBins`: number of particle size bins (note that if the number of bins is increased beyond 10, the model structure has to be adjusted)
- `Solubility gain per charge`: factor by which the solubility increases with each ionization step

Finally, the following parameters are model extensions and it is generally recommended to keep the default values unchanged:

- `SDS Dose`: Mass of an excipient in the tablet (e.g. sodium dodecyl sulfate) that may enhance the solubility of the drug according to a log-linear relationship; for the sake of simplicity, this excipient is assumed to be instantaneously dissolved in the model 

- `SDS dependent Solubility`: Boolean function switching the log-linear relationship between dissolved excipient concentration and drug solubility on (set value to `1`) or off (set value to `0`). If set to `1`, the drug's solubility is calculated as follows: 

  <a href="https://www.codecogs.com/eqnedit.php?latex=\texttt{Solubility&space;}&space;=&space;\texttt{&space;SDS&space;mediated&space;increase&space;in&space;solubility&space;factor&space;}\cdot\texttt{&space;}log\left({\texttt{SDS&space;concentration}}\right)\texttt{&space;}&plus;\texttt{&space;Solubility&space;at&space;reference&space;pH&space;}\cdot\texttt{&space;}&space;\frac{\texttt{Solubility&space;pKa&space;reference&space;pH&space;factor}}{\texttt{Solubility&space;pKa&space;pH&space;factor}}" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\texttt{Solubility&space;}&space;=&space;\texttt{&space;SDS&space;mediated&space;increase&space;in&space;solubility&space;factor&space;}\cdot\texttt{&space;}log\left({\texttt{SDS&space;concentration}}\right)\texttt{&space;}&plus;\texttt{&space;Solubility&space;at&space;reference&space;pH&space;}\cdot\texttt{&space;}&space;\frac{\texttt{Solubility&space;pKa&space;reference&space;pH&space;factor}}{\texttt{Solubility&space;pKa&space;pH&space;factor}}" title="\texttt{Solubility } = \texttt{ SDS mediated increase in solubility factor }\cdot\texttt{ }log\left({\texttt{SDS concentration}}\right)\texttt{ }+\texttt{ Solubility at reference pH }\cdot\texttt{ } \frac{\texttt{Solubility pKa reference pH factor}}{\texttt{Solubility pKa pH factor}}" /></a>

- `SDS-mediated increase in solubility factor`: slope factor of the log-linear relationship between dissolved excipient concentration and drug solubility. 

- `Disintegration Lambda`: Variable used in the Weibull function to reduce the initial number of particles

- `enableTabletDisintegration`: Boolean function switching the tablet disintegration on (set value to `1`) or off (set value to `0`). If set to `0`, the typical dissolution function with a constant number of drug particles in each particle size bin is used [[2,3](#References)]. If set to `1`, drug dissolution is initially slowed down by multiplying the number of particles in the dissolution equation with a time-dependent factor that increases from 0 to 1 according to a Weibull function. The number of particles is then calculated as follows: 

  <a href="https://www.codecogs.com/eqnedit.php?latex=\texttt{Number&space;of&space;particles&space;}=\texttt{&space;}(1-exp(-\texttt{lambda}\cdot&space;\frac{\texttt{Time}}{\texttt{Tablet&space;surface&space;area}}^{\texttt{alpha}}))\cdot&space;(\texttt{Maximum&space;number&space;of&space;particles}-1)&plus;1" target="_blank"><img src="https://latex.codecogs.com/gif.latex?\texttt{Number&space;of&space;particles&space;}=\texttt{&space;}(1-exp(-\texttt{lambda}\cdot&space;\frac{\texttt{Time}}{\texttt{Tablet&space;surface&space;area}}^{\texttt{alpha}}))\cdot&space;(\texttt{Maximum&space;number&space;of&space;particles}-1)&plus;1" title="\texttt{Number of particles }=\texttt{ }(1-exp(-\texttt{lambda}\cdot \frac{\texttt{Time}}{\texttt{Tablet surface area}}^{\texttt{alpha}}))\cdot (\texttt{Maximum number of particles}-1)+1" /></a>

- `Disintegration Alpha`: Variable used in the Weibull function to reduce the initial number of particles

Additionally, the Excel file containing the particle size radii and their probability generated in step 1 of this workflow (in this example: `PSV_CompoundA_BatchX_lognormal.xlsx`) has to be imported in the existing `Paramater Start Values` building block in MoBi<sup>®</sup>. This will import the values for the initial radii of the bins and the relative amount of the bins and overwrite existing values. The imported values should be displayed at the end of the list:

<img width="768" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/PSV_screenshot.png" alt="Parameter Start Values" />

Of note, if dissolution has been measured under different experimental conditions (e.g. different dose, biorelevant media pH or volume, or particle size distribution), it is recommended to clone an existing `Paramater Start Values` building block and manually adjust the respective parameter values. 

Once all `Paramater Start Values` have been defined, the simulation(s) can be set up and unknown parameter(s) can be fitted via the `Parameter Identification` module. In this example, the `Aqueous diffusion coefficient` and `Solubility at reference pH` have been optimized with the following results:

<img width="512" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/InVitroDisso_FitResult.png" alt="InVitroDisso_FitResult" />

<img width="640" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/InVitroDisso_FitResult2.png" alt="InVitroDisso_FitResult2" />

### 3. Transfer particle size distribution and particle dissolution parameters to PK-Sim<sup>®</sup>

In this step, the discretized particle sizes generated in step 1 and the parameters of the particle dissolution function from step 2 will be transferred to PK-Sim<sup>®</sup> for predicting dissolution in the gastrointestinal tract *in vivo*.

The following files are used in this step:

* `JSON.R`
* `BB_Administration_ParticleDissolution_10Bins_input.json`
* `BB_Formulation_ParticleDissolution_10Bins_input.json` 

#### Detailed descriptions and instructions:

`JSON.R` is a R script file that generates `Administration` and `Formulation` building blocks with the correct properties (e.g. drug mass of a given dose in each particle size bin and particle radius of each bin) that can be loaded in PK-Sim<sup>®</sup>. In the first section of the R script, the Excel file containing the particle size radii and their probability generated in step 1 of this workflow (in this example: `PSV_CompoundA_BatchX_lognormal.xlsx`) is loaded; note that the file name of this Excel file may need to be adjusted. Thereafter, the script contains the following sections:

* `PARSE FORMULATION BB`: loads the input file `BB_Formulation_ParticleDissolution_10Bins_input.json`, a dummy `Formulation` building block in `json` file format containing values of the following formulation parameters for 10 particle size bins each:

  * `Thickness (unstirred water layer)`: per default set to 20 µm (the value can be changed, if necessary)
  * `Type of particle size distribution`: in the current workflow, the value of this parameter is irrelevant and can be kept as is
  * `Particle radius (mean)`: the particle size radius of each bin.

  For each particle size bin, the value of `Particle radius (mean)` is set according to the particle size radii generated in step 1 of this workflow that are stored in the respective Excel file (in this example: `PSV_CompoundA_BatchX_lognormal.xlsx`). Note that the particle size radii are converted from µm to mm, to keep the character encoding compliant with ASCII.

* `PARSE ADMINISTRATION BB`: loads the input file `BB_Administration_ParticleDissolution_10Bins_input.json`, a dummy `Administration` building block in `json` file format containing values of the following oral administration parameters for 10 particle size bins each:

  * `StartTime`: start time of administration in hours (per default `0`)
  * `InputDose`: drug mass present in each particle size bin
  * `Volume of water/body weight`: volume of water per kg body weight that is administered with the drug mass.

  For each particle size bin, the value of `InputDose` is calculated according to the probability density of each particle size radius (`rel_amountFactor_i`) generated in step 1 of this workflow that is stored in the respective Excel file (in this example: `PSV_CompoundA_BatchX_lognormal.xlsx`). For this calculation, the (total) dose and molecular weight of the drug need to be defined manually in the script (line 35, 36). 

Note that these dummy files contain 10 particle size bins and need to be adjusted, if the number of bins is different.

To generate building blocks for import in PK-Sim<sup>®</sup>, adjust the R script where necessary and execute the code. This will generate the following two output files in the working directory:

* `BB_Administration_ParticleDissolution_10Bins_output.json`
* `BB_Formulation_ParticleDissolution_10Bins_output.json`

These output files can be loaded as building blocks in PK-Sim<sup>®</sup> if the application is started in developer mode via the Windows command prompt as follows (see also [here](https://github.com/Open-Systems-Pharmacology/Forum/issues/305) for an alternative way): 

`cd C:\Program Files\Open Systems Pharmacology\PK-Sim 9.1`

`PKSim /dev`

Note that the PK-Sim<sup>®</sup> directory folder (`C:\Program Files\Open Systems Pharmacology\PK-Sim 9.1`) may be different and needs to be adjusted in this case.

After executing the commands above, PK-Sim<sup>®</sup> will be started in developed mode which adds some extra features, specifically on context menus (all suffixed with the text `(Developer Only)`), that are mostly used to debug issues or generate template files. It does not change the behavior of the application. The generated output files can now be loaded as building blocks via the context menu as shown below.

<img width="384" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/PKSim-dev_LoadFormulationBB.png" alt="PKSim-dev_LoadFormulationBB" /> 

<img width="384" src="https://github.com/AndreDlm/IVIVC-with-particle-dissolution-module-in-OSP/blob/master/Figures/PKSim-dev_LoadAdministrationBB.png" alt="PKSim-dev_LoadAdministrationBB" /> 

Once the building blocks are loaded in PK-Sim<sup>®</sup>, a simulation can be set up to simulate dissolution and absorption in the gastrointestinal tract. Please note that compound-specific parameters relevant for particle dissolution that have been fitted in MoBi<sup>®</sup> (e.g. the `Aqueous diffusion coefficient` or `Solubility at reference pH`) should be the same in the `Compound` building block in PK-Sim<sup>®</sup> and may hence need to be adjusted before running a simulation.




## References

[1][ Noyes, A. A., & Whitney, W. R. (1897). The rate of solution of solid substances in their own solutions. *Journal of the American Chemical Society*, *19*(12), 930-934.](https://pubs.acs.org/doi/pdf/10.1021/ja02086a003)\
[2][ Dressman, J. B., & Fleisher, D. (1986). Mixing-tank model for predicting dissolution rate control of oral absorption. *Journal of pharmaceutical sciences*, *75*(2), 109-116.](https://pubmed.ncbi.nlm.nih.gov/3958917/)\
[3][ Hintz, R. J., & Johnson, K. C. (1989). The effect of particle size distribution on dissolution rate and oral absorption. *International Journal of Pharmaceutics*, *51*(1), 9-17.](https://www.sciencedirect.com/science/article/pii/0378517389900690)\
[4][ Open Systems Pharmacology Suite Manual. Version 7.4, October 2018, accessed 07-28-2020](https://docs.open-systems-pharmacology.org/working-with-pk-sim/pk-sim-documentation/pk-sim-formulations#particle-dissolution)
