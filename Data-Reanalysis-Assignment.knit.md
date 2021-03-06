---
title: "Data-Reanalysis-Assignment"
author: "Arora"
date: "December 3, 2017"
output: html_document
---
#Replication of Ellefsen and Smith 2016

##Introduction 

#####The following is the reanalysis of Ellefsen and Smith's 2016 paper Manuel hierarchical clustering of regional geochemical data using a Bayesian finite mixture model. 

#####In this study the authors use a Bayesian finite mixture model to cluster geochemical data from a USGS survey ofsoil geochemistry in Colorado. They use a Bayesian model instead of other clustering techniques that others have previously used. 

#####In their paper they use hierarchical modeling to create two clusters of the data, with the goal of breaking those clusters into two more clusters. Each set of clusters should represent different geologic, climatic, and biological processes occuring at different scales. Clustering is a useful technique because this type of survey data can encompass thousands of samples with many elements measured (given the constraints of my equipment, I measure 13 however in this study they begin with 44 elements). Such large datasets are difficult to analyze without multivariate techniques. 

#####To derieve the parameters for their model they use the Monte Carlo sampling and then check each model again previously known geologic models. The model which best fits that previous knowledge is used from analysis. 


##Dataset

#####The dataset used in the paper and in this replication can be found in the GcClust package which can be download from https://pubs.er.usgs.gov/publication/tm7C13

#####The dataset for this paper was collected for a USGS (United States Geological Service) survey of soil geochemistry in the state of Colorado. The original dataset for this study consisted of 966 samples with 44 elements measured. Six samples were excludeded because of issues with their location and one was removed because of anthropogenic effects. Five elements were removed because their measured concentrations often fell below their detection limits. This leaves 959 samples with 39 measured elements. Before analysis began the sum of the concentrations of the excluded elements were added up into a column, and the concentrations of the elements were scaled to concentration(mg/kg). 

##Outline of Analyses

#####Each of these steps, and the arguments for making them will be outline during the replication itself. 

#####1. Transform compositional data in isometric log-ratio (ilr) coordinates, and transform the ilr coordinates with the robust principle components transform. 

#####2. Select a subset of components. 

#####3. Use Monte Carlo Sampling to produce the chains. 

#####4. Select chains for furthur anslysis.

#####5.Combining the chains and switching them into their proper place. (The switching could not be done for reasons stated below).

#####6. Calculate conditional probability that field sample is associated with the first probability density function in the finite mixture model. 

#####7.  Use the conditional probabilities to calculate mean vectors, standard deviations, and correlation matrices for both probability density functions for the finite mixture model. 

#####8. Plot the observed statistics (mean vectors, standard deviations, and correlation matrices). 

#####9. Transform the observed data into simplex, which is compositional center and variation matrix. 

#####10. Visualize the data by looking at compositional centers and variation matrices. Map the clusters in terms of their relationship with the probability density functions.  


##Compositional Data

#####Compositional data is always postive and adds up to 100% and in geology is often derieved from counts data which is scaled into the units of concentration. One of the problems with composition data is that because all the data is in the positive real space, measures such as variance are considered impractical. This particular issue is called the constraint problem and can be solved through a transformation. Currently the most robust, though statistically least understood, method for transformation is the isometric log-ratio transformation. Like most transformations, it is difficult to relate the results after the transformation has taken place with the original dataset. For this reason compositional data should be analyzed in the following way: (1) transform the data, (2) perform any desired statistical analyses, (3) analyze the results, usually by transforming back into the original units. 

#####One important concept about compositional data that is central to geochemical work is subcompositional coherence. This essentially means that taking a part out of the whole does not change the value of the part. 


##Reanalysis

#####First download and open the following packages: {colorspace}, {GcClust}, {ggplot2}, {maps}, {mvtnorm}, {reshape2}, {robustbase}, {rstan}, {sp}, {shiny}

```r
library(colorspace)
```

```
## Warning: package 'colorspace' was built under R version 3.4.2
```

```r
library(GcClust)
library(ggplot2)
```

```
## Warning: package 'ggplot2' was built under R version 3.4.2
```

```r
library(maps)
```

```
## Warning: package 'maps' was built under R version 3.4.2
```

