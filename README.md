# Library_sampling
Simulating sampling from a library, such as during the recombination step with the landing pad cells


Simulating library recombination
================
KAM
8/8/2021

``` r
nudt15 <- read.csv(file = "Data/NUDT15_missense.csv", header = TRUE, stringsAsFactors = FALSE)
nudt15 <- rbind(c("WT","WT",round(sum(nudt15$count)*0.3,0)),nudt15)  ## Because there were about 25% WTs
nudt15$count <- as.integer(nudt15$count)
nudt15$freq <- nudt15$count / sum(nudt15$count)
nudt15$zscore <- (nudt15$freq - mean(nudt15$freq))/sd(nudt15$freq)

## Bring in the PTEN data
pten <- read.csv(file = "Data/PTEN_library_distribution.csv", header = TRUE, stringsAsFactors = TRUE)
colnames(pten) <- c("row_number","variant","freq")

#pten_missense <- subset(pten, variant != "_remaining")[,c("variant","freq")]
pten_missense <- pten
pten_missense$freq <- as.numeric(pten_missense$freq)
pten_missense$freq <- pten_missense$freq / sum(pten_missense$freq)
pten_missense$zscore <- (pten_missense$freq - mean(pten_missense$freq))/sd(pten_missense$freq)

Fraction_of_uniform_distribution_plot <- ggplot() + theme_bw() + scale_x_log10() +
  labs(x = "Normalized to uniform distribution frequency", y = "Density", title = "Black is PTEN, red is NUDT15") +
  geom_density(data = pten_missense, aes(x = freq/(1/nrow(pten_missense)))) + 
  geom_density(data = nudt15, aes(x = freq/(1/nrow(nudt15))), color = "red") 
ggsave(file = "Plots/Fraction_of_uniform_distribution_plot.png", Fraction_of_uniform_distribution_plot, height = 3, width = 4)
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 3101 rows containing non-finite values (stat_density).

    ## Warning: Removed 34 rows containing non-finite values (stat_density).

``` r
Fraction_of_uniform_distribution_plot
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 3101 rows containing non-finite values (stat_density).

    ## Warning: Removed 34 rows containing non-finite values (stat_density).

![](Simulating_library_sampling_files/figure-gfm/Imporating%20the%20NUDT15%20and%20PTEN%20library%20Illumina%20data%20and%20getting%20a%20z-score-1.png)<!-- -->

``` r
## CV for NUDT15
sd(nudt15$freq) / mean(nudt15$freq)
```

    ## [1] 15.94276

``` r
## CV for PTEN
sd(pten_missense$freq) / mean(pten_missense$freq)
```

    ## [1] 39.86653

``` r
nudt15_uniq_number <- length(unique(nudt15$variant))
nudt15_sampling_frame <- data.frame("cells" = c(1e4,3e4,1e5,3e5,1e6,3e6), "fraction_observed" = 0)
for(x in 1:nrow(nudt15_sampling_frame)){
  nudt15_sampling <- sample(nudt15$variant, nudt15_sampling_frame$cells[x], prob = nudt15$freq, replace = T)
  nudt15_sampling_df <- data.frame(table(nudt15_sampling))
  nudt15_sampling_df2 <- subset(nudt15_sampling_df, Freq >= 1) # Whatever cutoff you want to use
  nudt15_sampling_frame$fraction_observed[x] <- nrow(nudt15_sampling_df2) / nudt15_uniq_number
}


pten_library_number <- length(unique(subset(pten_missense, freq > 0)$variant))

pten_sampling_frame <- data.frame("cells" = c(1e4,3e4,1e5,3e5,1e6,3e6), "fraction_observed" = 0)
for(x in 1:nrow(pten_sampling_frame)){
  pten_sampling <- sample(pten_missense$variant, pten_sampling_frame$cells[x], prob = pten_missense$freq, replace = T)
  pten_sampling_df <- data.frame(table(pten_sampling))
  pten_sampling_df2 <- subset(pten_sampling_df, Freq >= 1) # Whatever cutoff you want to use
  pten_sampling_frame$fraction_observed[x] <- nrow(pten_sampling_df2) / pten_library_number
}

real_summary_frame <- data.frame("cells" = pten_sampling_frame$cells, "pten" = pten_sampling_frame$fraction_observed, "nudt15" = nudt15_sampling_frame$fraction_observed)

real_summary_frame_melted <- melt(real_summary_frame, id = "cells")
colnames(real_summary_frame_melted)[2] <- "library"

nudt15_pten_comparison_plot <- ggplot() + theme_bw() +  theme(legend.position = "top") +
  labs(y = "Fraction of possible variants\nrecombined at least once", x = "Number of cells recombined") +
  scale_x_log10(expand = c(0,0.01)) + scale_y_continuous(limits = c(0.5,1), expand = c(0,0.01)) + scale_colour_manual(values = c("black", "red")) +
  geom_line(data = real_summary_frame_melted, aes(x = cells, y = value, color = library)) +
  geom_point(data = real_summary_frame_melted, aes(x = cells, y = value, color = library))
ggsave(file = "Plots/Real_data_nudt15_pten_comparison_plot.png", nudt15_pten_comparison_plot, height = 3, width = 4)
nudt15_pten_comparison_plot
```

![](Simulating_library_sampling_files/figure-gfm/Simulating%20sampling%20the%20library%20from%20the%20real%20plasmid%20Illumina%20data-1.png)<!-- -->

``` r
nudt15_2 <- subset(nudt15, freq > 0)
nudt15_2$log10 <- log10(nudt15_2$freq)

nudt15_fit <- fitdistr(nudt15_2$freq, "log-normal")
nudt15_para <- nudt15_fit$estimate
print(paste("NUDT15 meanlog:",as.character(round(nudt15_para[1],2))))
```

    ## [1] "NUDT15 meanlog: -8.92"

