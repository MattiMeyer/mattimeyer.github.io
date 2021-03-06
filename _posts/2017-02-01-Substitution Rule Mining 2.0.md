---
layout: post
title: Substitution Rule Mining 2.0
subtitle: Better performance
---
In my [last post](http://mattimeyer.github.io/2016-12-21-Substitution-Rule-Mining/) I've introduced an implementation of a Substitution Rule Mining algorithm. Now I've developed a new version for better performance. Thus I used the purrr and tibble package from the tidyverse. Tibbles need much less memory like normal data frames and the purrr package contains fast loop functions that can also specify their outputs. If you want to learn more about purrr, Jenny Bryan developed [this great tutorial](https://jennybc.github.io/purrr-tutorial/). 

Furthermore I've deleted the part of the algorithm that searched for combination of several subsitution products which was included in the old version and is mentioned in the article by Teng, Hsieh and Chen (2002). In practive I didn't found good combination with it and I think it was more interesting to look for substitution for specific products.

```r
SRM <- function(TransData, MinSup, MinConf, pMin, pChi, itemLabel, nTID){
  
  # Packages ----------------------------------------------------------------
  
  if (sum(search() %in% "package:arules") == 0) {
    stop("Please load package arules")
  }  
  
  if (sum(search() %in% "package:purrr") == 0) {
    stop("Please load package purrr")
  }
  
  if (sum(search() %in% "package:dplyr") == 0) {
    stop("Please load package dplyr")
  }
  
  if (sum(search() %in% "package:tibble") == 0) {
    stop("Please load package tibble")
  }
  
  # Checking Input data -----------------------------------------------------
  if (missing(TransData)) {
    stop("Transaction data is missing")
  }
  
  if (is.numeric(nTID) == FALSE) {
    stop("nTID has to be one numeric number for the count of Transactions")
  }
  
  if (length(nTID) > 1) {
    stop("nTID has to be one number for the count of Transactions")
  }
  
  if (is.character(itemLabel) == FALSE) {
    stop("itemLabel has to be a character")
  }
  # Adding Complements ---------------------------------------------------
  
  compl_trans <- addComplement(TransData,labels = itemLabel)
  # matrix for support
  compl_tab <- crossTable(compl_trans,"support")
  
  # Concrete Itemsets ---------------------------------------------------------------
  
  
  # first loop for one item
  complement_data <- map_df(c(1 : (length(itemLabel) - 1)), function(i){
  
    # second loop combines it with all other items
    map_df((i + 1) : length(itemLabel), function(u){
      
      
      # getting chi value from Teng
      a <-  itemLabel[i]
      b <-  itemLabel[u]
      ca <- paste0("!", itemLabel[i])
      cb <- paste0("!", itemLabel[u])
      
      chiValue <- nTID * (
        compl_tab[ca, cb] ^ 2 / (compl_tab[ca, ca] * compl_tab[cb, cb]) +
          compl_tab[ca, b] ^ 2 / (compl_tab[ca, ca] * compl_tab[b, b]) +
          compl_tab[a, cb] ^ 2 / (compl_tab[a, a] * compl_tab[cb, cb]) +
          compl_tab[a, b] ^ 2 / (compl_tab[a, a] * compl_tab[b, b]) - 1)
      
      
      
      # conditions to be dependent
      try({
        if (compl_tab[a, b] > compl_tab[a, a] * compl_tab[b, b] && chiValue >= qchisq(pChi, 1) && 
            compl_tab[a, a] >= MinSup && compl_tab[b, b] >= MinSup ) {
          
          tibble(X = a,
                 Y = b)
          
        } })
      
    })
  })
  
  if (nrow(complement_data) == 0) {
    stop("No complement item sets could have been found")
  }
  
  
  #  changing mode  
  complement_data$X <- as.character(complement_data$X)
  complement_data$Y <- as.character(complement_data$Y)
  
  
  
  # Correlation -------------------------------------------------------------
  
  
  # mixing all combinations of concrete Itemsets
  combis <- unique(c(complement_data$X, complement_data$Y))
  
  rules <- map_df( 1 : (length(combis) - 1), function(i) {
    # second loop combines it with all other items
    map_df((i + 1) : length(combis), function(u) {
      
      first <- combis[i]
      second <- combis[u]
      
      # correlation
      corXY <- (compl_tab[first, second] - (compl_tab[first, first] * compl_tab[second, second])) /
        (sqrt((compl_tab[first, first] * (1 - compl_tab[first,first])) *
                (compl_tab[second, second] * (1 - compl_tab[second, second]))))
      
      
      # confidence
      conf1 <- compl_tab[first, paste0("!", second)] / compl_tab[first, first]
      conf2 <- compl_tab[second, paste0("!", first)] / compl_tab[second, second]
      
      two_rules <- tibble(
        Substitute = c(paste("{", first, "}"), 
                       paste("{", second, "}")),
        Product = c(paste("=>", "{", second, "}"),
                    paste("=>", "{", first, "}")),
        Support = c(compl_tab[first, paste0("!", second)], compl_tab[second, paste0("!", first)]),
        Confidence = c(conf1, conf2),
        Correlation = c(corXY, corXY)
      )
      
      # conditions
      try({
        if (two_rules$Correlation[1] < pMin) {
          
          # checking if both rules fulfill conditions
          if(two_rules$Support[1] >= MinSup && two_rules$Confidence[1] >= MinConf &&
             two_rules$Support[2] >= MinSup && two_rules$Confidence[2] >= MinConf) {
            
            bind_rows(two_rules[1, ],two_rules[2, ])
            
          } else if(two_rules$Support[1] >= MinSup && two_rules$Confidence[1] >= MinConf &&
                    two_rules$Support[2] <= MinSup && two_rules$Confidence[2] <= MinConf) {
            
            two_rules[1, ]
            
          } else if(two_rules$Support[1] <= MinSup && two_rules$Confidence[1] <= MinConf &&
                    two_rules$Support[2] >= MinSup && two_rules$Confidence[2] >= MinConf) {
            
            two_rules[2, ]
            
          }
          
        }
        
      })
      
    })
  })
  
  
  
  if (nrow(rules) == 0) {
    message("Sorry no rules could have been calculated. Maybe change input conditions.")
  } else {
    return(rules)
  }
  
  # end
}



```

