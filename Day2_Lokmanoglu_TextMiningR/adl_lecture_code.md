---
title: "Topic Modeling in R Studio"
subtitle: "2nd Summer School in Computational Social Sciences"
author: "Ayse Deniz Lokmanoglu"
date: 'July 26, 2022'
output: github_document

---

```{r setup, }
knitr::opts_chunk$set(warning = FALSE, message = FALSE) 
```

## What is text analysis?
* Deriving information from a text
* When we have a lot of text, computational methods help us derive detailed information from the text faster, and with less researcher bias
* Today we will learn how to do topic modeling using R Studio
---

## What is Topic Modeling?
* Topic modeling is a bag-of-words approach
* Topic modeling, or Latent Dirichlet Allocation (LDA), is a computational content analysis tool that surfaces the "hidden thematic structure of a collection of text" (Maier et al., 2018: 93)
* Through an inductive approach to quantitative measurements, it allows researchers to conduct semantic analysis on a large number of texts. 
* LDA conducts measurements in three levels: corpus, documents, and terms. The corpus consists of a collection of documents, and each document consists of a collection of words (referred to as terms in the algorithm). 
* The LDA algorithm models the representation of the words, with each other, within a document and within the corpus, through "topics" (Maier et al., 2018: 94). 
* These facilitate researchers to label topics inductively, by using both the words within each topic, as well as the documents in each topic. 
* Thus, LDA analysis allows a document to represent multiple topics, providing a deeper insight into the thematic structure of the corpus.
---
[![image.png](https://i.postimg.cc/6591RPdk/image.png)](https://postimg.cc/SjvrbD1d)
---
## Let's start coding!
First we install all the packages! You only have to do this once, for all other times you just need to load the packages (see next slide)

```{r needed-packages, eval=FALSE}
# this code is to install all the packages, you only need to run this once, afterwards all you need is to load the packages 
install.packages(c(
  "tidyverse",   # foundation packages needed for text analysis
  "tidytext",    # foundation packages needed for text analysis
  "dplyr",       # foundation packages needed for text analysis
  "tm",          # text mining package   
  "quanteda",    # Quantitative Analysis of Textual Data
  "ldatuning",   # Tuning of the Latent Dirichlet Allocation Models Parameters
  "topicmodels", # Topic Model package
  "scales",      # scale functions for visualizations
  "ggthemes",    # graph theme options
  "jtools",      # ggplot2 themes
  "ggplot2",     # visualization
  "lubridate",   # for dates
  "zoo"          # another package for dates
  ))
```


---
### Load packages

```{r load-packages, warning=FALSE}
library(tidyverse)
library(tidytext)
library(dplyr)
library(tm)
library(quanteda)
library(ldatuning)
library(topicmodels)
library(scales)
library(ggthemes)
library(lubridate)
library(jtools)
```

How to find details on packages?
Type the package name preceded by ? and you will see the package details on the help window

```{r package-help, eval=FALSE, warning=FALSE}
?tidyverse
```

---

### Our dataset

```{r load-dataset}
url<-c("https://dataverse.harvard.edu/api/access/datafile/6389385")
mydata <- read_csv(url)
glimpse(mydata)
```

---
As you can see we have an index column from csv labelled *...1*, and date column. So first lets remove the extra column, make sure date is coded as date.
* Why did we write the command `select()` with its package name?

```{r clean-dataset}
mydata <- mydata %>%
  dplyr::select(-...1) %>% #from dplyr we drop the column with -, and this case 
  mutate(date=ymd(date)) %>% #using lubridate we change date column to date variable
  mutate(text=originaltext) #Create a new column labelled text - to keep original text safe
glimpse(mydata) #lets see what our dataset is made of
```

---
### Preprocessing, getting ready for LDA
* Tokenize it

```{r, warning=FALSE}
toks <- tokens(mydata$text,
               remove_punct = TRUE,
               remove_symbols = TRUE,
               remove_numbers = TRUE,
               remove_url = TRUE,
               remove_separators = TRUE,
               split_hyphens = FALSE,
               include_docvars = TRUE,
               padding = FALSE) %>%
  tokens_remove(stopwords(language = "en")) %>% #for this we used combined stopwords list from google with quanteda 
  tokens_select(min_nchar = 2)
head(toks)
```

---
### Next Steps
* Change it into a [document-feature matrix](https://quanteda.io/reference/dfm.html)
* Match your dfm object with your original data frame through index

```{r, warning=FALSE}
dfm_counts<- dfm(toks) 
rm(toks) 
docnames(dfm_counts)<-mydata$index#remove unused files to save space
```

---
### LDA Object
* Convert dfm object to an LDA object

```{r, warning=FALSE}
dtm_lda <- convert(dfm_counts, to = "topicmodels",docvars = dfm_counts@docvars) #convert the data set to a document term matrix
n <- nrow(dtm_lda) # number of rows for cross-validation method
rm(dfm_counts) # remove for space
dtm_lda
```

---



# Let's run our topic model!

---
### Find K
* This function is from [ldatuning](https://cran.r-project.org/web/packages/ldatuning/vignettes/topics.html) package
**I ran the code already to save time** *You can run it on your own time by erasing the markdown option 'eval=FALSE'* 

```{r, eval=FALSE}
Sys.time()
result <- FindTopicsNumber(
  dtm_lda,
  topics = seq(2,50,by=10), # Specify how many topics you want to try.
  metrics = c("Griffiths2004", "CaoJuan2009", "Arun2010", "Deveaud2014"),
  method = "Gibbs",
  control = list(seed = 9), # random seed number
  mc.cores = 2L,
  verbose = TRUE
)
Sys.time()
save(result, file="Class_FindK.Rda")
FindTopicsNumber_plot(result)
ggsave("Class_Find_K.jpg", width=8.5, height=5, dpi=150)
```


---
### Plot Result
[![Class-Find-K.jpg](https://i.postimg.cc/dtPr1JNq/Class-Find-K.jpg)](https://postimg.cc/SjdJ1bp5)

---

### Let's run our topic model!
We identified our optimal k as 22 from the graph, but for our ease of analysis we will model on 5 topics

```{r, warning=FALSE}
Sys.time()
covid_lda <- LDA(dtm_lda, k = 5, control = list(seed = 1234))
save(covid_lda, file="Class_lda_K5.Rda") #always save your variables
Sys.time()
covid_lda
```

---

### Extract data from the lda model
* We can extract top words and documents

```{r, warning=FALSE}
covid_topics <- tidy(covid_lda, matrix = "beta")
head(covid_topics)
```


---

### Visualize top words

```{r topis-topterms, warning=FALSE}
covid_top_terms <- covid_topics %>%
  group_by(topic) %>%
  slice_max(beta, n = 10) %>% 
  ungroup() %>%
  arrange(topic, -beta)
```
```{r topterms-plot, echo=FALSE}
covid_top_terms %>%
  mutate(term = reorder_within(term, beta, topic)) %>%
  ggplot(aes(beta, term, fill = factor(topic))) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~ topic, scales = "free") +
  scale_y_reordered()
```


---

### We can label topics using top words
* In our case it looks like
  + Topic 1: Variants
  + Topic 2: Vaccine Mandates
  + Topic 3: Pandemic Regulations
  + Topic 4: Relief Stimulus
  + Topic 5: Covid19 and Government
* We should create a variable called topic_names and save it for future

```{r topic-names, warning=FALSE}
topic_names<-c("Variants", "Vaccine_Mandates", "Pandemic_Regulations", "Relief_Stimulus", "Covid19_and_Government")
```

***Why did I use underscore when creating the `topic_names` variable?*** 

---

### Document-topic probabilities
* We will extract the γ (“gamma”) value which is per-document-per-topic probabilities.This value estimates the proportion of words from each document that belong to that topic.  

```{r topic-gamma, warning=FALSE}
covid_documents <- tidy(covid_lda, matrix = "gamma")
glimpse(covid_documents)
```


---

### Join with original document?
* We saw in our gamma values we have a document number **equal to our index** (from previous slides). We can join with our document and see for example which topics come from what domains?
* But first we can see that the dimensions of `covid_documents` and `mydata` are different, why?

```{r doc-dimension, warning=FALSE}
dim(covid_documents)
dim(mydata)
```


---

### Wide documents
* What we have is a long document, What we need to do is change the document to wider, having each document with topics as columns
* We can reshape the data frame using`dpylr`'s `pivot_wider()` and `pivot_longer()`

```{r pivot-wider-help, eval=FALSE}
?pivot_wider()
```


```{r pivot-wider, warning=FALSE}
covid_documents_wide<- covid_documents %>%
  pivot_wider(names_from = topic,
              values_from = gamma)
dim(covid_documents_wide)
dim(mydata)
```


---
### Column Names - Good Practice
* It is good practice to not have numbers as column names, so let's add a prefix of X

```{r change-colnames, warning=FALSE}
colnames(covid_documents_wide)[2:6] <- paste("X", colnames(covid_documents_wide[,c(2:6)]), sep = "_")
colnames(covid_documents_wide)

```

* We can also add our topic_names the same way

```{r change-colnames-labels, warning=FALSE}
covid_documents_wide_test<-covid_documents_wide # to save a backup copy
colnames(covid_documents_wide_test)[2:6] <- topic_names
colnames(covid_documents_wide_test)
```


---

### Now let's join!

```{r wrong-meta-theta-df, error = TRUE}
meta_theta_df<-left_join(mydata, covid_documents_wide, by=c("index" = "document"))
```

We need to change document in covid_documents_wide to number

```{r, warning=FALSE}
covid_documents_wide <- covid_documents_wide %>%
  mutate(document=as.numeric(document))
typeof(covid_documents_wide$document) # this is a way to check the type 
```

Let's try again!

```{r meta-theta-df}
meta_theta_df<-left_join(mydata, covid_documents_wide, by=c("index" = "document"))
meta_theta_df
```


---

### Let's look at the domains

```{r topics-domains, warning=FALSE}
domains <- meta_theta_df %>%
  dplyr::select(source.domain, X_1:X_5) %>% # selected domains and topic gammas for each document
  group_by(source.domain) %>% # grouped by domains 
  summarise(across(everything(), sum)) # summed all the topic gammas
dim(domains)
```

***Now you can see we have a new data-set with 472 domains, and the topic probabilities for each domain***

---

### Which domain has the highest topic probabilities?
* let's do topic 4 and the top 10 domains

```{r slicemax, warning=FALSE}
topic4 <- domains %>%
  slice_max(X_4, n=10)
head(topic4)
```

* play with different topics and `slice_max()` & `slice_min()`from `dplyr` [package](https://dplyr.tidyverse.org/reference/slice.html)

---

### Visualize comparison of domains
* Let's compare cnn.com and nbcnews.com
  + We want to make a graph bar graph that has both domains and topic probabilities. 
  + First let's create a smaller dataframe with the two domains


```{r domain-comp, warning=FALSE}
domain_comp<- domains %>%
  filter(source.domain=="cnn.com" | source.domain=="nbcnews.com")
domain_comp
```


---

### Set the data for bar graph
* We need to make domains a group, topics as x axis and gamma values as y.
* So we need to make the document long, by `dplyr` packages `pivot_longer()`

```{r domain-long, warning=FALSE}
domain_long <- domain_comp %>%
  pivot_longer(!source.domain,
               names_to = "topics", # names as topic
               values_to = "gamma")
head(domain_long)
```


---

### Now lets graph it

```{r domain-graph, echo=FALSE}
ggplot(domain_long,
       aes(fill=source.domain, y=gamma, x=topics)) + 
    geom_bar(position="dodge", stat="identity")
```


---

### Play with graphs
* We can also add our topic labels, play styles using `ggthemes()` package

```{r domain-graph-fancy, echo=FALSE}
ggplot(domain_long,
       aes(fill=source.domain, y=gamma, x=topics)) + 
    geom_bar(position="dodge", stat="identity") +
  scale_x_discrete(labels=c("X_1" = "Variants", "X_2" = "Vaccine Mandates", "X_3" = "Pandemic Regulations", "X_4" =  "Relief Stimulus", "X_5" = "Covid19 & Government")) +     theme_wsj(color="white") +
  labs(x = " ",
       y = " ",
       title = "Topic Probability Domain Comparison") +
  theme(text=element_text(size=8,family="Sans"),
        title=element_text(size=8,family="Sans"),
        axis.text.x=element_text(size=8, angle=60, hjust=1, family="Sans"),
        axis.text.y=element_text(size=8, family="Sans"),
        axis.title.x=element_text(vjust=-0.25, size=8, family="Sans"),
        axis.title.y=element_text(vjust=-0.25, size=8, family="Sans"),
        legend.position = "right")
```


---

### Lastly let's do topics over time
* For this we well again use our meta_theta_df document, this time we will summarize by dates

```{r topics-time, warning=FALSE}
topics_time <- meta_theta_df %>%
  dplyr::select(date, X_1:X_5) %>% # selected dates and topic gammas for each document
  group_by(date) %>% # grouped by dates 
  summarise(across(everything(), mean)) # summed all the topic gammas
```

* We have to make this document long to plot it, using `pivot_longer()`

```{r pivot-long, warning=FALSE}
topic_time_long <- topics_time %>%
  pivot_longer(!date, # long from date
               names_to = "topics", # names as topic
               values_to = "gamma") # values as gamma
topic_time_long
```


---

### Visualize it

```{r topictime-plot, echo=FALSE}
ggplot(topic_time_long, aes(
  x = date,
  y = gamma,
  group = topics,
  color = topics)) +
  geom_line() +
  theme_nice() +
  scale_x_date(date_breaks="1 month", date_labels = "%b-%Y")+
  labs(x = " ",
       y = "Topic Probabilities (mean)",
       title = " ") +
  theme(text=element_text(size=10,family="Sans"),
        title=element_text(size=10,family="Sans"),
        axis.text.x=element_text(size=8, angle=60, hjust=1, family="Sans"),
        axis.text.y=element_text(size=8, family="Sans"),
        axis.title.x=element_text(vjust=-0.25, size=8, family="Sans"),
        axis.title.y=element_text(vjust=-0.25, size=8, family="Sans"),
        legend.position = "bottom")
```

* This is very crowded 

---

### Simpler graph
* let's pick topics 2 and 4

```{r topictime-plot-2, echo=FALSE}
topic_time_long %>%
  filter(topics %in% c("X_2", "X_4")) %>%
  ggplot(aes(
  x = date,
  y = gamma,
  group = topics,
  color = topics)) +
  geom_line() +
    theme_wsj(color="white") +
  scale_x_date(date_breaks="1 month", date_labels = "%b-%Y")+
  labs(x = " ",
       y = "Topic Probabilities (mean)",
       title = " ") +
  theme(text=element_text(size=10,family="Sans"),
        title=element_text(size=10,family="Sans"),
        axis.text.x=element_text(size=8, angle=60, hjust=1, family="Sans"),
        axis.text.y=element_text(size=8, family="Sans"),
        axis.title.x=element_text(vjust=-0.25, size=8, family="Sans"),
        axis.title.y=element_text(vjust=-0.25, size=8, family="Sans"),
        legend.position = "bottom")
```


---

### Even simpler graph
* Make it monthly

```{r date-month, warning=FALSE}
meta_theta_df$yearmonth<-my(zoo::as.yearmon(meta_theta_df$date))
topics_year_mon <- meta_theta_df %>%
  dplyr::select(yearmonth, X_1:X_5) %>% # selected dates and topic gammas for each document
  group_by(yearmonth) %>% # grouped by dates 
  summarise(across(everything(), mean))
# pivot longer like before
topics_year_mon_long <- topics_year_mon %>%
  pivot_longer(!yearmonth, # long from date
               names_to = "topics", # names as topic
               values_to = "gamma") # values as gamma
head(topics_year_mon_long)
```


---
### Lets graph it again
* Mean monthly topic probabilities

```{r topicmonth-plot, echo=FALSE}
ggplot(topics_year_mon_long, aes(
  x = yearmonth,
  y = gamma,
  group = topics,
  color = topics)) +
  geom_line() +
  theme_nice() +
  scale_x_date(date_breaks="1 month", date_labels = "%b-%Y")+
  labs(x = " ",
       y = "Topic Probabilities (mean)",
       title = " ") +
  theme(text=element_text(size=10,family="Sans"),
        title=element_text(size=10,family="Sans"),
        axis.text.x=element_text(size=8, angle=60, hjust=1, family="Sans"),
        axis.text.y=element_text(size=8, family="Sans"),
        axis.title.x=element_text(vjust=-0.25, size=8, family="Sans"),
        axis.title.y=element_text(vjust=-0.25, size=8, family="Sans"),
        legend.position = "bottom")
```


---



# Questions?

---



# Thank you!
My email is ayse.lokmanoglu@northwestern.edu and my github page where I have more challenging topic model codes to play with!

