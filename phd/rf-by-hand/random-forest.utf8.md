---
title: "Random Forests by Hand"
output:
  rmarkdown::html_vignette:
    fig_width: 5
    fig_height: 3.5
    fig_cap: TRUE
    toc: yes
    css: style.css
biblio-style: "apalike"
link-citations: yes
bibliography: bibliography.bib
header-includes:
  - \usepackage{mathtools}
editor_options: 
  chunk_output_type: console
---

<style type="text/css">
#TOC {
  margin: 0 270px;
  width: 425px;
}
</style>
</style>
<div class="outer">
<img src="./logo.png"  width="150px" display="block" align="bottom">
</div>
<b>
<center>
<a href="https://brunaw.com/"> Bruna Wundervald </a><br/>
<code>brunadaviesw at gmail.com</code><br/>
PhD Candidate in Statistics, Hamilton Institute
</center>
</b>
</div>
</div>

---




# Introduction

Statistical machine learning is a field focused on predictions tasks. 
When the variable of interest already exists, we are dealing with a
supervised problem. The general idea is that we can predict the
response variable $Y$ based on the information brought by a set of covariables
$\mathbf{X}$. This is done by fitting a model that relates these predictors to the 
variable of interest, that can either be quantitative 
(continuous or discrete) or be a factor variable, with two or more classes. 


When the response is discrete, the prediction method is configured 
as classification method. Classical statistical learning methods
for classification are: logistic regression, support vector machines,
neural networks, decision trees and random forests (@HastieTrevor).
Classification problems emerge in the most different range of occasions, examples include:
predict whether someone would pay a bank loan or not based on their 
financial situation, predict in which party a person someone is more likely to
vote based on their political views, ect. 

As for the problems involving a quantitative response variable, we call them
regression problems. The response can represent infinite different 
measures, such as human height, human weight, the count of people in a
building each day, how long it takes for a bus to arrive at a bus 
stop, salaries for technology professions, and so on. Some methods
used for classification can also be applied in this cases, as neural
networks, regression trees and random forests. Other methods for
regression include least squares linear regression, survival regression, 
time series models and several more (@HastieTrevor). 


# Regression Trees

Decision trees are flexible non-parametric methods that be can
applied to both regression and classification problems. 
It is based on the estimation of a series of binary conditional 
statements (`if-else`) on the predictors space. As an illustration, 
let us get back to a classification problem demonstrated in Figure 1.
On the left we see the plot of two predictor variables $x_1$ and $x_2$ 
and the color represents the class to each data point belongs. 
Suppose the two classes are the name of the colors, "red" and "green". 
An optimal way of correctly classyfing the points is shown on the right 
plot. The rule is: if $x_1 <= 50$ and $x_2 <= 0.5$ or $x_1 > 50$ and $x_2 > 0.5$,
the point should be classified as "red"; if the point is not in any
of these two regions, it should be classified as "green". 


<div class="figure" style="text-align: center">
<img src="img/tree.png" alt="Figure 1: Illustration of a decision space" width="35%" /><img src="img/decision.png" alt="Figure 1: Illustration of a decision space" width="35%" />
<p class="caption">Figure 1: Illustration of a decision space</p>
</div>

Figure 1 also demonstrates an important characteristic of the trees: its 
ability to handle non-linear relationships between the predictor
variables and the response. 

Trees can also be shown as a graph. In Figure 2 we have a simple graph
of the rules of our illustration. In this case, the observations in 
terminal nodes 7 and 4 would be the "red" class, and the nodes
5 and 6 would be the "green" ones. 


<div class="figure" style="text-align: center">
<img src="img/graph.png" alt="Figure 2: The graph of the decision rules." width="75%" />
<p class="caption">Figure 2: The graph of the decision rules.</p>
</div>


For a classification task, the prediction for each division is a class, 
and the partitions of the predictor spare are chosen in order to
minimize the rate of incorrect classification. In the 
regression case, on the other hand, the prediction consists is a 
constant value, usually taken as the mean of $y$ in each region. The 
same way, the minimized measure is the residual sum of squares, given by

$$ RSS_{tree} =  \sum_{j = 1}^{J} \sum_{i \in R_j} (y_i - \hat y_{R_j})^2$$


