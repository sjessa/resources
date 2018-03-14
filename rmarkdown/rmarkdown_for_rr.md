---
title: 'Using R Notebooks for reproducible, documented research'
author: Selin Jessa
output:
    html_document:
        toc: yes
        toc_float: yes
        toc_depth: 4
---

*Not a getting-started guide, but some tips I've collected for incorporating
R Notebooks/R Markdown into a research workflow*

### Markdown header:

```
---
title: '`chromswitch`: Analysis'
author: "Selin Jessa"
date: "`r format(Sys.time(), '%d %B, %Y')`"
output:
  html_document:
    df_print: paged
    number_sections: yes
    toc: yes
    toc_float: yes
    code_folding: show
---
```

* `df_print` displays data frames nicely in the output, so that only 10 rows are displayed at a time and you can click through the data
* `toc_float` makes a table of contents that's locked to the left side of the screen as you scroll (only possible for HTML output) like in this doc
* Add the date in this format to grab the current date every time the document is rendered
* `code_folding` puts a nice button in the top-right corner of the HTML document that lets you toggle between showing/hiding all code at once


### Document-wide chunk options
This is usually the first chunk in my document
```` 
```{r setup, echo = FALSE, message = FALSE, warning = FALSE}
knitr::opts_chunk$set(
                      # Display PNG but also keep PDF (reverse the order for PDF outputs)
                      dev = c("png", "pdf"), 
                      # In general, keep all the figures produced in a chunk
                      fig.keep = "all",      
                      # Save all figures
                      fig.path = "figures/", 
                      # Cache chunk results and use cache if the chunk hasn't
                      # been changed since last rendering
                      cache = TRUE,          
                      cache.path = "cache/")
```
````

### Organizing an analysis with multiple documents
A structure that's worked for me:

```
project/
  analysis/
    run.sh
    01_compute.Rmd
    02_analyze.Rmd
    03_viz.Rmd
    figures/01
            02
            03
    cache/01
          02
          03
```
With multiple documents, cache and figure outputs can get messy, so I add a subdirectory with the prefix of the analysis to keep things organized.
For example, in `01_compute.Rmd`, the first chunk would have `fig.path = "figures/01/"` and `cache.path = "cache/01/"`.

### Strategies for handling long/expensive computations

#### 1. Cache chunk output

* *Any* change to the chunk will trigger the chunk being re-run when the document is next rendered, even a new space or new line
* A change to an object in a prior chunk won't trigger re-running of later chunks which use that object, which often makes debugging tricky. A common issue I run into is that data changes but when the ggplot is done in a different chunk where the code doesn't change, the plot is not recreated to reflect the new data
* Check here for specifying chunk dependencies on previous chunks: https://yihui.name/knitr/options/
* Before finalizing an analysis it's a good idea to turn off the `cache` option and render the document once more, just to avoid unexpected surprises

#### 2. Save/Load RData objects

This is the paradigm I usually use: 

````
```{r long_computation, eval = TRUE}
# Run the long computation in this chunk

df <- runLongComputation()
save(df, file = "computations/df.RData")
```

