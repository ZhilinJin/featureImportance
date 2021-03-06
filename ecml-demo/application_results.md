Simulations and Application
================

# Introduction

In this short tutorial, we provide the code that reproduces the results
of the application section of our article entitled “Visualizing the
Feature Importance for Black Box Models”. We used
[`batchtools`](https://github.com/mllg/batchtools) to run our
experiments. The files
[`application_pi_simulation.R`](https://github.com/giuseppec/featureImportance/blob/master/ecml-demo/application_pi_simulation.R),
[`application_shapley_simulation.R`](https://github.com/giuseppec/featureImportance/blob/master/ecml-demo/application_shapley_simulation.R)
and
[`application_importance_realdata.R`](https://github.com/giuseppec/featureImportance/blob/master/ecml-demo/application_importance_realdata.R)
contain the `batchtools` code to reproduce the expermients and can be
found in this
[directory](https://github.com/giuseppec/featureImportance/tree/master/ecml-demo).
The directory also includes the results of both files in an `.Rds` file
which is used in the code below to produce the figures and tables.

# Load Packages

``` r
# load required packages
library(data.table)
library(ggplot2)
library(gridExtra)
library(ggExtra)
library(xtable)
library(knitr)
library(featureImportance)
source("helper/functions.R")
```

# Simulation

## PI and ICI Plots

``` r
res = readRDS("application_pi_simulation.Rds")

imp.global = rbindlist(lapply(res, function(x) {
  getImpTable(x, sort = FALSE)
}))

# compute conditional PFI based on feature V3 = 0
imp0 = rbindlist(lapply(res, function(x) {
  use0 = x[features == "V3" & feature.value == 0, unique(replace.id)]
  getImpTable(x, obs.id = use0, sort = FALSE)
}))

# compute conditional PFI based on feature V3 = 1
imp1 = rbindlist(lapply(res, function(x) {
  use1 = x[features == "V3" & feature.value == 1, unique(replace.id)]
  getImpTable(x, obs.id = use1, sort = FALSE)
}))

tab = rbind(
  imp.global[, lapply(.SD, pasteMeanSd)],
  imp0[, lapply(.SD, pasteMeanSd)],
  imp1[, lapply(.SD, pasteMeanSd)]
)
kable(tab)
```

| V1             | V2             | V3           |
| :------------- | :------------- | :----------- |
| 77.98 (14.15)  | 76.76 (13.89)  | 62.76 (7.82) |
| 152.49 (26.06) | 1.43 (1.32)    | 0            |
| 1.26 (1.03)    | 151.49 (24.69) | 0            |

``` r
pfi = res[[1]]

# Get index of observations for V3 = 0 and V3 = 1
use0 = pfi[features == "V3" & feature.value == 0, unique(replace.id)]
use1 = pfi[features == "V3" & feature.value == 1, unique(replace.id)]

vars = c("V1", "V2")
pi = lapply(vars, function(var) {
  pi.cond = conditionalPFI(pfi, var, use0, use1, group.var = "V3")
  ici = subset(pfi, features == var)
  ici[, V3 := as.factor(as.numeric(row.id %in% use1))]
  ind = gsub("[[:alpha:]]", "", var)
  
  plotImportance(pfi, feat = var, mid = "mse", hline = FALSE) +
    geom_line(data = pi.cond, aes(color = V3)) +
    geom_point(data = pi.cond, aes(color = V3)) +
    labs(x = bquote(X[.(ind)]), color = bquote(X[3])) +
    ylim(c(-10, 300))
})
marrangeGrob(pi, nrow = 1, ncol = 2)
```

![PI curves of \(X_1\) and \(X_2\) calculated using all observations
(black line) and conditional on \(X_3 = 0\) (red line) and \(X_3 = 1\)
(green line). The points plotted on the lines refer to the observed
feature values that were used as grid points to produce the
corresponding PI
curves.](application_results_files/figure-gfm/conditional-1.png)

## Shapley Feature Importance

``` r
res = readRDS("application_shapley_simulation.Rds")
# use shorter learner name
res[, learner := factor(gsub("regr.", "", learner))]
# compute ratio of the importance values w.r.t feature "V3"
res[, ratio := mse/mse[feature == "V3"], by = c("method", "learner", "repl")]

### Plot simulation results of all 500 repititions
shap = subset(res, method %in% c("pfi.diff", "pfi.ratio", "shapley") & feature != "V3")
new.names = setNames(expression(X[1]/X[3], X[2]/X[3]), c("V1", "V2"))

pp = ggplot(data = shap, aes(x = feature, y = ratio)) + 
  geom_boxplot(aes(fill = method), lwd = 0.2, outlier.size = 0.8) + 
  facet_grid(. ~ learner, scales = "free") +
  scale_fill_grey(labels = c("PFI (Diff.)", "PFI (Ratio)", "SFIMP"), start = 0.4, end = 0.9) +
  scale_x_discrete(labels = new.names) +
  labs(title = "(b) Simulation with 500 repetitions", 
    x = "Features involved to compute the ratio", y = "Value of the ratio")

### Plot example of an individual repetition (2nd replication)
shap2 = subset(res, repl == 2 & method %in% c("shapley", "geP"))
# reorder features for plotting
feat.order = c("V3", "V2", "V1", "geP")
shap2$feature = factor(shap2$feature, levels = feat.order)
# change sign
shap2[, mse := round(ifelse(method == "geP", mse, -mse), 2)]
# add column containing proportion of explained importance
shap2[, perc := ifelse(feature == "geP", NA, mse/sum(mse[feature != "geP"])), by = "learner"]
# add column containing drop in MSE + proportion of explained importance
shap2[, lab := ifelse(feature == "geP", mse, paste0(mse, " (", round(perc*100, 0), "%)"))]

col = c(gray.colors(3, start = 0.4, end = 0.9), hcl(h = 195, l = 65, c = 100))
col = setNames(col, feat.order)
legend = c("V1" = bquote(phi[1]), "V2" = bquote(phi[2]), 
  "V3" = bquote(phi[3]), "geP" = bquote(widehat(GE)[P]))

pp2 = ggplot(shap2, aes(x = learner, y = mse, fill = feature)) + 
  geom_bar(stat = "identity", colour = "white", pos = "stack") + 
  geom_text(aes(label = lab), position = position_stack(vjust = 0.5), size = 3) + 
  coord_flip() + 
  scale_fill_manual(values = col, name = " performance \n explained by", labels = legend) +
  labs(title = "(a) Comparing the model performance and SFIMP values across different models", 
    x = "", y = "performance (MSE)")

grid.arrange(pp2, pp, heights = c(3, 5))
```

![Panel (a) shows the results of a single run, consisting of sampling
test data and computing the importance on the previously fitted models.
The first numbers on the left refer to the model performance (MSE) using
all features. The other numbers are the SFIMP values which sum up to the
total explainable performance. The percentages refer to the proportion
of explained importance. Panel (b) shows the results of 500 repetitions
of the experiment. The plots display the distribution of ratios of the
importance values for \(X_1\) and \(X_2\) with respect to \(X_3\)
computed by SFIMP, by the difference-based PFI and by the ratio-based
PFI.](application_results_files/figure-gfm/shapley-1.png)

# Application on Real Data

## Produce Table

``` r
pfi = readRDS("application_importance_realdata.Rds")

# get index for LSTAT <= 10 in order to keep those observations
pi.ind = unique(pfi[features == "LSTAT" & feature.value <= 10, replace.id])
# compute integral of each ICI curve and select observations with positive ICI integral
ici = subset(pfi, features == "LSTAT")
ici.area = ici[, lapply(.SD, mean, na.rm = TRUE), .SDcols = "mse", by = "row.id"]
ici.ind = which(ici.area$mse > 0)

# produce table
imp = getImpTable(pfi)
imp.pi = getImpTable(pfi, pi.ind)
imp.ici = getImpTable(pfi, ici.ind)
kable(rbind(imp, imp.pi, imp.ici))
```

| LSTAT |   RM | NOX | DIS | CRIM | PTRATIO | AGE | TAX | INDUS | RAD |   B | CHAS |  ZN |
| ----: | ---: | --: | --: | ---: | ------: | --: | --: | ----: | --: | --: | ---: | --: |
|  32.0 | 15.6 | 3.9 | 2.7 |  2.6 |     2.2 | 1.2 | 1.0 |   1.0 | 0.8 | 0.8 |  0.1 | 0.1 |
|  10.4 | 29.6 | 1.5 | 3.3 |  0.8 |     2.3 | 0.8 | 1.2 |   0.5 | 1.1 | 0.6 |  0.2 | 0.2 |
|  35.3 | 17.0 | 4.3 | 2.4 |  2.5 |     2.5 | 1.1 | 0.8 |   1.2 | 0.9 | 0.8 |  0.1 | 0.1 |

## Produce PI and ICI plots

``` r
features = c("LSTAT", "RM")

pp = lapply(features, function(feat) {
  ici = subset(pfi, features == feat)
  ici.area = ici[, lapply(.SD, mean, na.rm = TRUE), .SDcols = "mse", by = "row.id"]
  ind = c(which.min(ici.area$mse), which.max(ici.area$mse))
  ici.obs = subset(ici, row.id %in% ici.area$row.id[ind])
  
  # PI plot
  pi.plot = plotImportance(pfi, feat, "mse") + theme(legend.position = "none")
  
  # ICI plot
  ici.plot = plotImportance(pfi, feat, "mse", individual = TRUE, grid.points = FALSE, hline = FALSE) +
    geom_line(data = ici.obs, aes(color = factor(row.id), group = row.id)) +
    theme(legend.position = "none")
  
  list(ggMarginal(pi.plot, type = "histogram", fill = "transparent", margins = "x"), ici.plot)
})
pp = unlist(pp, recursive = FALSE)
marrangeGrob(pp, nrow = 2, ncol = 2)
```

![PI and ICI plots for a random forest and the two most important
features of the Boston housing data (LSTAT and RM). The horizontal lines
in the PI plots represent the value of the global PFI (i.e. the integral
of the PI curve). Marginal distribution histograms for features are
added to the PI and ICI plot margins. The ICI curve with the largest
integral is highlighted in green and the curve with the smallest
integral in red.](application_results_files/figure-gfm/piplot-1.png)