where $\hat y_{R_j}$ is the mean response for the observations in the 
*j*th region of the predictors space. It is computationally too 
expensive to consider a combination of every possible partition 
of the feature space into $J$ regions, a approach called 
*recursive binary splitting* is taken. The algorithm is described
as 

  1. Select a predictor $X_j$,
  2. Select a respective cutpoint $s$ to split
  the feature space into regions $\{X|X_j < s \}$ and 
  $\{X|X_j \geq s \}$ that leads to the greatest reduction in the 
  $RSS_{tree}$, described by 
  
  $$ \sum_{i: x_i \in R_1 (j, s)} (y_i - \hat y_{R_1})^2 +
  \sum_{i: x_i \in R_2 (j, s)} (y_i - \hat y_{R_2})^2$$
  
  with  $\hat y_{R_1}$ being the mean response in $R_1 (j, s)$
  and $\hat y_{R_2}$ the mean response in $R_2 (j, s)$, 
  
  3. Break the feature space into the region and come back to 
  Step 1, taking in consideration the previous nodes. 

The process continues until a criterion is reached. Once the regions
are created, we predict the response for the observations as the 
means in each of the partitions. In the following we have a very simple
implementation of the three algorithm. Note that here we considered
that each node has to contain at least 5\% of the dataset, in order
to avoid having meaningless nodes. Besides that, we choose to use only
$m \approx \sqrt(p)$, p being the total of available features, in each
tree for reasons that will be explained in the next section. 

## R Code

The following `chunk` represents one way of implementing regression 
trees in R. The function receives the formula of the model and the
data and returns a list of the data with the prediction, the 
variables where the splits happened and the cutpoints.
That is not a perfect way and improvements can be done, as making
it generic to receives all types of predictors and to make it possible
for the splits repeat a variable. 



