tfruns: Track, Visualize, and Manage Training Runs
================

Overview
--------

<img src="images/view_run.png" align="right" width=220/>

The **tfruns** package provides a suite of tools for managing TensorFlow training runs and experiments from R:

-   Track the hyperparameters, metrics, output, and source code of every training run.

-   Compare hyperparmaeters and metrics across runs to find the best performing model.

-   Automatically generate reports to visualize individual training runs or comparisons between runs.

-   No changes to source code required (run data is automatically captured for all Keras and TF Estimator models).

Installation
------------

You can install the **tfruns** package from GitHub as follows:

``` r
devtools::install_github("rstudio/tfruns")
```

The package is intended to be used with the [keras](https://tensorflow.rstudio.com/keras) and/or the [tfestimators](https://tensorflow.rstudio.com/keras) packages, both of which provide higher level interfaces to TensorFlow from R. These packages can be installed with:

``` r
# keras
install.packages("keras")

# tfestimators
devtools::install_github("rstudio/tfestimators")
```

Training
--------

In the following sections we'll describe the various capabilities of **tfruns**. Our example training script ([mnist\_mlp.R](https://github.com/rstudio/tfruns/blob/master/inst/examples/mnist_mlp/mnist_mlp.R)) trains a Keras model to recognize MNIST digits.

To train a model with **tfruns**, just use the `training_run()` function in place of the `source()` function to execute your R script. For example:

``` r
library(tfruns)
training_run("mnist_mlp.R")
```

When training is completed, a summary of the run will automatically be displayed if you are within an interactive R session:

<kbd><img src="images/view_run.png" width=675/></kbd>

The metrics and output of each run are automatically captured within a *run directory* which is unique for each run that you initiate. Note that for Keras and TF Estimator models this data is captured automatically (no changes to your source code are required).

You can call the `latest_run()` function to view the results of the last run (including the path to the run directory which stores all of the run's output):

``` r
latest_run()
```

    $ run_dir           : chr "runs/2017-10-02T14-23-38Z"
    $ eval_loss         : num 0.0956
    $ eval_acc          : num 0.98
    $ metric_loss       : num 0.0624
    $ metric_acc        : num 0.984
    $ metric_val_loss   : num 0.0962
    $ metric_val_acc    : num 0.98
    $ flag_dropout1     : num 0.4
    $ flag_dropout2     : num 0.3
    $ samples           : int 48000
    $ validation_samples: int 12000
    $ batch_size        : int 128
    $ epochs            : int 20
    $ epochs_completed  : int 20
    $ metrics           : chr "(metrics data frame)"
    $ model             : chr "(model summary)"
    $ loss_function     : chr "categorical_crossentropy"
    $ optimizer         : chr "RMSprop"
    $ learning_rate     : num 0.001
    $ script            : chr "mnist_mlp.R"
    $ start             : POSIXct[1:1], format: "2017-10-02 14:23:38"
    $ end               : POSIXct[1:1], format: "2017-10-02 14:24:24"
    $ completed         : logi TRUE
    $ output            : chr "(script ouptut)"
    $ source_code       : chr "(source archive)"
    $ context           : chr "local"
    $ type              : chr "training"

The run directory used in the example above is "runs/2017-10-02T14-23-38Z". Run directories are by default generated within the "runs" subdirectory of the current working directory, and use a timestamp as the name of the run directory. You can view the report for any given run using the `view_run()` function:

``` r
view_run("runs/2017-10-02T14-23-38Z")
```

Comparing Runs
--------------

Let's make a couple of changes to our training script to see if we can improve model performance. We'll change the number of units in our first dense layer to 128, change the `learning_rate` from 0.001 to 0.003 and run 30 rather than 20 `epochs`. After making these changes to the source code we re-run the script using `training_run()` as before:

``` r
training_run("mnist_mlp.R")
```

This will also show us a report summarizing the results of the run, but what we are really interested in is a comparison between this run and the previous one. We can view a comparison via the `compare_runs()` function:

``` r
compare_runs()
```

<kbd><img src="images/compare_runs.png" width=675/></kbd>

The comparison report shows the model attributes and metrics side-by-side, as well as differences in the source code and output of the training script.

Note that `compare_runs()` will by default compare the last two runs, however you can pass any two run directories you like to be compared.

Using Flags
-----------

Tuning a model often requires exploring the impact of changes to many hyperparameters. The best way to approach this is generally not by changing the source code of the training script as we did above, but instead by defining flags for key parameters you may want to vary. In the example script you can see that we have done this for the `dropout` layers:

``` r
FLAGS <- flags(
  flag_numeric("dropout1", 0.4),
  flag_numeric("dropout2", 0.3)
)
```

These flags are then used in the definition of our model here:

``` r
model <- keras_model_sequential()
model %>%
  layer_dense(units = 128, activation = 'relu', input_shape = c(784)) %>%
  layer_dropout(rate = FLAGS$dropout1) %>%
  layer_dense(units = 128, activation = 'relu') %>%
  layer_dropout(rate = FLAGS$dropout2) %>%
  layer_dense(units = 10, activation = 'softmax')
```

Once we've defined flags, we can pass alternate flag values to `training_run()` as follows:

``` r
training_run('mnist_mlp.R', flags = c(dropout1 = 0.2, dropout2 = 0.2))
```

You aren't required to specify all of the flags (any flags excluded will simply use their default value).

Flags make it very straightforward to systematically explore the impact of changes to hyperparameters on model performance, for example:

``` r
for (dropout1 in c(0.1, 0.2, 0.3))
  training_run('mnist_mlp.R', flags = c(dropout1 = dropout1))
```

Flag values are automatically included in run data with a "flag\_" prefix (e.g. `flag_dropout1`, `flag_dropout2`).

Analyzing Runs
--------------

We've demonstrated visualizing and comparing one or two runs, however as you accumulate more runs you'll generally want to analyze and compare runs many runs. You can use the `ls_runs()` function to yield a data frame with summary information on all of the runs you've conducted within a given directory:

``` r
ls_runs()
```

    # A tibble: 6 x 27
                        run_dir eval_loss eval_acc metric_loss metric_acc metric_val_loss
                          <chr>     <dbl>    <dbl>       <dbl>      <dbl>           <dbl>
    1 runs/2017-10-02T14-56-57Z    0.1263   0.9784      0.0773     0.9807          0.1283
    2 runs/2017-10-02T14-56-04Z    0.1323   0.9783      0.0545     0.9860          0.1414
    3 runs/2017-10-02T14-55-11Z    0.1407   0.9804      0.0348     0.9914          0.1542
    4 runs/2017-10-02T14-51-44Z    0.1164   0.9801      0.0448     0.9882          0.1396
    5 runs/2017-10-02T14-37-00Z    0.1338   0.9750      0.1097     0.9732          0.1328
    6 runs/2017-10-02T14-23-38Z    0.0956   0.9796      0.0624     0.9835          0.0962
    # ... with 21 more variables: metric_val_acc <dbl>, flag_dropout1 <dbl>,
    #   flag_dropout2 <dbl>, samples <int>, validation_samples <int>, batch_size <int>,
    #   epochs <int>, epochs_completed <int>, metrics <chr>, model <chr>, loss_function <chr>,
    #   optimizer <chr>, learning_rate <dbl>, script <chr>, start <dttm>, end <dttm>,
    #   completed <lgl>, output <chr>, source_code <chr>, context <chr>, type <chr>

Since `ls_runs()` returns a data frame you can also render a sortable, filterable version of it within RStudio using the `View()` function:

``` r
View(ls_runs())
```

<kbd><img src="images/ls_runs_rstudio.png" width=675/></kbd>

The `ls_runs()` function also supports `subset` and `order` arguments. For example, the following will yield all runs with an eval accuracy better than 0.98:

``` r
ls_runs(eval_acc > 0.98, order = eval_acc)
```

You can pass the results of `ls_runs()` to compare runs (which will always compare the first two runs passed). For example, this will compare the two runs that performed best in terms of evaluation accuracy:

``` r
compare_runs(ls_runs(eval_acc > 0.98, order = eval_acc))
```

<kbd><img src="images/ls_runs_compare.png" width=675/></kbd>

Run Output
----------

Any graphical or console output as well as file artifacts created by a training run (e.g. saved models or saved model weights) can be viewed from the **Output** tab of the run view:

<kbd><img src="images/view_run_output.png" width=675/></kbd>

You can use the `copy_run_files()` function to export file artifacts from runs into another directory. For example:

``` r
copy_run_files("runs/2017-09-24T10-54-00Z", to = "saved-model")
```

You can also use the `copy_run()` function to export a run directory in it's entirety. For example, this code exports the specified run to a "best-run" directory:

``` r
copy_run("runs/2017-09-24T10-54-00Z", to = "best-run")
```

Note that `copy_run()` will accept any number of runs. For example, this code exports all run directories with an evaluation accuracy greater than 0.98 to a "best-runs" directory:

``` r
copy_run(ls_runs(eval_acc >= 0.98), to = "best-runs")
```

Managing Runs
-------------

### Cleaning Runs

You can use the `clean_runs()` function to archive a set of runs you no longer need the data from. For example, this code archives all runs with an eval accuracy less than 0.98:

``` r
clean_runs(ls_runs(eval_acc < 0.98))
```

If you don't specify a set of runs to clean then all runs will be archived:

``` r
clean_runs() # archives all runs in the "runs" directory
```

Note that you'll always get a confirmation prompt before the runs are actually archived.

### Purging Runs

When runs are archived they are moved to the "archive" subdirectory of the "runs" directory. If you want to permanently remove runs from the archive you call the `purge_runs()` function:

``` r
purge_runs()
```

### Experiment Scopes

By default all runs go into the "runs" sub-directory of the current working directory. For various types of ad-hoc experimentation this works well, but in some cases you may want to create a separate directory scope for the set of runs that compose an experiment.

To do this you can either call `clean_runs()` before beginning work on a new experiment, or you can set the `tfruns.runs_dir` global option to ensure that all run operations (including queries with `ls_runs()` use a separate scope). To return to the previous example of experimenting with various dropout values, I might have a driver script that looks like this:

``` r
library(tfruns)

# use 'dropout_experiment_runs' as the run_dir
options(tfruns.runs_dir = "dropout_experiment_runs")

# try 9 perumutations of dropout
for (dropout1 in c(0.1, 0.2, 0.3))
  for (dropout2 in c(0.1, 0.2, 0.3))
    training_run('mnist_mlp.R', flags = c(dropout1 = dropout1, dropout2 = dropout2))

# see which combination of dropout values performed best
ls_runs(order = eval_acc)[,c("eval_acc", "metric_acc", "flag_dropout1", "flag_dropout2")]
```

    # A tibble: 9 x 4
      eval_acc metric_acc flag_dropout1 flag_dropout2
         <dbl>      <dbl>         <dbl>         <dbl>
    1   0.9832     0.9936           0.2           0.1
    2   0.9827     0.9920           0.2           0.3
    3   0.9824     0.9903           0.3           0.2
    4   0.9817     0.9949           0.1           0.3
    5   0.9813     0.9932           0.2           0.2
    6   0.9811     0.9968           0.1           0.1
    7   0.9810     0.9883           0.3           0.3
    8   0.9798     0.9958           0.1           0.2
    9   0.9794     0.9904           0.3           0.1

RStudio IDE
-----------

If you use RStudio with **tfruns**, it's strongly recommended that you update to the current [Preview Release](https://www.rstudio.com/products/rstudio/download/preview/) of RStudio v1.1, as there are are a number of points of integration with the IDE that require this newer release.

### Addin

The **tfruns** package installs an RStudio IDE addin which provides quick access to frequently used functions from the Addins menu:

<kbd><img src="images/rstudio_addin.png" width=675/></kbd>

Note that you can use **Tools** -&gt; **Modify Keyboard Shortcuts** within RStudio to assign a keyboard shortcut to one or more of the addin commands.

### Background Training

RStudio v1.1 includes a Terminal pane alongside the Console pane. Since training runs can become quite lengthy, it's often useful to run them in the background in order to keep the R console free for other work. You can do this from a Terminal as follows:

<kbd><img src="images/rstudio_terminal.png" width=675/></kbd>

If you are not running within RStudio then you can of course use a system terminal window for background training.

### Publishing Reports

Training run views and comparisons are HTML documents which can be saved and shared with others. When viewing a report within RStudio v1.1 you can save a copy of the report or publish it to RPubs or RStudio Connect:

<kbd><img src="images/rstudio_publish.png" width=675/></kbd>

If you are not running within RStudio then you can use the `save_run_view()` and `save_run_comparison()` functions to create standalone HTML versions of run reports.