```{r load_results}
# Load the results and do the downstream analys, viz afterwards

load("computations/df.RData")
df %>% ggplot(aes(x = X, y = Y)) + ...
```
````
* **Don't forget to change `eval = TRUE` to `FALSE`** in the first chunk with the long computation before rendering the document again! That will have the consequence that the code chunk is still shown (since by default `echo = TRUE`) but the code inside is *not* run.
* Make sure that you name the `file` argument to `save()` as it's done here... otherwise, after running the expensive computation the chunk will throw an error :'(

#### 3. Rendering the document on the commandline/a cluster

You can render an R Markdown document/ R Notebook from within RStudio, but it can also be done from the R console and consequently the bash commandline, so it's easy to run the job on a cluster.

Here's an example of a script to `qsub` a job to a Torque queue:

```
#!/usr/bin/bash
#PBS -o logs/
#PBS -e logs/
#PBS -l walltime=01:00:00
#PBS -l nodes=1:ppn=1
#PBS -l mem=50G
#PBS -l vmem=50G

cd /mnt/KLEINMAN_SCRATCH/projects/sjessa/analysis
mkdir -p logs/

module load R/3.3.2
R --no-save -e "rmarkdown::render('analysis.Rmd', 'html_document')"
```

Another option, if you have multiple Rmd documents that you will want to render,
is to have one script which takes the document name as a variable on the commandline.
The last line of the above script would then be:

```
R --no-save -e "rmarkdown::render('${rmd}', 'html_document')"
```

and it would be run on the commandline like:

```
$ qsub -v rmd=analysis.Rmd run.sh
```

### Plotting tips and tricks

* You can resize the plot output from within the chunk: `{r plot_chunk, fig.width = 5, fig.height = 5}`
* If you're plotting with base R for example, in layers, and only want to keep the final figure produced in the chunk: `{r layered_plot, fig.keep = "last", fig.show = "last"}`
* Mentioning it explicitly here: for HTML documents, it's important that the first option to `dev` in the first chunk be `png`, and for PDFs, that it be `pdf`. If figures aren't showing up in HTML, or are poor quality in PDF, this may be one explanation.
* I find it very helpful to save tidy data as it is just before it's passed to ggplot as a `tsv` file, which makes it easy to tweak figures for presentations and papers, etc., later without having to mess with the analysis. The tee function from `magrittr`, `%T>%`, pipes the RHS to the LHS but *returns the RHS to the next function* and makes this simple to do in a pipeline:

```
library(magrittr)
library(tidyverse)

df %>%
  gather(key = method, value = performance, Hierarchical, Kmeans) %T>% # Magic happens here
  readr::write_tsv("fig_data/figure1_data.tsv") %>% # Write a TSV with sane defaults
  ggplot(aes(x = method, y = performance)) +        # ggplot receives the output of gather!    
  geom_bar() +
  ...

```

### Miscellaneous: avoiding heartbreak with safer programming strategies

#### 1. Error-handling in independent computations

````
```{r safe_programming}

# Make a safe version of the function
safeCallWholeRegion <- purrr::safely(chromswitch::callWholeRegion)

# Run the analysis using the safe version
out_raw <- mclapply(benchmark, safeCallWholeRegion,
                            peaks = pk_k4me3,
                            metadata = meta,
                            ... # other arguments,
                            mc.cores = n_cores) %>% 
            # Transform the list of result-error pairs into two lists,
            # one of results, one of errors
            purrr::transpose()

# Keep the successful results by subsetting to entries where the error is NULL
out_ok <- out_raw$result[purrr::map_lgl(out_raw$error, is_null)] %>%
    # If the output can be coerced to a dataframe
    bind_rows()

# Save the raw output for inspection of errors later, and the succeeded results
save(out_ok, out_raw, file = "computations/safe_out.RData")
```
````

#### 2. Namespacing commonly used functions
e.g. `select`: This function *always* seems to be masked by a function in another package which inevitably 
causes errors, so first, load it as the *last* library, and second, namespace
the function in the analysis by calling `dplyr::select()` instead of just `select()`, 
to avoid ambiguity. This sometimes is an issue for `dplyr::mutate()` as well,
and some of the functions in `GenomicRanges`.


### Session Info
As a lightweight way of tracking dependencies, put a call to `sessionInfo()` in the very last chunk:
````
```{r sinfo}
sessionInfo()
```
````
This will automatically print an updated list of the packages loaded directly and indirectly in the document, and more importantly, their versions.

### Further reading
* http://t-redactyl.io/blog/2016/10/a-crash-course-in-reproducible-research-in-r.html
* http://blog.jom.link/implementation_basic_reproductible_workflow.html
* https://github.com/leekgroup/polyester_code
* https://rachaellappan.github.io/rmarkdown/
* http://timoast.github.io/2017/04/03/comp-notebook/