```r
library(tidyverse)

# Defining the function to create the minimizing criteria
sum_of_squares <- function(cut_point, variable_index, data_ss, X){
  
  # This function receives a variable and calculates the residual 
  # sum of squares if we partition the data accordingly to a cutpoint
  # in the chosen variable, considering all the pre-existent nodes
  
  cuts <- data_ss %>% 
    mutate(var = !!sym(colnames(X)[variable_index])) %>% 
    group_by(node) %>% 
    mutate(criteria = ifelse(var > cut_point, "set1", "set2")) %>%
    group_by(criteria) %>% 
    mutate(mean = mean(!!sym(colnames(data_ss)[1])), 
           sqr = (!!sym(colnames(data_ss)[1]) - mean)^2)  %>%
    group_by(node) %>% 
    summarise(sqr = sum(sqr)) %>% 
    arrange(sqr) %>% 
    slice(1) %>% 
    c()
} 



my_trees <- function(formula, your_data){
  
  # Taking some action if the data has too many NAs
  if(dim(na.omit(your_data))[1] < 10){
    stop("Your data has too many NAs.")
  }
  # Removing the intercept
  formula <- as.formula(paste(c(formula), "- 1"))
  
  # Extracting the model structure
  m <- model.frame(formula, data = your_data)
  response <- all.vars(formula)[1]
  
  # Defining the matrix of predictor variables
  X <- model.matrix(formula, m)
  colnames(X) <- colnames(m)[-1]
  
  # Defining the response
  y <- model.response(m)
  
  # Selecting m predictors 
  options <- 1:ncol(X)
  node_num <- round(sqrt(ncol(X))) + 1
  n_pred <- round(sqrt(ncol(X)))
  selected_predictors <- sample(options, size = n_pred, replace = FALSE)
  
  # Creating the matrix to save the criteria for the cuts
  criteria <- matrix(NA, ncol = 4, nrow = n_pred)
  
  # Creating the minimum of observations that should be in each node
  keep_node <- round(dim(your_data)[1]*0.05)
  
  # Creating the node for each observation
  m$node<- "node 1"
  
  
  # Defining auxiliar variables 
  counter <- 0
  var  <-  vector("numeric", n_pred)
  final_vars <-  c(0)
  cut_point <-  vector("numeric", n_pred)
  final_cuts <-  c(0)
  parents  <-  vector("numeric", n_pred)
  final_parents<-  c(0)
  
  temps <- vector("numeric", n_pred)
  iter <-  2

  for(j in 2:node_num){
    
    # Finding the minimun sum of squares by testing all variables and different
    # cutpoints
    for(i in 1:n_pred){
      
      # Selecting a variables
      variable <- selected_predictors[i]
      # Defining sequence of cutpoints to be tested
      cut_points <- c(X[, variable])
      
      # Mapping the RSS function to the cutpoints
      points <- cut_points %>% purrr::map(sum_of_squares, 
                                          variable_index = variable,
                                          data_ss = m, X = X) 
      
      # Saving the results
      sqrs <- points %>% map("sqr") %>% unlist()
      node_new <- points %>% map("node") %>% unlist()
      cut <- sqrs %>% which.min() %>% unlist()
      
      # Creating the criteria matrix to check for the variable that produced the
      #  smallest rss
      criteria[i, 1:4] <- c(i, cut_points[cut], sqrs[cut], node_new[cut])
    }  
    

    # Verifying which variable and cutpoint gave the best results
    min_sqr <- which.min(criteria[, 3])
    root <- selected_predictors[min_sqr]
    var[j-1] <- colnames(X)[root]
    cut_point[j-1] <- as.numeric(criteria[min_sqr, 2])
    parents[j-1] <- criteria[min_sqr, 4] 
    
    
    # Checking the dimension of the possible new nodes to see if it 
    # they have the minimum of observations
    temps[j] <- m %>% 
      filter(node == parents[j-1]) %>% 
      mutate(groups =  ifelse(!!sym(var[j-1]) > cut_point[j-1], 'set1', 'set2')) %>% 
      group_by(groups) %>% 
      count() %>% 
      ungroup() %>% 
      arrange(n) %>% slice(1) %>% pull()
    
    
    # Creating the very first split 
    if(j == 2 && temps[2] >= keep_node){
      
      m <- m %>% 
        mutate(node = ifelse(!!sym(var[j-1]) > cut_point[j-1],
                             paste(node, iter + 1), paste(node, iter)))

      counter <- counter + 1
      iter <-  iter + 1
      
      # Feeding the final vectors of variables and cutpoints
      final_vars[counter] <- var[j-1] 
      final_cuts[counter] <- cut_point[j-1] 
      final_parents[counter] <- parents[j-1]
      
    } else {
      # Creating the next splits and changing the auxiliar variables 
      if(temps[j] >= keep_node && temps[j] != temps[j-1]){
        m <- m %>% 
          mutate(node = ifelse(node == parents[j-1],
                               ifelse(!!sym(var[j-1]) > cut_point[j-1],
                                      paste(node, iter+counter+1), paste(node, iter+counter)), 
                               paste(node)))
        counter = counter + 1
        iter = iter + 1
        
        final_vars[counter] <- var[j-1] 
        final_cuts[counter] <- cut_point[j-1] 
        final_parents[counter] <- parents[j-1]
        
      } else {
        counter = counter
        iter = iter
      }
    }
    
    # Updating objects 
    selected_predictors <- selected_predictors[-min_sqr]
    n_pred <- n_pred - 1
    criteria <- matrix(NA, ncol = 4, nrow = n_pred)
    j <- j + 1
    
  }
  
  # Calculating the mean for each final node  
  m <- m %>% 
    group_by(node) %>% 
    mutate(prediction = round(mean(!!sym(response)), 3))
  
  # Calculating the means
  means <- m %>% 
    group_by(node) %>% 
    summarise(means = round(mean(!!sym(response)), 3)) %>% 
    pull(means)
  
    nodes <- m %>% distinct(node)
  
  return(list(model_data = m, 
              vars = final_vars, 
              cuts = as.numeric(final_cuts),
              parents = final_parents,
              means = means, 
              nodes = nodes))
  
}
```

### Example: the `mtcars` data 

The `mtcars` data was extracted from the 1974 Motor Trend US magazine,
and comprises fuel consumption and 10 aspects of automobile design 
and performance for 32 automobiles (1973–74 models). Here we try to 
predict miles per gallon using as covariables the 
number of cylinders, displacement, gross horsepower, rear axle ratio,
weight, 1/4 mile time, transmission, number of forward gears and
number of carburetors for each car model. 


```r
# Using the function
tree_cars <- my_trees(formula = mpg ~ cyl + disp + hp + drat +
                        wt + qsec + am + gear + carb, 
                      mtcars)
tree_cars
```