``` r
print(paste("NUDT15 sdlog:",as.character(round(nudt15_para[2],2))))
```

    ## [1] "NUDT15 sdlog: 0.71"

``` r
nudt15_recreation2 <- data.frame("value" = rlnorm(n = 4775, meanlog = nudt15_para[1], sdlog = nudt15_para[2]))
#nudt15_recreation <- data.frame("value" = rnorm(n = 4775, mean = nudt15_para[1], sd = nudt15_para[2]))
nudt15_recreation2$variant = seq(1,nrow(nudt15_recreation2))

nudt15_fitted_distribution_plot <- ggplot() + theme_bw() + labs(title = "NUDT15: Black in real, Red is recreated") + scale_x_log10() +
  geom_density(data = nudt15, aes(x = freq), color = "black") +
  geom_density(data = nudt15_recreation2, aes(x = value), color = "red")
nudt15_fitted_distribution_plot
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 34 rows containing non-finite values (stat_density).

![](Simulating_library_sampling_files/figure-gfm/Trying%20to%20recreate%20the%20distribution%20using%20a%20log-normal%20approximation-1.png)<!-- -->

``` r
  ggsave(file = "Plots/nudt15_fitted_distribution_plot.png", nudt15_fitted_distribution_plot, height = 3, width = 4)
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 34 rows containing non-finite values (stat_density).

``` r
pten_fit <- fitdistr(subset(pten_missense, freq > 0)$freq, "log-normal")
pten_para <- pten_fit$estimate
print(paste("PTEN meanlog:",as.character(round(pten_para[1],2))))
```

    ## [1] "PTEN meanlog: -9.58"

``` r
print(paste("PTEN sdlog:",as.character(round(pten_para[2],2))))
```

    ## [1] "PTEN sdlog: 0.99"

``` r
pten_recreation2 <- data.frame("value" = rlnorm(n = 4939, meanlog = pten_para[1], sdlog = pten_para[2]))
pten_recreation2$variant = seq(1,nrow(pten_recreation2))

pten_fitted_distribution_plot <- ggplot() + theme_bw() + labs(title = "PTEN: Black in real, Red is recreated") + scale_x_log10() +
  geom_density(data = pten_missense, aes(x = freq), color = "black") +
  geom_density(data = pten_recreation2, aes(x = value), color = "red")
pten_fitted_distribution_plot
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 3101 rows containing non-finite values (stat_density).

![](Simulating_library_sampling_files/figure-gfm/Trying%20to%20recreate%20the%20distribution%20using%20a%20log-normal%20approximation-2.png)<!-- -->

``` r
ggsave(file = "Plots/pten_fitted_distribution_plot.png", pten_fitted_distribution_plot, height = 3, width = 4)
```

    ## Warning: Transformation introduced infinite values in continuous x-axis

    ## Warning: Removed 3101 rows containing non-finite values (stat_density).

``` r
nudt15_sampling_frame <- data.frame("cells" = c(1e4,3e4,1e5,3e5,1e6,3e6), "fraction_observed" = 0)
for(x in 1:nrow(nudt15_sampling_frame)){
  nudt15_sampling <- sample(nudt15_recreation2$variant, nudt15_sampling_frame$cells[x], prob = nudt15_recreation2$value, replace = T)
  nudt15_sampling_df <- data.frame(table(nudt15_sampling))
  nudt15_sampling_df2 <- subset(nudt15_sampling_df, Freq >= 1)
  nudt15_sampling_frame$fraction_observed[x] <- nrow(nudt15_sampling_df2) / nrow(nudt15_recreation2)
}

pten_sampling_frame <- data.frame("cells" = c(1e4,3e4,1e5,3e5,1e6,3e6), "fraction_observed" = 0)
for(x in 1:nrow(pten_sampling_frame)){
  pten_sampling <- sample(pten_recreation2$variant, pten_sampling_frame$cells[x], prob = pten_recreation2$value, replace = T)
  pten_sampling_df <- data.frame(table(pten_sampling))
  pten_sampling_df2 <- subset(pten_sampling_df, Freq >= 1)
  pten_sampling_frame$fraction_observed[x] <- nrow(pten_sampling_df2) / nrow(pten_recreation2)
}

summary_frame <- data.frame("cells" = pten_sampling_frame$cells, "pten" = pten_sampling_frame$fraction_observed, "nudt15" = nudt15_sampling_frame$fraction_observed)

summary_frame_melted <- melt(summary_frame, id = "cells")
colnames(summary_frame_melted)[2] <- "library"

nudt15_pten_comparison_plot <- ggplot() + 
  theme_bw() + theme(legend.position = "top") +
  labs(y = "Fraction of possible variants\nrecombined at least once", x = "Number of cells recombined") + 
  scale_x_log10(expand = c(0,0.01)) + scale_y_continuous(limits = c(0.5,1), expand = c(0,0.01)) + scale_colour_manual(values = c("black", "red")) +
  geom_line(data = summary_frame_melted, aes(x = cells, y = value, color = library)) +
  geom_point(data = summary_frame_melted, aes(x = cells, y = value, color = library))
ggsave(file = "Plots/Recreated_data_nudt15_pten_comparison_plot.png", nudt15_pten_comparison_plot, height = 3, width = 4)
nudt15_pten_comparison_plot
```

![](Simulating_library_sampling_files/figure-gfm/Simulating%20sampling%20the%20library%20from%20the%20recreation-1.png)<!-- -->