```r
library(mvtnorm)
library(reshape2)
```

```
## Warning: package 'reshape2' was built under R version 3.4.2
```

```r
library(robustbase)
```

```
## Warning: package 'robustbase' was built under R version 3.4.2
```

```r
library(rstan)
```

```
## Warning: package 'rstan' was built under R version 3.4.2
```

```
## Loading required package: StanHeaders
```

```
## Warning: package 'StanHeaders' was built under R version 3.4.2
```

```
## rstan (Version 2.16.2, packaged: 2017-07-03 09:24:58 UTC, GitRev: 2e1f913d3ca3)
```

```
## For execution on a local, multicore CPU with excess RAM we recommend calling
## rstan_options(auto_write = TRUE)
## options(mc.cores = parallel::detectCores())
```

```r
library(sp)
```

```
## Warning: package 'sp' was built under R version 3.4.2
```

```r
library(shiny)
```

```
## Warning: package 'shiny' was built under R version 3.4.2
```
#####As per the instructions within the supplementary information for the paper, add dataset into gcData. 

```r
gcData<-CoGeochemData 
#in their instructions they save their data at each point in .dat files however I did not.
#the data needs to be put into gcData because many of the functions (found on their github site here: https://github.com/USGS-R/GcClust/blob/master/R/GcClusterFunctions.R) require the dataset to be in a list called gcData. 
```
#####The following is a map of the points where samples were taken. 

```r
maps::map(database="state", regions = "Colorado", fill=FALSE)
plot(gcData$concData, add = TRUE, pch =
16, cex = 1/3)
```

<img src="Data-Reanalysis-Assignment_files/figure-html/unnamed-chunk-3-1.png" width="672" />

```r
#their original code had fill, a white border, and red points however I could not get that particular code to work so I edited so the points were back and the fill was removed. 
```

![This is the map Ellefsen and Smith included in their paper. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Original_Map.jpg)

#####Step 1: Transform the concentration data first by isometric log-ratio (ilr) transform and then the robust principle components transform. The ilr transform changes the data into a form that can be clustered. The robust principle componetns transform is done because the authors assume it will make the following model more stable by reducing the number of dimensions. Both transformations occur within the transformGcData function. 


```r
transData<-transformGcData(gcData)
#I decided not to use the code head(transData) because when doing so, it seems the entire data comes up rather than just the beginning
```

#####Step 2: Select the number of principle components. The authors present several plots to determine the number of principle components they will use however in the end decide to use the cumulative method, which is when the analyst keeps adding principle components until a certain percentage of the dataset is selected for. For the purpose of replicating their study I have followed their example. 

####The following are two plots of the principle components. The first is a box plot and the second is a violin plot, ,though they show the same information. As with box plots, the lower end of the whisper shows the 25th percentile and the end of the upper whisker shows the 75th percentile. At this stage my plots and their plots look exactly the same. 


```r
plotEdaDist(transData)
```

<img src="Data-Reanalysis-Assignment_files/figure-html/unnamed-chunk-5-1.png" width="672" />