```
## $model_data
## # A tibble: 32 x 12
## # Groups:   node [3]
##      mpg   cyl  disp    hp  drat    wt  qsec    am  gear  carb node 
##    <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <dbl> <chr>
##  1  21       6  160    110  3.9   2.62  16.5     1     4     4 node…
##  2  21       6  160    110  3.9   2.88  17.0     1     4     4 node…
##  3  22.8     4  108     93  3.85  2.32  18.6     1     4     1 node…
##  4  21.4     6  258    110  3.08  3.22  19.4     0     3     1 node…
##  5  18.7     8  360    175  3.15  3.44  17.0     0     3     2 node…
##  6  18.1     6  225    105  2.76  3.46  20.2     0     3     1 node…
##  7  14.3     8  360    245  3.21  3.57  15.8     0     3     4 node…
##  8  24.4     4  147.    62  3.69  3.19  20       0     4     2 node…
##  9  22.8     4  141.    95  3.92  3.15  22.9     0     4     2 node…
## 10  19.2     6  168.   123  3.92  3.44  18.3     0     4     4 node…
## # … with 22 more rows, and 1 more variable: prediction <dbl>
## 
## $vars
## [1] "hp"   "drat"
## 
## $cuts
## [1] 113.00   3.73
## 
## $parents
## [1] "node 1"   "node 1 3"
## 
## $means
## [1] 24.987 15.379 17.600
## 
## $nodes
## # A tibble: 3 x 1
## # Groups:   node [3]
##   node      
##   <chr>     
## 1 node 1 2  
## 2 node 1 3 4
## 3 node 1 3 5
```

```r
# Finding the rss
sum((mtcars$mpg - tree_cars$model_data$prediction)^2)
```

```
## [1] 437.0209
```


### Predicting for the tree

The next `chunk` shows a function that predicts for the previous model. 
A random sample from the `mtcars` was selected to demonstrate its use.  


```r
predict_tree <- function(model, newdata){
  data <- newdata
  vars <- model$vars
  cuts <- model$cuts
  nodes <- model$nodes
  parents <- model$parents
  means <- data.frame(node = nodes,
                      prediction = model$means)
  
  data$node <- "node 1"
  
  counter <- 0
  iter <-  2
  i = 2
  
  for(i in 2:(length(vars)+1)){
    
    if(i == 2){
      
      data <- data %>% 
        mutate(node = ifelse(!!sym(vars[i-1]) > cuts[i-1],
                             paste(node, iter + 1), paste(node, iter)))
      
      counter <- counter + 1
      iter <-  iter + 1
      
    } else {
      # Creating the next splits and changing the auxiliar variables 
      data <- data %>% 
        mutate(node = ifelse(node == parents[i-1],
                             ifelse(!!sym(vars[i-1]) > cuts[i-1],
                                    paste(node, iter+counter+1), paste(node, iter+counter)), 
                             paste(node)))
      counter = counter + 1
      iter = iter + 1
    }
  }
  
  
  
  predicted <- data %>% 
    full_join(means, by = "node") %>% 
    pull(prediction)
  
  return(predicted)
}

# Sampling from the mtcars data
new_data <- mtcars %>% 
  sample_n(20)

predictions <- predict_tree(tree_cars, newdata = new_data)
predictions
```

```
##  [1] 15.379 24.987 15.379 24.987 24.987 24.987 24.987 15.379 15.379 24.987
## [11] 24.987 24.987 15.379 15.379 24.987 17.600 24.987 17.600 15.379 24.987
```



# Random Forests

Decision trees usually have high variance. A natural way to reduce
variance in a model is to take many training sets from the population,
build a separate prediction for each training set and average the 
resulting predictions, forming a equation like

$$ \hat f_{avg}(x) = \frac{1}{B} \sum_{b=1}^{B} \hat f^b(x),$$

where B is the number of training sets and $\hat f^b(x)$ are the
estimated functions. With that, even if each model has high variance, 
their average will have the variance reduced. One way of producing this 
many training sets for trees is using bootstrap (@Hesterberg2011). We
construct B regression trees using B bootstrapped sets and average the
predictions. This method is called bagging (@HastieTrevor). 

Random forests (@Breiman2001) are a big combination of trees that 
improves the bagging method by decorrelating the trees. We still 
build trees on bootstrapped samples, but for each tree, only 
a random sample of predictors is considered as candidates variables. 
The size of this sample is usually taken as 
$m \approx \sqrt(p)$, with $p$ being the total of available features. By
doing this simple change, we prevent that strong predictor variables would 
always appear on the top of the trees, since sometimes they will not
even be considered for the split. The average of the trees will be 
less variable and hence more reliable (@HastieTrevor). 


