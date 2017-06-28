# Job Scheduling on RStudio Connect

This demo shows how to set up scheduled data updates (generated from python code) which are picked up by downstream apps using RStudio Connect.

## The Data Pull

The `python_etl_twitter.Rmd` file pulls in the latest batch of twitter data for the #rstats tag and performs some text cleansing. The notebook is deployed to RStudio Connect and scheduled as "report".

### Compared to CRON
An alternative to using Connect would be to set up a crontab that runs the python script. The benefit of using Connect over this approach are:

1. The updates can be scheduled with a simple web UI

2. Connect will email you if a task fails and can optionally email you the html file (generated from the ETL Rmd file) as an artefact of success

3. Placing the ETL code in an Rmd file allows the code and documentation for the pipeline to live side-by-side


## The App

The python_etl_twitter_app.Rmd file creates a Shiny application that is deployed and running on Connect. The app visualizes the resulting text data.

The application includes a download handler. When clicked, this download handler allows users to download a zipped file that includes the most current data set and an Rmd template (stored in /Resources/results.Rmd). The idea is that a user can download the results and an Rmd file, and then knit the Rmd file to have a pdf report. This step could be simplified even further by having the download handler knit the Rmd on the fly, and then shipping a zip file with a resulting dataset and pdf to the client. 

## The Glue Holding Things Together

The Rmd file is using python to generate an aggregated, cleansed view of the data. This view is saved as a [feather file](https://github.com/wesm/feather). Feather is a optimized file format designed to be an intermediary between python and R. The downstream app uses [reactivePoll](http://shiny.rstudio.com/reference/shiny/latest/reactivePoll.html) to listen for changes to this file. 

Additionally, the ETL Rmd updates a log file. This is done in an R code chunk with a system command. The log file is read in the app using a second `reactiveFileReader`. 

## Technical Pre-Reqs

The following python libraries must be installed on Connect:

``` bash
pip install tweepy
pip install feather
pip install nltk
pip install pandas
```
You may also need to update the python feather package and the feather format:

```bash
pip install feather --upgrade
pip install feather-format --upgrade
```

Be sure to have the latest version of the R feather package as well:

```R
install.packages("feather")
```

The `nltk` library requires a one-time download of the stopwords dataset:

```python
import nltk
nltk.download('stopwords')
```

The resulting feather file and log file need to be accessible by both the App and the Rmd file performing the data updates. To do so, a directory was created on Connect with read/write priveleges for the `rstudio-connect` user. This directory was hard coded into both Rmd files prior to deploying the content.

## A note on API Authentication

The twitter API requires credentials. These credentials were stored in a separate file that was read in the python ETL script. This file lives side-by-side with the python_etl_twitter.Rmd file and should be included in the deployment of that file. The credential file will end up living on the Connect server along side the python_etl_twitter.Rmd file and will be readable by the rstudio-connect user.

I used feather to store these credentials. This has the benefit that a feather file is not human readable. However, any file format can be used or you could hardcode the credentials into the python code chunk.

To create this file:

1. Set up a twitter app: https://apps.twitter.com/

2. Create a feather file that contains the credentials in a dataframe:

```r
library(feather)
cred <- data.frame(
      consumer_key = CONSUMER_KEY,
      consumer_secret = CONSUMER_SECRET,
      access_token = ACCESS_TOKEN,
      access_token_key = ACCESS_TOKEN_SECRET
)
write_feather(cred, "cred.feather")  
```

Be sure to publish this file alongside of the python_etl_twitter.Rmd

## Thanks .... 


Some inspiration for the twitter preprocessing code:
https://marcobonzanini.com/2015/03/09/mining-twitter-data-with-python-part-2/