![This is a correlation matrix of the principle components using the Pearson's correlation coefficient, which is like R-squared, so a -1 exactly negative correlation, a zero is correlation, and a +1 is complete positive correlation. However if this graph works like the same type of graph below (that graph is the matrices used for model checking), then the graph should look very similar across the red diagonal, and the value of the coefficient is not really taken into account. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Principle_Component_Plots.jpg)

#####I did not find this particular graph helpful when attempting to determine the number of principle components to select. 


```r
plotEdaCorr(transData)
```

```
## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.
```

<img src="Data-Reanalysis-Assignment_files/figure-html/unnamed-chunk-6-1.png" width="672" />

![This is their correlation matrix and their matrix looks very similar across the red diagonal. The histogram of the correlations and sits in the range of very poor correlation in the both positive and negative directions.](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Correlation_Matrix_for_PCA.jpg)
#####I also did not find these graphs helpful when attempting to determine principle components. When I begin to edit the functions and add in my own dataset, I will disregard these graphs. 


#####Now we will analyze a scree plot. In R in Action a Scree plot is described as one way of determing the number of principle components to keep for analysis. While Ellefsen and Smith (2016) decide to use add up principle components until they come up with a certain value, a scree plot is uses a matrix of Eigenvalues versus the component. Depending on the type of scree test performed, the user is either looking for a bend or "elbow" in the graph or the number of components where the eigenvalue is less than one. Because the eigenvalues were not plotted, instead we will look for the dip to determine the number of princpal componetns as well as the cumulative variance we want to account for.And while choosing 39 principle components will describe all the data, we perform principle components analysis to essentially reduce the number of variables. 


```r
plotEdaVar(transData)
```

<img src="Data-Reanalysis-Assignment_files/figure-html/unnamed-chunk-7-1.png" width="672" />
![This is the scree plot presented in the paper. The authors wanted to select enough principle components to explain 75% to 95% of the data. To do this we could choose any principle component starting at 6 and going until 21, however they decided to select 22 components because that represents 96% of the data. I would probably have chosen 10 principle components. I decided to use 22 principle components after I had to edit some of their functions and wanted to see how my results would differ if I kept everything else the same; I plan to see how changing the number of principle components does change the analysis however running the model below takes upwards of two hours on my compter.](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Scree_Plott.jpg)


```r
nPCs <- 22
```

#####Step 3: Use the mixture model. This file is from their github repository, cited below. I have little experience with Bayesian statistics and no knowledge of the stan programming language, so I have yet to parse out what their code is actually doiong. Expect updates here. 


```r
tmp<-normalizePath(path.package("GcClust"))
load(paste(tmp, "\\stan\\MixtureModel.bin", sep=""))
```

#####The following code is used to sample the chains, however I had to edit this code because R ran an error with some of their code. This function is used in the samplePars function below. This code is available from their github repository. 


```r
sampleFmm <- function(transData, nPCs, sm,
                      priorParams,
                      nWuSamples = 500,
                      nPwuSamples = 500,
                      nChainsPerCore = 2,
                      nCores = 4,
                      procDir = ".") {

  rstanParallelSampler <- function(stanData, sm, nWuSamples, nPwuSamples,nChainsPerCore, nCores, procDir ) {
    CL <- parallel::makeCluster(nCores)
    parallel::clusterExport(cl = CL,
       c("stanData", "sm", "nWuSamples", "nPwuSamples","nChainsPerCore", "procDir"), envir=environment())
    fnlist <- parallel::parLapply(CL, 1:nCores, fun = function(cid) {
      require(rstan, quietly = TRUE)
      fileNames <- vector(mode = "character", length = nChainsPerCore)
      for(i in 1:nChainsPerCore) {
        rng_seed <- sample.int(.Machine$integer.max,1)
        gen_inits <- function() {
          areInGrp1 <- sample(c(TRUE,FALSE), size = stanData$N,
                            prob = c(0.3, 0.7), replace = TRUE)
          return(list(
            theta = runif(1, min = 0.35, max = 0.65),
            mu1 = apply(stanData$Z[areInGrp1,], 2, mean ),
            mu2 = apply(stanData$Z[!areInGrp1,], 2, mean ),
            tau1 = apply(stanData$Z[areInGrp1,], 2, sd ),
            tau2 = apply(stanData$Z[!areInGrp1,], 2, sd ),
            L_Omega1 = diag(stanData$M),
            L_Omega2 = diag(stanData$M)
          ))
        }
        rawSamples <- rstan::sampling(sm, data = stanData,
                                      init = gen_inits,
                                      # control = list(stepsize = 0.00001),
                                      control = list(stepsize = 0.0001),
                                      # control = list(adapt_delta = 0.95),
                                      chains = 1,
                                      iter = nWuSamples + nPwuSamples,
                                      warmup = nWuSamples,
                                      seed = rng_seed, chain_id = cid,
                                      pars=c("theta", "mu1", "mu2",
                                             "tau1", "tau2",
                                             "L_Omega1", "L_Omega2", "log_lik"))
        
        #I took out save_dso=FALSE because R did not run save_dso 
        
        fileNames[i] <- paste("RawSamples", cid, "-", i, ".dat", sep = "")
        save( rawSamples, file = paste(procDir, "\\", fileNames[i], sep = "") )
      }
      return(fileNames)
    } )
    parallel::stopCluster(CL)
    return(unlist(fnlist))
  }
  stanData <- list( M = nPCs,
                    N = nrow(transData$robustPCs),
                    Z = transData$robustPCs[,1:nPCs],
                    priorParams = priorParams )
  fileNames <- rstanParallelSampler(stanData, sm, nWuSamples, nPwuSamples, nChainsPerCore, nCores, procDir )
  return(list(nChains = nChainsPerCore * nCores,
              nWuSamples = nWuSamples,
              nPwuSamples = nPwuSamples,
              fileNames = fileNames))
}
```

#####Now its time to actually run the model. To do this the analyst needs to check how many cores he or she has. To do this open up the library {parallel} and run detectCores(). I have four cores and decided to run 5 chains per core which took around 2.5 hours. The authors had more chains and ran a total of 35 cores which took them about 2 hours. 

#####Our results will look different from here. However the code below will note run because this model cannot knit. I have included images of my own results and interpreted those results as I  can find no way of load a .dat file into R and loading my previous chains into SamplePars.  
 

```r
priorParams <- c(4, 3, 3, 2)
samplePars <- sampleFmm(transData, nPCs,sm, priorParams, nChainsPerCore = 5, nCores = 4)
```

#####Step 4: Selecting the chains for furthur analysis. Currently there are 20 chains will 551 model parameters. To select the chains, first plot the chains. 



```r
plotSelectedTraces(samplePars)
#I only included the picture of the first chain instead of all twenty (from the model I run prior to knitting) because adding all twenty seemed excessive 
```

![This is their image for the first chain. In the standard deviation vector, the second probabiity density function is switched with the first. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Trace_Chain_1.jpg)


![In my first chain, the mean has the second probability density function switched and on top while the standard deviation.](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_chain_1.JPG)

#####Next check each chain to select the model which best fits the data. While it is tempting to select the model with the highest log-likelihood ratio, the modes should be selected by how well they explain previous geochemical knowledge. However the authors do not explain how the models they selected best fit their previous geological knowledge about soils in Colorado


```r
plotPointStats(samplePars)
```

![In their point statistics the first and third chains are switched. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Point_Statistics.jpg)


![In my plot the second and fourth chains are switched and would need to be switched back so that the second probability density function (which is red) is below the first probability density function (which is blue).](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_point_stats.JPG)

#####I too decided to choose the first four chains. Their explanation about how to choose modes that best fit the geochemical data was unclear and I continue to look for furthur resources about how to choose the best model given the previous information.

#####The csv file below allows the combinedChains function to combine the desired chains. I wrote into the csv for the second and fourth chains to be switched. All the information from the first probability density function should be on top of the information from the second probability function. Despite the switching not working, this is explained below, the results of the analysis are very similar.  . 


```r
f<-read.csv(file="C:/Users/Arora/Desktop/selectedChains.csv")
f    
```

```
##   Chain is.Switched
## 1     1       FALSE
## 2     2        TRUE
## 3     3       FALSE
## 4     4        TRUE
```

```r
#Ellefsen and Smith give the format for this file in their guide to using GcClust.
```


```r
selectedChains<-read.csv(file="C:/Users/Arora/Desktop/selectedChains.csv", header=TRUE, stringsAsFactors = FALSE)
```


```r
combinedChains <- combineChains(samplePars,selectedChains)
```


```r
combineChainz <- function(samplePars, selectedChains, procDir = ".") {
  sfList <- vector(mode = "list")
  for(k in 1:nrow(selectedChains)) {
    iChain <- selectedChains[k, "Chain"]
    load( paste(procDir, "\\", samplePars$fileNames[iChain], sep = ""))
    sfList[[k]] <- rawSamples
  return(rstan::sflist2stanfit(sfList))
  }
}
```

#####This step combines the chains together so that we can use them for the next step. The if statement in their combineChains function did not work so I removed that statement and replaced their funciton with combineChainz (with a z).Despite the chains not switching into their correct places, the analysis below worked out very similar to their analysis. 


```r
combinedChains<-combineChainz(samplePars, selectedChains, procDir = ".")
```

#####Now its time to check the model, which includes the mean vectors, the standard deviation vectors, and the correlation matrices for the two probability density functions. To check the model these are compared with the same statistics calculated from the principle components. 

#####To begin checking the model, we need to calculate the conditional probability that a particular sample is associated with the first probability density function. These conditional probabilities, along with the principle components are used to calculate the mean vectors, standard deviation vectors, and correlation matrices necessary for checking our model. 


```r
condProbs1 <- calcCondProbs1 (transData,nPCs, combinedChains)
```

#####This matrix is calculated from the 22 principle components and the conditional probabilities. 


```r
obsTestStats <- calcObsTestStats (transData,nPCs, condProbs1)
```

#####To check the model we will compare the observed statistics (code right above this) with the replicated test statistics which we get from samples of the posterior probability density function. 

#####In this study, the replicated test statistics are the same as the samples of the mean vectors, standard deviation vectors, and correlation coefficients that we combined together in CombinedChains.

#####The following is a plot of the mean and standard deviation vectors. The red dot is the test statistic which should be close to the median, the vertical black line represents the 95% confidence interval. This figure also gives p value between the observed and replicated statistics. 


```r
plotTMeanSd(combinedChains, obsTestStats)
```

![This is Ellefsen and Smith's plots for checking their models. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Model_Checking.jpg)

![These plots look very similar for both runs, except that mine looks more squat. This is surprising seeing as I did not switch any chains. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_plot_meansd.JPG)


#####Now its time to compare the correlation matrices between the test statistics and the replicated statistics. 


```r
plotTCorr(combinedChains, obsTestStats)
#this set of four graphs looks very complex at first. Let's first concentrate on the first and third graphs which are fully coloured and have the red diagonal line. The top right triangle of each graph represents the correlation matrix from the principle components. The bottom left triangle represents the correlation matrix from the replicated samples. The comparison between the two is do they look the same? In their case their graphs do look the same across the red line. 
#The two blue graphs which only fill the top right portion of the graph present the p-values. However these values will be mirror across the diagonal line so both sides are not shown. In this particular analysis, the largest possible posterior predictive p-value is 0.5 and so in their graphs the p-values are, as they state, moderate to large. 
```

![This is their correlation matrices for the probability density functions. For plots A and C, the values look similar across the red line. For plots B and D most of the squares are dark, showing that high p-values for each probability density function. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Model_Check_Matrices.jpg)

![My plots look very similar to their with the plots on the left looking similar across the red line and moderate to high p values for the right two graphs.](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_correlation_matrices.JPG). 

#####The values from combinedChains (mean vector, standard deviation vector, and correlation matrices) are transformed in simplex. Simplex here refers to an algorithm in linear programming which helps with optimization. 


```r
simplexModPar <- backTransform (gcData,nPCs, transData, combinedChains)
```

#####This is reording the elements. I would have done this alphabetically or perhaps in the order of major elements followed by trace elements. 


```r
elementOrder <- c("Sr", "U", "Y", "Nb", "La", "Ce", "Th", "Na", "Al", "Ga","Be", "K", "Rb", "Ba", "Pb", "Cu", "Zn", "Mo", "Mg", "Sc", "Co", "Fe", "V", "Ni","Cr", "Ca", "P", "Ti", "Li", "Mn", "Sn","As", "Bi", "Cd", "In", "S", "Sb", "Tl","W", "EE")
```

#####Using the elementOrder decided on above, median of the compositional centers for each element (the compositional centers is a vector) is plotted. This graph is useful because the units are in concentration, making it easy to compare to other datasets.

#####In the graph there are two dots: one red and one blue. For a particular element the blue dot represents the median concentration as calculated by the first probability density function, and the red dot presents the median concentration as calculated by the second probability density function. In some of the elements it is easy to make out that there is a large difference between the two probability density functions, however some elements are right on top of each other according to this cale. It's for these reasons that the next transformation is done. 


```r
plotCompMeans(simplexModPar, elementOrder)
```

![This is Ellefsen and Smith's compositional centers plot. Note how the values for Thallium and Antinomy are essentially on top of each other so indistinguishable.
](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Compositional_Centers.jpg)

![Note in my plot how the values for Rubidium, Potassium, and Barium are also instituguishable. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_comp_means.JPG)

#####Using simplex again, the statistics are translated to allow us to tell the difference between the medians of the first probability density function versus the second. Ellefsen and Smith also want to add additional information to the plot involving the replicated statistics.


```r
simplexStats <- calcSimplexStats(gcData)
```

#####This plot has thhe translated data which is cleared to read and has the addition of the 95% confidence interval. As always, blue represents the first probaility density function and red represents the second. The only problem is is that with the transformation, we loose the units of concentration and so this data is less comparable to other data sets. 


```r
plotTransCompMeans(simplexModPar,simplexStats, gcData, elementOrder)
```

![Note how due to the translation, we can now distinguish between the two probability density functions for Thallium and Antimony](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Translated_Compositional_Centers.jpg)
 

![]()

#####Our next visualization is looking at the variation matrices for both probability density functions. Like the second and fourth graphs produced in the plotTCorr, the graph is symmetrical across the diagonal line. Unlike the plots in plotTCorr, Ellefsen and Smith decided to combine both variation matrices into one matrix. 

#####The first probability density function is in the upper right triangle and the second is in the lower left triangle, which I think makes understanding the graphs more difficult. 

#####This graph is the the standard deviation of the log-ratios between two elements. I've done similar analyses before however each element is plotted against each other individually which makes the graphs very large but its easier to see the variation (in the case I'm discussing a lot of variation looks like a cloud with no patterns; I'll try and insert an image of it in here however the computer we use might be in Greece still).

#####In mine.... 


```r
plotSqrtVarMatrices( simplexModPar,elementOrder, colorScale = "rainbow" )
```

![In theirs variation matrix it looks like the variance is smaller in the second probability density function than in the first.](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Variation_Matrix.jpg)

![In my variation matrix it looks like the upper triangle has a higher standard deviation than the lower triangle. 
](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_variation_correlation_matrix.JPG)

#####Let's graph the clusters onto the map of Colorado. There are four categories created.

#####The association between the field sample and the probability distribution is calculated using the range of median probabilities calculated within the conditional probability matrix for the first probability density function (condProb1). The key to interpreting the map and the median probabilities associated with each category explained in the table below. 


```r
m<-read.csv(file="C:/Users/Arora/Desktop/MapKey.csv")
m    
```

```
##    Color Median.Probability.Range Associated.Probability.Density.Function
## 1   Blue                  0.9-1.0                                       1
## 2  Green                  0.5-0.9                                       1
## 3    Red                  0.1-0.5                                       2
## 4 Yellow                  0.0-0.1                                       2
##   Degree.of.Association
## 1                Strong
## 2              Moderate
## 3              Moderate
## 4                Strong
```

```r
map(database = "state", regions = "colorado", fill = TRUE, col = "grey95", border = "white")
map.axes()
plotClusters(gcData, condProbs1, symbolSizes = rep.int(2/3, 4))
```
![Their mode map is mostly blue and red. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/Map_Mode.jpg)


![Like their map, mine is also mostly blue and red. This means both our points are either strongly associated with the first probability function or moderately associated with the second. ](https://github.com/geochemica/data-reanalysis-assignment/blob/master/my_mode_map.JPG)


##Works Cited

#####Ellefsen, Karl J. and David B. Smith 2016.  Manual hierarchical clustering of regional geochemical data using a Bayesian finite mixture model. Applied Jeochemistry 75: 200-2010.

#####Ellefsen, Karl J. and David B Smith 2016. User's guide for GcClust - An R package for clustering of regional geochemical data: U.S.Geological Survey Techniques and Methods 7-C13, 21 p., http://dx.doi.org/10.3133/tm7c13

#####Kabacoff, Robert I. 2015.  R in Action: Data analysis and graphics with R, Second Edition.Shelter Island, Manning. 

#####Pawlosky-Ghan, Vera, J.J. Egozcue, and Raimon Tolosana-Delgado 2012. Modelinng and analysis of compositional data. Chichester, john Wiley and Sons, Ltd.

#####Simplex algorith. Wikipedia, accessed December 7, 2017. https://en.wikipedia.org/wiki/Simplex_algorithm
