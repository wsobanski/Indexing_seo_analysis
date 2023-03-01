# **SEO analysis - problems with page indexing

This html file contains analysis of indexing dataset. Document is divided into five parts that are listed in left corner. This file has ability to show R code by clicking on the **CODE** tab.


# **Dataset information**

Structure of dataset is prestened in table below:
```{r}
data=data.frame(read.csv2("indexing (2).csv", sep=",", na.strings = "", encoding = "UTF-8"))
data$date=as.Date(data$date, format = "%Y-%m-%d")
data$status = as.factor(data$status)
data$status_details=as.factor(data$status_details)
summary(data)
```
```{r, echo = FALSE, results='hide', message=FALSE, warning=FALSE}
data$date=as.character(data$date, format = "%Y-%m-%d")
data$status = as.character(data$status)
data$status_details=as.character(data$status_details)
```


Dataset contains time series from 6.12.2021 to 16.12.2021 with information about
indexing status of 100 domains in Google search engine. There are 6863 observations.
Most of the observations have status *Excluded*.

This time series have no information for *date* column in 67 cases, for days such
as 8th and 12th December – those observations will be omitted for further analysis.
Original dataset contains null values for 69 observations in column *status* –
those missing values can be imputed based on *status_details* that are avaliable for those observations.
Observations without values in *status_details* will be ignored.
There are also 64 rows without information about number of pages - these cases
will be omitted as well.

# **Dealing with null values**

As written before, all null values will be omitted, except for nulls in column 
*status*. For imputing these rows data frame containg all statuses and status details will be created.

```{r}
df = data.frame(unique(data[,c('status', 'status_details')] ))|>
na.omit()
df = arrange(df, desc(status), status_details)
df
```


Now values can be filled automaticly by loop comparing *status_details* columns in both tables. 

```{r}
for (i in 1:nrow(data)) {
  if (is.na(data$status[i])==TRUE){
    for (j in 1:nrow(df)){
      if (df$status_details[j]==data$status_details[i]){
        data$status[i]=df$status[j]
      }
    }
  }
}

```

After imputing not avaliable data for *status* column, other nulls can be omitted.
Summary of clean dataset is presented in the table below.

```{r}
data = na.omit(data)
data$date=as.Date(data$date, format = "%Y-%m-%d")
data$status = as.factor(data$status)
data$status_details=as.factor(data$status_details)
summary(data)
```





# **Data transformation**

Data transoframtion is needed to achieve cleaner and less cluttered visualisations in further steps.

Due to the fact that spread between minimum and maximum value in *pages* column is very large, there will be aplied cutoff for top and bottom 5% of observations by number of pages.

Summary of data after triming is presented below:
```{r}
data = subset(data,pages>0 & pages >= quantile(data$pages, probs = 0.05) & pages <= quantile(data$pages, probs = 0.95))
summary(data)
```

After this transformation there are 5622 observations and values in *pages* column have lower spread.
```{r, echo = FALSE, results='hide', message=FALSE, warning=FALSE}
data$date=as.character(data$date, format = "%Y-%m-%d")
data$status = as.character(data$status)
data$status_details=as.character(data$status_details)
```

Another applied transformation is grouping some categories in *status_details* column.

```{r}
data$status_details_grouped=""
for (i in 1:nrow(data)){
  if (str_detect(data$status_details[i], "Duplicate") == TRUE){
    data$status_details_grouped[i]="Excluded due to duplication"
  }
    else if (data$status_details[i] == "Not found (404)" || data$status_details[i] == "Soft 404"){
      data$status_details_grouped[i]="Excluded due to 404 error"
    }
      else{
        data$status_details_grouped[i]=data$status_details[i]  
      }
}
```
After that transformation dataset includes new variable *status_details_grouped* which contains categories:

* **Excluded due to duplication** - contains all observations with status_details containing duplication problem,
* **Excluded due to 404 error** - contains all observations with status_details: *Not found(404)* and *Soft 404*,
* Other categories remains the same as in original dataset.


```{r, echo = FALSE, results='hide', message=FALSE, warning=FALSE}
data$date=as.Date(data$date, format = "%Y-%m-%d")
data$status = as.factor(data$status)
data$status_details=as.factor(data$status_details)
data$status_details_grouped=as.factor(data$status_details_grouped)
```



# **Data visualization**

## **Treemap of status details**

For needs of creating a graph, there will be generated another table that shows all *status_details* with *number of pages* and their percentage contribution in total number of pages.
```{r}
df1 = data.frame(unique(data[,c('status', 'status_details_grouped')] ))|>
  na.omit()
df1 = arrange(df1, desc(status), status_details_grouped)

#computing sum of pages by status_details 
for (i in 1:nrow(df1)){
  df1$value[i] = sum(data[which(data$status_details_grouped==df1$status_details_grouped[i]),5])
}

# computing percentage contribution of pages with given status_details in total number of pages
for (i in 1:nrow(df1)){
df1$percent[i] =round(df1$value[i]/sum(df1$value)*100,2)
}

```