## R Code

The following code implements a function that produces a random forest.
As for the trees, this function can be improved in various points. 


```r
library(tidyverse)
library(parallel)
options(warn=-1)


rf <- function(formula, rf_data, number_of_trees = 15){

  # Paralellizing all the trees 
  result <- mclapply(1:number_of_trees, FUN = function(i, formula, rf_data){
      trees <- my_trees(formula = formula, your_data = rf_data)
      Sys.sleep(0.2)
      return(trees)
    }, rf_data = rf_data, formula = formula)

  # Calculating predictions by extracting it from the list of trees
  predictions <- result %>% map("model_data") %>% 
    map(`[`, c("prediction")) %>% 
    bind_cols() %>% 
    mutate(final_prediction = rowSums(.)/number_of_trees) %>% 
    pull(final_prediction)
  
  # Returning results 
  return(list(trees_result = result, prediction = predictions))
}
```



### Example: the `mtcars` data 

This is the same problem as considered before, but now using the 
`rf` function to fit a random forest model.


```r
# Using the rf function
rf_cars <- rf(formula = mpg ~ cyl + disp + hp + drat +
                        wt + qsec + am + gear + carb, 
              rf_data = mtcars)

# Finding the rss
sum((mtcars$mpg - rf_cars$prediction)^2)
```

```
## [1] 194.5121
```

In the following we have the function that predicts a response vector 
for the random forest model. The new data used is the same as for the 
tree model.


```r
predict_rf <- function(model, newdata){
  
  number_of_trees <- lengths(model)[1]
  prediction <- model$trees_result %>% 
    map(predict_tree, newdata = newdata) %>% 
    as.data.frame() %>% 
    mutate(final_prediction = rowSums(.)/number_of_trees) %>% 
    pull(final_prediction)
  
  return(prediction)
}

predict_rf(rf_cars, new_data)
```

```
##  [1] 23.28560 23.05827 23.61227 23.01400 22.68840 22.68840 22.68840
##  [8] 23.28560 23.28560 23.01400 22.68840 24.58187 23.28560 23.28560
## [15] 27.20760 21.08840 24.58187 21.08840 25.56880 23.05827
```

By varying the number of trees in the random forest, we can check what is 
the residual sum of squares for each value. Figure 3 shows results for 
different numbers of trees. The plot tells us that a small number of trees
(around 15) produces as good results as a big amount of trees. 


```r
# Evaluating the number of trees
seq_number_trees <- seq(2, 50, by = 5)

sequence_rf <- seq_number_trees %>% map(rf, formula = mpg ~ cyl + disp + hp + drat +
                                   wt + qsec + am + gear + carb, rf_data = mtcars)
```


```r
rss <- function(prediction) sum((mtcars$mpg - prediction)^2)

results <- sequence_rf %>% 
  map("prediction") %>% 
  map(rss) %>% 
  map_df(data.frame) %>% 
  setNames("rss") %>% 
  mutate(n = seq_number_trees)

results %>% 
  ggplot(aes(n, rss)) +
    geom_line(colour = '#c03728', linetype = 'dotted') +
  geom_point(colour = '#919c4c', size = 3) +
  labs(x = "Number of trees", y = "Residual sum of squares",
       title = "Results for the random forests applied to the mtcars data") +
  theme_bw()
```

<div class="figure" style="text-align: center">
<img src="random-forest_files/figure-html/unnamed-chunk-9-1.png" alt="Figure 3: Results of the residual sum of squares for different numbers of trees"  />
<p class="caption">Figure 3: Results of the residual sum of squares for different numbers of trees</p>
</div>


# Conclusions and final remarks

For this project, the task was to implement a full random forest model. To 
do that, the first task was to implement the decision trees, 
once a random forest is a big combination  of them. After that, it is
necessary to create a big set of trees and use their average to produce
a random forest. The code shown here implements a fit function and
prediction function for both the trees and random forests models. 

This code is definitely not final and improvements should be made. First, 
the decision tree function needs to be faster and allow for the tree
to split the same variable more than once in the model. With that, the
random forest model will also be more efficient and correct. The complexity
parameter of the tree, which was not discussed here, is a fundamental 
detail that needs to be included. The prediction functions could use a 
little more of efficiency too. 


# References
