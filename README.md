# PSMCplus

A new implementation of Li and Durbin's pairwise sequentially Markovian coalescent ([PSMC](https://pubmed.ncbi.nlm.nih.gov/21753753/)) that accounts for heterogeneity across the genome. 

## Installation

PSMC+ is written in Python. You will need numpy, numba, pandas, joblib, scipy, psutil and matplotlib. You can install these with:

```
conda create --name PSMCplus
conda install numpy numba pandas joblib scipy psutil matplotlib
conda activate PSMCplus

```

To check whether the installation worked, you can run with some test data:<br>
`python /home/<user>/PSMCplus/PSMCplus.py -in /home/<user>/PSMCplus/simulations/constpopsize.mhs -D 10 -b 100 -its 1 -o /tmp/deleteme`

`Arun: does this create a /tmp/deleteme folder? or should the user make one?`

## Quick start

### Input files

PSMC+ takes multi-hetsep (mhs) files as introduced by Stephan Schiffels. To generate these files, you can use [his tutorial](https://github.com/stschiff/msmc-tools/blob/master/msmc-tutorial/guide.md). If you have a CRAM/BAM file, you can use my [Snakefile](https://github.com/trevorcousins/PSMCplus/blob/master/Snakefiles/processing_data/Snakefile) as guide for how to process this into mhs files. 

### Inference of population size history

You can run PSMCplus with the following command line: 

`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> | tee <logfile>`

`Arun: are all these command line args required? If not, what's the minimal number that are required?`

`D` is the number of discrete time interval windows, `b` is the genomic bin size, and `its` is the number of iterations (see the Advanced section for a more detailed explanation). `<infiles>` is a string (separated by a space if more than one) that points to the mhs files. The inferred parameters will be saved to `<outprefix>final_parameters.txt` and a log file will be saved to `<logfile>`. The output file contains `D` rows and 3 columns, which are left time boundary, the right time boundary, and the coalescence rate per discrete time window. To scale the times into generations, you must divide by `mu`. We get the effective population size by taking the inverse of the inverse coalescence rate and dividing this by `mu`. 

You can plot this in Python using the following code:
```
final_params = np.loadtxt(final_parameters_file)
time_array = list(final_params[:,1])
time_array.insert(0,0)
time_array = np.array(time_array)
plt.stairs(edges=(time_array/mu)*gen,values=(1/final_params[:,2])/mu,label=population,linewidth=4,linestyle="solid",baseline=None)
plt.xscale('log')
plt.ylabel('$N_e(t)$')
plt.xlabel('Years')
```   
    
See the [Inference Tutorial notebook](https://github.com/trevorcousins/PSMCplus/blob/master/tutorial/Inference_tutorial.ipynb) for a specific example, and the Advanced section for a more detailed explanation of the hyperparameters. 

### Decoding 

You can decode the HMM (that is, get the inferred coalescence times across the genome) with: 

`python /home/<user>/PSMCplus/PSMCplus.py -in <infile> -D <D> -o <outprefix> -decode -decode_downsample <decode_downsize> | tee <logfile.txt>`

The argument `-decode` tells PSMC+ to decode the HMM, as opposed to inferring the $N_e$ parameters. The decoding file is large - you can reduce disc space by saving the posterior probabilities only at every X base pairs with the `-decode_downsample` argument, where `X = b*decode_downsize` (`b` is the genomic binsize, see above or Advanced) `Arun: can you tell me how to get the output every 10KB? do I set it to -decode_downsample 10000?`. I recommend that when decoding you provide the command line with the inferred inverse coalescence rate parameters, as assuming an incorrect demography can induces bias (see Schweiger et al Sup Fig X `Arun: link to this figure. Is this the genome research paper?`). See `-lambda_A_fg` in the Advanced section for more information.  See the [Tutorial](https://github.com/trevorcousins/PSMCplus/blob/master/tutorial/Inference_tutorial.ipynb) for an example.

For the output file, the first row is the position, and the remaining rows are the probability of coalescing in each time window (which is set by the `-D` flag), with the first row being the most recent and the last row the most ancient. The time windows are in coalescent units (`Arun: 2N0? 4N0?`), and the boundaries are detailed in the comment at the top of the file. See the [Inference Tutorial](https://github.com/trevorcousins/PSMCplus/blob/master/tutorial/Inference_tutorial.ipynb) for parsing this file. 

# Advanced

Here is a more in depth explanation of the hyperparameters for PSMC+. Quick preliminaries: PSMC+ works in coalescent units, so uses `theta` and `rho`. theta is the scaled mutation rate and is equal to `4*N*mu` where `mu` is the rate per generation per base; `rho` is the scaled recombination rate and is equal to `4*N*r` where `r` is the rate per generation per neighbouring base pairs. The inferred parameters are scaled to time in years by a user-provided mu and generation time, and to real units of `N` also by `mu`. 

-theta 
The scaled mutation rate, `theta = 4*N*mu`. 

Default behaviour is to calculate this from the data with theta=number_hets/(sequence_length-number_masked_bases). It is not subsequently updated as part of the expectation-maximisation (EM) algorithm. Usage: if you instead want to give a particular value of theta, e.g. 0.001, you can do: 
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -theta 0.001 | tee <logfile>`

-mu_over_rho_ratio 
The ratio of the mutation rate to the recombination rate (this parameter should really be called "theta_over_rho_ratio").

If you know the ratio of the mutation to recombination rate, then the starting value of rho is `rho=theta/mu_over_rho_ratio`. 
Default behaviour is to set `mu_over_rho_ratio=1.5`.

In humans, the genome wide average rate of `mu` and `r` are currently taken to be 1.25e-8 and 1e-8, respectively. This ratio will not be true in all species and it requires a bit of thought. Note that if `r>mu` then it is not recommended to use PSMC, as the algorithm is not able to accurately infer the coalescence times because a typical stretch of the genome will experience a recombination before a mutation. If there are no mutations on a genomic segment then there is no way for its age to be estimated.

Usage: if you want to give a particular value of `mu_over_rho_ratio`, e.g. 2, you can do: 
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -mu_over_rho_ratio 2 | tee <logfile>`

-rho
The scaled recombination rate, `rho = 4*N*r`. 

Default behaviour is to set `rho=theta/mu_over_rho_ratio` (see above), then update rho every iteration as part of the EM algorithm. Useage: if you have a particualrly good idea about what rho is, e.g. 0.0005, you can give it with
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -rho 0.0005 | tee <logfile>`

-rho_fixed
Do not infer rho as part of the EM algorithm. 

By default rho is updated as part of the EM algorithm. This has been shown not to be particularly accurate, but historically has been standard practise. If you do not want to update rho, you can add the flag -rho_fixed which will force rho to remain at its starting value. Usage: 
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -rho_fixed | tee <logfile>`

-b 
Genomic bin size. 

You can bin the genome into windows of size `b`, which enforces the assumption that recombinations can occur only between bins. The primary advantage of doing this is speeding up the code by a factor of `b`, and also reducing RAM by a factor of `b`. Default behaviour is `b=100`. In humans, `b=1` and `b=100` seem to be indistinguishable, and one could probably get away with using even `b=200` or possibly even greater. Usage: if you want to use a bin size of 100 you can do:
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b 100 -its <its> -o <outprefix> -rho_fixed | tee <logfile>`

-lambda_segments 
Fix some adjacent time windows to have the same coalescence rate.

You may want to force adjacent intervals to have the same coalescent rates, as in PSMC and MSMC/MSMC2, though I don't recommend this. For the undeterred, if for example you want to use 64 discrete time windows with the first 4 fixed to be the same, the next 20 in pairs of two, then the final 20 intervals to be free you can do: 
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -lambda_segments 1*4,20*2,20*1 | tee <logfile>`
so the argument is parsed as "1 lot of four fixed intervals, 20 lots of two, and 20 lots of one". If you don't want to infer parameters for some time windows, instead leaving it at its starting value, you can give it a "0". For example if you want 32 time windows with the first 10 to be fixed at their starting value, you can do: 
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -lambda_segments 10*0,22*1 | tee <logfile>`

Default behaviour is to assume all intervals are free.

-lambda_A_fg
The first guess for the inverse coalescence rates, in each discrete time interval. 

A comma separated list of floats that are taken are used as the starting guess for the inverse coalescence rates. You should also use this to decode with the inferred population size parameters. The length of this list must be equal to the number of segments as specified by the `-lambda_segments` argument. If you have 10 deiscrete time intervals and know your inverse coalescence rates are [1,1,5,5,5,5,5,1,1,1] (a population that experiences a five-fold bottleneck) yuo can do:
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o inference/final_parameters.txt -lambda_A_fg 1,1,5,5,5,5,5,1,1,1 | tee <logfile>`

Default behaviour is to assume `lambda_A_fg` is 1 everywhere. 
See the [Inference Tutorial](https://github.com/trevorcousins/PSMCplus/blob/master/tutorial/Inference_tutorial.ipynb) for more information on this, especially for the decoding. 

-thresh 
Stop iterating the EM algorithm after the change in log-likelihood is less than thresh. 

Typically, PSMC or MSMC/MSMC2 are run for 20 or 30 iterations. Heng Li demonstrates that this is sufficient (atleast in humans) for a satisfactory "goodness of fit" (see Supplementary Figure XXX), and for recovering $N_e$ parameters. However you might wish for a more specific convergence criteria, such as the change in log-likelihood of the EM algorithm to be less than a particular value: you can achieve this with `thresh`. 
Default behaviour is to run for 30 iterations - if both `its` and `thresh` are given, then the algorithm will terminate when the first of either is satisfied. Usage: if you want a `thresh` of 1 then you can do 

`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -thresh 1 | tee <logfile>`

-spread_1, spread_2
The spread of the discrete time window boundaries. 

The time window boundaries are given by the following equation: 
$\tau_i = spread_1 exp\left( \frac{i}{D}log\left(1+\frac{spread_2}{\omega}\right)-1\right)$
Where spread_1 controls the dispersion of intervals in recent time, and spread_2 controls the dispersion in ancient time. In humans, I recommend `spread_1=0.05` and `spread_2=50`. This may not be optimal in other species and it is probably a good idea to experiment with different combinations. 
Default behaviour is to set `spread1=0.05` and `spread2=50`. Usage: if you want `spread_1=0.05` and `spread_2=50`
`python /home/<user>/PSMCplus/PSMCplus.py -in <infiles> -D <D> -b <b> -its <its> -o <outprefix> -spread_1 0.05 -spread_2 50 | tee <logfile>`

## Simulation

If you want to simulate a panmictic demography, you probably want to use msprime or SLiM as these are extremely powerful and flexible. However, I also provide functionality to simulate directly from the PSMC HMM (this is necessarily simulating from the SMC' model, which is marginally a very good approximation to the full coalescent with recombination - see Wilton et al 2015). An example command line to simulate a constant population size is:
`python /home/<user>/PSMCplus/simulate_HMM.py -D 10 -theta 0.001 -rho 0.0005 -o_mhs simulations/sim1_variants.mhs -o_coal simulations/sim1_coal.txt.gz -spread_1 0.1 -spread_2 50 -L 1000000`
`D`, `theta`, `rho`, `b`, `spread_1` and `spread_2`, are as described above. `L` is an integer for the sequence length. 
This will save an mhs file detailing the variant positions to simulations/sim1_variants.mhs, and a file detailing the coalescent data (pairwise coalescence times across the genome) to `simulations/sim1_coal.txt.gz`. The third column of this file is the index of the coalescent time between the genomic positions described in the first and second column. 

To simulate with a changing population size, you must give an argument `-lambda_A`, which is a comma separated string where each value is the inverse coalescence rate in each time interval. For example if you want a simulation with 32 discrete time windows with a population of relative size 1 that doubles in size between time index 10 and 20, then you can do: 
`python /home/trevor/PSMCplus/simulate_HMM.py -D 32 -lambda_A 1,1,1,1,1,1,1,1,1,1,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,1,1,1,1,1,1,1,1,1,1,1,1,1,1 -theta 0.001 -rho 0.0005 -o_mhs simulations/sim2_variants.mhs -o_coal simulations/sim2_coal.txt.gz -L 1000000`
You can visualise the desired demography and parameters from the simulation in the [Simulation Tutorial](https://github.com/trevorcousins/PSMCplus/blob/master/tutorial/Simulation_tutorial.ipynb).


## Varying mutation (or recombination) rates

TODO elaborate. 
PSMC+ enables a user to provided a map of varying mutation or recombination rates. These are bed files where the columns are chromosome, start, end, rate. Rate corresponds to the local mutation map. Note that recombination maps are not yet fully implemented. 