```{r}
df1
```


After that, treemap can be properly created and displayed.

```{r}
ggplot(df1, aes(area = value, fill = status,  label = paste(status_details_grouped, '\n', percent,'%'), subgroup = status)) +
  geom_treemap()+
  geom_treemap_subgroup_border(colour="black")+
  geom_treemap_text(colour = 'black', place = 'topleft', size = 10, reflow = TRUE, grow = FALSE, min.size = 7)+
  scale_fill_manual(values=c("#B03A2E", "#888384", "#9CCC65", "#FFF176"))+
  guides(alpha = 'none')+
  labs(title = "Treemap of statuses details by number of pages",
       subtitle = "(% of total number of pages)",
       fill = "Status: " )+
  theme_fivethirtyeight()+
  theme(plot.title =element_text(size = 18, face = "bold"),
        plot.subtitle = element_text(size = 14),
        legend.text = element_text(size = 10),
        legend.title = element_text(face = "bold"))

```

![treemap](https://user-images.githubusercontent.com/94312553/222275897-5ed74936-cd21-4a04-b7be-3d1ba09dd83e.png)

As is visible from the graph, most of domains in given dataset have status *Excluded* which means that domain is not indexed. 68.8% of all pages are connected to this status. Three most common reasons for domain being excluded were:

* *Page with redirect* - 14.09% of all pages had this status,
* *Crawled - currently not indexed* - 12.5% of all pages,
* *Alternate page with proper canonical tag* - 11.15%.

Only 19.93% of pages are *Valid* domains. 8.49% percent of pages are tagged with warning. That means that only 28.42% of pages are associated with indexed pages. 

## **Trend chart**

To create chart showing trends in change of popularity of statuses, there is needed table containing information about aggregated number of pages by each day for every status.

```{r}
df3 = data.frame(date = data$date,status_det = data$status_details_grouped, pages = data$pages)
df3 = aggregate(data$pages, by = list(status_details = data$status_details_grouped, date = data$date), FUN = sum)
df3 = merge(x = df3, y = df1, by.x = 'status_details', by.y='status_details_grouped', all.x = TRUE)
df3 = df3[,c('status', 'status_details', 'date', 'x')]
colnames(df3)[4] = 'number_of_pages'

```


Now, when table is ready, trend chart can be created.

```{r}
aggregate(df3['number_of_pages'], by = list(status = df3$status, date = df3$date), FUN = 'sum')|>
ggplot(aes(x = as.Date(date), y = number_of_pages, col = status))+
  geom_line(size = 1.5, alpha = 0.75)+
  theme_fivethirtyeight()+
  scale_color_manual(values=c("#B03A2E", "#888384", "#9CCC65", "#FFF176"))+
  scale_y_continuous(labels = comma_format(big.mark = ".", decimal.mark = ","))+
  scale_x_date(limits = as.Date(c('2021-12-06','2021-12-16')))+
  labs(title = 'Popularity of statuses in given period of time',
       subtitle = "by number of pages",
       col ='Status: ',
       x = 'Date',
       y = 'Number of pages')+
  theme(axis.title = element_text(face = 'bold'),
        plot.title = element_text(size = 18, face = 'bold'),
        plot.subtitle = element_text(size = 14),
        legend.text = element_text(size = 10),
        legend.title = element_text(face = "bold"))



```

![popularitypng](https://user-images.githubusercontent.com/94312553/222275891-666d6e71-fd35-4ee3-ab40-15bd3089fe99.png)

Graph shows that at 11 December there was big fall in number of indexed pages. Moreover, there is some kind of 5/6-day cycle 
visible with peak in the middle of it.    


## **Issues with excluded domains**

Another graph will present change over time of most common issues with *Excluded* domains by number of pages connected to these domains. To create this graph, data frame created before have to be transformed by adding column containing computed ranks for each day for all status details for excluded domains.




Created bump chart looks like this:
```{r}
df3|>
  dplyr::filter(status == "Excluded")|>
  dplyr::arrange(desc(date), desc(number_of_pages))|>
  dplyr::group_by(date)|>
  mutate(rank = rank(-number_of_pages))|>
ggplot(aes(x = date, y = rank, color = status_details))+
    geom_line(alpha = 0.85, size = 1.5)+
    geom_point(size = 4, show.legend = FALSE)+
    
    scale_y_reverse(breaks = 1:nrow(df3), sec.axis = dup_axis())+
    scale_x_date()+
    scale_color_manual(values = c('#64B6AC', '#4C5B61', '#31081F', '#D7907B', '#2C423F', '#FF7F11', '#FF1B1C', '#B80C09', '#BCED09', '#FFC759', '#702632'))+
    labs(title = 'Popularity of issues with excluded domains',
         subtitle = 'by number of pages - change over time',
         color ='Issue with excluded domains:')+
    theme_fivethirtyeight()+
  guides(color = guide_legend(override.aes = list(size = 7), title.position = 'top', ncol = 3, byrow =  F ), alpha = 'none', size = 'none')+
    theme(axis.title = element_text(face = 'bold'),
          plot.title = element_text(size = 18, face = 'bold'),
          plot.subtitle = element_text(size = 14),
          legend.text = element_text(size = 7),
          legend.title = element_text(face = "bold", size = 12))
```

![rankpng](https://user-images.githubusercontent.com/94312553/222275895-37431877-5ae3-42de-bbb8-3e4f5d2f9ae1.png)

This graph shows all reasons for domain being excluded and which reasons were most common in every day from analyzed time 
peroid. As it is visible from the graph, in four days the most common issue with domains that led to exclusion from indexing was tagged as *Page with redirect*. Another rather common problem was *Alternate page with proper canonical tag*. 

## **Average number of pages by status**

Boxplot is helpful to identify average size (by numer of pages) of domains with certain status. 


```{r}
df1|> group_by(status)|>
  ggplot(aes(x=status, y=value, fill = status))+
  geom_boxplot(outlier.size = 2, outlier.colour = 'red')+
  scale_y_continuous( labels = comma_format(big.mark = ".", decimal.mark = ","))+
  scale_fill_manual(values=c("#B03A2E", "#888384", "#9CCC65", "#FFF176"))+
  labs(title = 'Boxplot of number of pages by status',
       fill ='Status: ')+
  theme_fivethirtyeight()+
  theme(axis.title = element_text(face = 'bold'),
        plot.title = element_text(size = 18, face = 'bold'),
        plot.subtitle = element_text(size = 14),
        legend.text = element_text(size = 10),
        legend.title = element_text(face = "bold"))
```

![boxpng](https://user-images.githubusercontent.com/94312553/222275884-e5a39c2d-cd83-4068-9a69-c65481a36868.png)

As it is visible from the graph, Excluded domains have the biggest spread in number of pages. Valid domains are more 
concentrated and these domains are rather big as far as number of pages is concerned. Domains with *error* status have small number of pages.



## **Pareto chart for error types**

Pareto chart shows number of pages by error type and cumulative sum for pages ordered from most common error type to the least common. Line in Pareto chart indicates contribution of cumulative number of pages in total number of pages with error.
Table below presents errors with the most common types at the top and cumulative sum of pages with its contribution in total number of pages with error.

```{r}
z = df3 |>
  filter(status=="Error")
z = aggregate(z['number_of_pages'], by =list(status_details = z$status_details),  FUN = 'sum')
z = z |> arrange(desc(number_of_pages))|>
  na.omit()|>
  mutate(cumulative = cumsum(z$number_of_pages), percent_of_total = round(cumulative/sum(number_of_pages)*100,2))
z

```
```{r, echo =TRUE, warning=FALSE, message= FALSE, results =FALSE}
library('ggQC')
ggplot(data = z, aes(x=status_details, y=number_of_pages)) +
ggQC::stat_pareto(point.color = "#B03A2E",
                  point.size = 3,
                  line.color = "black",
                  line.size = 1.5,
                  bars.fill = "#888384") +
  theme_fivethirtyeight()+
  labs(title = 'Pareto chart of number of pages',
         subtitle = 'by error types',
         x = NULL)+
  scale_y_continuous(labels = comma_format(big.mark = ".", decimal.mark = ","),
                     name = "number of pages",
                     sec.axis = sec_axis(~./sum(z$number_of_pages), name = 'percent of total errors',
                                         labels = scales::percent_format(accuracy = 1)))+
  theme(axis.text.x = element_text(angle = 90,hjust = 1, vjust=0.5, size = 8),
        axis.title = element_text(face = 'bold'),
        plot.title = element_text(size = 18, face = 'bold'),
        plot.subtitle = element_text(size = 14))
```


![pareto](https://user-images.githubusercontent.com/94312553/222275887-e11f457b-013e-4558-b82a-726323fb48c6.png)

Graph indicates that in most of the cases (55.5% percent of all errors) *redirect error* occured. Another frequent issue is *Server error* (29.37% of pages with errors has that status). Together these two types of errors stands for 84.87% of all errors, so both should be taken into consideration in further analysis.


# **Summary**

To conclude:

* most of the domains are excluded - nearly 70%;
* most common status details for excluded pages are: *Page with redirect*,  *Crawled - currently not indexed* and *Alternate page with proper canonical tag*;
* pages associated with valid domains stands for only 20% of all pages;
* excluded domains have greater variance than valid domains;
* almost 85% of all errors are either redirect errors or server errors.



