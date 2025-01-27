## Text Tutorial

* So far we've learned about time series objects, seasonal and long-term signals, and the influence of the past on current observations
* Take all that information and turn it into models
* Let's first load the packages we'll need for today

```r
install.packages("fpp3", repos = "http://cran.us.r-project.org")

library(fpp3)
library(tsibble) # convert time-series data in a tsibble
library(fable) # main package for modeling and forecasting with time-series data
library(feasts) # time-series data visualization
library(dplyr) # data manipulation
```

* Then load our data
data = read.csv('portal_timeseries.csv', stringsAsFactors = FALSE)
data$date<-as.Date(data$date, format = "%m/%d/%y")
data = dplyr::mutate(data, month=tsibble::yearmonth(date))
data_ts <-tsibble::as_tibble(data, index = month)

# Example conversion if not working 
df <- as.data.frame(data_ts)# back into dataframe with month as an addition
data_ts <- as_tsibble(df, index = month) # re-indexes month 
dplyr::select(data_ts,c(-date,-X)) #removes date and X column
autoplot(data_ts, NDVI)+geom_hline(yintercept = 0.18)


* We're going to be working with the NDVI data
* Reminder ourselves that that looks like

```r
gg_tsdisplay(data_ts, NDVI)
```

### White noise model

* We'll start with the simplest time-series model possible - white noise
* The data is normally distributed with a fixed mean and variance
* It takes the form

  > `y_t = c + e_t, where e_t ~ N(0, sigma)`

* So each time step in our model is a random draw from a normal distribution with a mean of `c`
* We fit time-series models using the `fable` package
* This model structure is provided by the `MEAN()` function

```r
MEAN()
```

* This output tells us that it is a model definition
* To fit that general model structure to our data we use the `model()` function

```r
avg_model = model(data_ts, MEAN(NDVI))
```

* We can then look at the resulting model information using the `report()` function

```r
report(avg_model)
```

* This shows us that the model has a white noise structure (indicated by `MEAN`), a mean value of 0.1791, and a variance of 0.0031
* To visualize the model with the data we have to first make the fitted values available using `augment()`

```r
avg_model_aug <- augment(avg_model)
avg_model_aug
```

* We can see that this produces a `tsibble` that includes month, NDVI, the fitted values from the model, & the model residuals
* Use `autoplot()` to look at the data and model together
* The predicted values from the model are stored in a special columns `.fitted`

```r
autoplot(avg_model_aug, NDVI) + autolayer(avg_model_aug, .fitted, color = "red")
```

* This simple model doesn't work very well
* There is clearly autocorrelation and seasonality in the time-series
* We can look at this directly by plotting the residuals and looking at their autocorrelation
* Our model assumes that the residuals are normally distributed and independent

```r
gg_tsresiduals(avg_model)
```

* We see the same autocorrelation structure as the original time-series, because we didn't do anything to model it
* We'll address that next, but first

> You do:
> * Fit a white noise model to the `rain` data
> * Plot your data with the model fit on top
> * Plot the residuals

## AR models

* Let's build a model that takes the autocorrelation into account
* Remember that we have lag 1 and lag 2 autocorrelation plus a season signal

```r
gg_tsdisplay(data_ts, NDVI)
```

* Let's start with just the lag 1 and lag 2 autocorrelation
* Use an "autoregressive" or AR model
* Current value depends on past values
* The simplest version of this type of model is an AR1 model

> *leave room to add y_t-2*

> `y_t = c + b_1 * y_t-1 + e_t, where e_t ~ N(0, sigma)`

* c is a constant, like the intercept in regression
* b_1 is a coefficient determining how y_t is related to y at a 1 time-step lag, i.e., the previous time step
* e_t is normally distributed error

* Does this model remind you of a biological model?
* This model is basically a Gompertz population model if y is log(N)
* The idea is that the current value influences the future values
* Makes a lot of sense for things like population dynamics

* Since we have also have lag 2 autocorrelation we add a term for two time steps back

> `y_t = c + b1 * y_t-1 + b2 * y_t-2 + e_t, where e_t ~ N(0, sigma)`

> Instructors note: Actually `y_t = (1 - b1 - b2) * c + b1 * y_t-1 + b2 * y_t-2 + e_t` due to non-zero mean 

* This type of model structure is available in `fable`'s `AR()` model
* If we want to specify how many autoregressive terms to include we specify the model as an R formula

```r
ar_model = model(data_ts, AR(NDVI ~ order(2)))
```

* The `order()` function lets us specify how many lags to include
* So an AR1 model would have `order(1)`
* We've written an AR2 model
* Let's look at the model using `report()`

```r
report(ar_model)
```

* There is a large, positive, ar1 value (b_1)
* So if NDVI was high at the previous time step it's expected to be high at the current time step
* There is a smaller, negative, ar2 value (b_2)
* So if NDVI was high two time steps back, it's expected to be lower at the current time step

```r
ar_model_aug = augment(ar_model)
ar_model_aug
```

* Note there aren't predictions for the first two time-steps
* Not possible because there are no y values before March 1992 for the model to use for prediction

```r
autoplot(ar_model_aug, NDVI) + autolayer(ar_model_aug, .fitted, color = "orange")
```

* This looks a lot better
* Let's take a look at the residuals

```r
gg_tsresiduals(ar_model)
```

* The residuals look better
* We successfully removed the short time-scale autocorrelation
* But the season signal is still present
* We'll work on that next time

> You do:
> * Fit an AR1 model to the `rain` data
> * Plot your data with the model fit on top
> * Plot the residuals

* How do the residuals look?
* Why do you think there might be a stronger two year autoregressive component that the one year component?
