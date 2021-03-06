---
title: "Bank Marketing Association Rule"
author: "Xolani Keith Mpala"
date: "2/26/2022"
output: html_document
---
# Association Rule

## Introduction
The digitalization of financial services enabled by widespread adoption of smart devices, cloud computing, data accessibility has increased a competitive landscape in the financial industry. Huge amount of electronic data is being maintained by financial institutions around the globe.  The huge size of these data bases makes it impossible for the financial institutions to analyze these data bases to retrieve useful information as per the need of the decision makers. One of the most important data mining techniques is association rule mining whose main purpose is to find frequent patterns, associations and relationship between various database items using different Algorithms.

This paper explores the use of this technique in a marketing bank data set. The data set gives information about a marketing campaign (phone calls) of a Portuguese banking institution. We analyze the data set by applying association rule mining to find the association between the marketing campaign and product uptake (whether the client subscribes for a bank term deposit or not). The rules enable the bank to find ways to look for future strategies in order to improve future marketing campaigns.


# About the Data

## Attribute Information of some of the dataset
•	Job: type of job ( 'admin.', 'blue-collar', 'management', 'retired', 'services', 'technician', 'other')

•	Marital: marital status ('divorced', 'married', 'single')

•	Education ('Primary','Secondary','Tertiary', 'unknown')

•	Default: has credit in default? ( 'no', 'yes')

•	Housing: has housing loan? ('no', 'yes')

•	Loan: has personal loan? ('no', 'yes') 

•	Contact: contact communication type ('cellular', 'telephone', 'unknown')

•	Duration: contact duration in seconds.

•	Campaign: number of contacts performed during this campaign.

•	Previous: number of contacts performed before this campaign.

•	Poutcome: outcome of the previous marketing campaign ('failure', 'other', 'success','unknown') 

### Load the libraries to begin the analysis
```{r}
#Load the libraries 
suppressPackageStartupMessages(library(arules))
suppressPackageStartupMessages(library(arulesViz))
suppressPackageStartupMessages(library(plotly))
suppressPackageStartupMessages(library(dplyr))
suppressPackageStartupMessages(library(ggplot2))
```

## Data Preprocessing
The numeric fields like ‘age’, ‘balance’, ‘day’, ‘duration’,’campaign’, ‘pdays’, ‘previous’ had to be discretized so that it could be used in the apriori algorithm.

```{r}
#Read in the data to be preprocessed
bank_data <- read.csv("bank.csv")
bd <- bank_data
```

Age was put into five age ranges of youth (18-24), young_adult (25-35), middle_aged_adults (36-54), senior-age (50-65), old (66-95). The histogram shows the frequency distribution of our continuous variable Age. The histogram also shows a density plot that visualises the distribution of variable age in our bank data set. 

```{r}
ggplot(bank_data, aes(age)) + geom_histogram(aes(y=..density..),color="black", fill="skyblue") + 
scale_x_continuous(breaks=c(18,24,35,50,65,95))+ theme_classic() + ggtitle('Age Distributition') + geom_density(alpha=.2, fill='black',color="#FF6666", linetype=0) 

bd$age <- cut(bd$age, breaks=c(18,24,35,50,65,Inf), labels=c('youth','young_adult','middle_aged_adult','senior_age','old'),include.lowest = TRUE)

```

The fields that contained “YES” or “NO” like ‘Default’, ’housing’, ’loan’, ‘deposit’ was transformed to “variable name = variable name = YES/NO” so that it would be clear in the rules what attribute is related to the YES or NO.

```{r}
#change variable names with yes/no.
bd$default <- dplyr::recode(bd$default, no='default=no',yes='default=yes',unknown='default=unknown')
bd$housing <- dplyr::recode(bd$housing, no='housing=no',yes='housing=yes',unknown='housing=unknown')
bd$loan <- dplyr::recode(bd$loan, no='loan=no',yes='loan=yes',unknown='loan=unknown')
bd$deposit <- dplyr::recode(bd$deposit, no='deposit=no',yes='deposit=yes')

```

The data for bank balance was therefore put into four separate ranges. A histogram was used to check the distribution of the data and also considering the minimum and the maximum value. The four categories were: -$6847-$0, $0-$4000, and $4000-$81204.

```{r}
ggplot(bank_data, aes(balance)) + geom_histogram(color="black", fill="#2E8BC0") + 
  scale_x_continuous(breaks=c(-6847,0,4000,10000))+theme_classic() + ggtitle('Bank Balance Distributition') 

min_balance <- min(bd$balance)
bd$balance = cut(bd$balance, breaks=c(min_balance, 0,4000,Inf), labels= c('-veBal','0-$4000', '$4000+'),include.lowest = TRUE)

```

The duration column shows time in seconds for the duration of the call. The column was converted into 3 categories of call time set in minutes: ‘less than 10 minutes’, ‘between 10 and 20 minutes’, ‘more than 20 minutes.A boxplot was plotted to check the distribution and outliers of variable duration.
```{r}
ggplot(bank_data, aes(duration,color=duration)) + 
  geom_boxplot(color="black", fill="#2E8BC0",outlier.color='#145DA0',outlier.alpha=0.2) +
  theme_classic() + ggtitle('Duration of a phone call in seconds')

min_duration <- min(bd$duration)
bd$duration <- cut(bd$duration, breaks = c(min_duration, 600,1200,Inf),labels= c('<10mins','10-20mins', '<20+mins'),include.lowest = TRUE)
```

Campaign shows number of contacts performed during this campaign and in the previous campaign if client was contacted. The continuous variable was transformed in three categorical variables of "1-3times if client was contacted 3 or less times, 4-6times if contact was made between 4 to 6 times, and finally +6 times if client was contacted more than 6 times A boxplot was plotted to check the distribution and outliers of variable campaign.

```{r}
ggplot(bd, aes(campaign,color=campaign)) + 
  geom_boxplot(color="black", fill="#2E8BC0",outlier.color='#145DA0',outlier.alpha=0.2) +
  theme_classic() + ggtitle('Number of times client contacted')

min_campaign <- min(bd$campaign)
bd$campaign <- cut(bd$campaign, breaks = c(min_campaign, 3,6, Inf), labels= c("1-3times", "4-6times","+6times"), include.lowest = TRUE)
```

The column ‘Previous’ shows the number of contacts performed before this campaign and ‘pdays’ shows number of days that passed by after the client was last contacted from a previous campaign. If ‘Previous’ = 0, this shows zero contact for the client and corresponds with -1 for ‘Pdays’ which shows client was not previously contacted. For this reason, the column pdays was discarded and previous column was transformed into two categories of PriorContact if client was contacted from a previous campaign and NoPriorContact if no contact was made.

```{r}
bd$previous <- cut(bd$previous, breaks= c(-Inf,0,Inf), labels= c('NoPriorContact','PriorContact'))
```

Column attributes pdays, days and month were discarded from our data set for association rule. This is because these columns highly affect our output target. For this reason, the variables were discarded in order to have a realistic predictive model. Character variables in the data set where converted into factors to apply association rule.
```{r}
# DELETE col Previous
bd <- select(bd, -pdays)
bd <- select(bd, -month)
bd <- select(bd,-day)
bd_test <- as.data.frame(unclass(bd), stringsAsFactors = TRUE)
```

## Summary statistics
Data set used in this study consists of 11162 transactions from “Bank Data set”. There are no missing values from the data sets. The job column has 1794 classified as other, 497 in education classified as other. 8326 and 537 in poutcome for previous campaign outcome were classified as unknown and other respectively while 2346 in contact column are classified as unknown.
The details summary statistics is shown below.

```{r}
summary(bd_test)
```

## Item Frequency
Item Frequency shows the items in the dataset that appear most. As we can see that most clients in the data set have not defaulted in their loan obligations followed by loan=no which shows the number of Clients that have not accessed any loan facilities.

```{r}
#item frequency
bd_matrix1 <- as(bd_test, "transactions")
itemFrequencyPlot(bd_matrix1, topN=25, type="relative", main="Item frequency", col="#2E8BC0")
```

## Apriori algorithm
The Apriori algorithm will be used. Apriori algorithm allows to reduce number of rules in the analysis as the minimum support level is defined at the beginning. Apriori heuristic assumes that if set of two items is frequent (meets minimum support condition), both of included items will be frequent too and that if given item is not frequent, any set of items including this item will not be frequent, too.

Association rules define relationship between occurrence of two or more items. They are characterized by a few of parameters.

Support which is a measure of how many times the joint itemset/rule appears in the database of use. 

The confidence level of a rule is how certain a rule is likely to happen. We can also define it as a ratio of support level of both consequent and antecedent items to support level of antecedent item (or items).

Lift is the most important measurement, creating levels of “interesting-ness” by determining if variables have influence on one another or if it was expected for these two variables to occur together. The higher it is, the higher the chance of co-occurrence of X and Y. Lift values higher than one stand for positive relationship of items or itemsets and value lower than one means 

 
## Creating the rules (general)
When creating the rules for this analysis, it was important to gather rules that would be useful for the business about their clients and what makes them subscribe or not subscribe for a term deposit.  Initially, a general overview of the entire data set was used to create rules to see if any deposit measure was present in the top 20 rules of the data set. These rules were created by the following parameters: (minimum support = 0.01, minimum confidence = 95%) and 5 as maximum number of items allowed in a rule. Then, a visual graph was produced to see what items were part of the biggest rulesets and get an over view.

```{r}
rules<- apriori(bd_test, parameter = list(supp=0.01, conf=0.95, maxlen = 5))
rules<- sort(rules, decreasing = FALSE, by ="lift")
plot(rules, measure = c("support", "lift"), shading = "confidence", main = "Support, Lift, and Confidence Top Rules")

generalRules20<- head(rules, n = 20, by = "lift")
inspect(generalRules20[1:10], linebreak = FALSE)
plot(generalRules20, method = "graph")
```

# Association Rule
## Rules for deposit = yes
After gathering general rules about the data set, the right-hand side (RHS) was set to either “deposit=deposit=yes” or “deposit=deposit=no” to examine what kind of clients subscribed for deposit after the campaign. The first set of rules were based on clients that subscribed for deposit and the parameters were set to support at 0.01 and confidence at 85%. Furthermore, the rules were limited to a maximum of six items per itemset to reduce redundancy with too many items per rule.

```{r}
#rules by confidence
yesrules<- apriori(bd_test, parameter = list(supp=0.01, conf = 0.85, maxlen = 6), 
                   appearance =list(default = "lhs", rhs="deposit=deposit=yes"),
                  control=list(verbose=F))
summary(yesrules)
```

With this chunk of code, we created 2810 rules. From the summary we can also determine the lift statistics for all the rules combined. Lift, as a remainder, is the dependency measure, where we compute chances of X and Y occurring together. From what we can see, our minimum lift is 1.794 which is above 1, therefore we can conclude that all the rules have the positive dependency.

## yesrules by Confidence

```{r}
#yesrules by confidence
conf_yesrules<- sort(yesrules , by ="confidence", decreasing = T )
inspect(conf_yesrules[1:10], linebreak = FALSE)
subrules <- head(conf_yesrules,8)
plot(subrules, method="graph", interactive=FALSE)

```


Highest confidence levels achieved by rules in the dataset exceed 90%. First two rules imply that clients who are in the senior age category, married, do not have a housing loan and the previous campaign outcome being a success will subscribe for a term deposit with 97,85% and 97.71% probability respectively. Unfortunately, Support levels in case of those two rules are around 1%, so this rule don’t occur too often – 135 and 128 times.   

## yesrules by Support

```{r}
#yesrules by support
supp_yesrules <- sort(yesrules, by='support', decreasing=TRUE)
inspect(supp_yesrules[1:10], linebreak = FALSE)
subrules <- head(supp_yesrules,10)
plot(subrules, method="graph", interactive=FALSE)
```

Highest support level achieved by rules determined in analyzed dataset is 10.11%. This level refers to clients who were contacted with cellphone, for a duration of 10-20 minutes and subscribed for a term deposit. This occurs in 10.11% of all 11162 clients – 1128 times.

## yesrules by Lift

```{r}
lift_yesrules <- sort(yesrules, by='lift', decreasing=TRUE)
inspect(lift_yesrules[1:10], linebreak = FALSE)
subrules <- head(lift_yesrules,10)
plot(subrules, method="graph", interactive=FALSE)
```

Rules can also be compared by lift values. Highest lift levels among rules in the dataset are 2.064 and 2.062 and imply that clients who are in the senior age category, married, do not have a housing loan, were contacted on cellular and previous outcome was success will subscribe for term deposit. In case of those two rules, confidence levels are 97.82% and 97.71%.

# Rules for deposit=deposit=no

```{r}
norules<- apriori(bd_test, parameter = list(supp=0.01, conf = 0.90, maxlen = 6), 
                  appearance =list(default = "lhs", rhs="deposit=deposit=no"),
                  control=list(verbose=F))
```

```{r}
#norules by Confidence
conf_norules<- sort(norules, by ="confidence", decreasing = TRUE)
inspect(conf_norules[1:5], linebreak = FALSE)
subrules <- head(conf_norules,8)
plot(subrules, method="graph", interactive=FALSE)
```

Highest confidence levels achieved by norules in the dataset exceed 90%. First rule imply that clients with a secondary education level,who  were contacted between 4 to 6 times during the campaign for less than 10 minutes call duration and with an unknown contact type will not subscribe for a term deposit with 96,55%  probability. Support levels in case of this rule are around 1%, so they occur 112 times.
Rule {3} imply that clients who are in the middle-age category, with a negative bank balance and were contacted for duration of less than 10 minutes with an unknown contact type, will not subscribe for a term deposit with 96.47% probability. Support levels  in case of this rule are around 1.22%, so they occur 137 times.

```{r}
#norules by support
supp_norules <- sort(norules, by='support', decreasing=TRUE)
inspect(supp_norules[1:5], linebreak = FALSE)
subrules <- head(supp_norules,6)
plot(subrules, method="graph", interactive=FALSE)
```

Rule {2} is the second highest support level achieved by rules determined in analyzed dataset with 15.39%. This level refers to clients who were contacted for a duration of less than 10 minutes, with contact type being unknown and had no prior contact in the previous campaign are likely not to subscribe for a term deposit. This occurs in 15.39% of all 11162 clients – 1718 times.

```{r}
#norules by lift
lift_norules <- sort(norules, by='lift', decreasing=TRUE)
inspect(lift_norules[1:5], linebreak = FALSE)
subrules <- head(lift_norules,7)
plot(subrules, method="graph", interactive=FALSE)
```

# Conclusion
Implementing the Association Rules measure can be extremely useful in the process of establishing behavior patterns. Apriori algorithm can be used to extract rules that can help in target marketing for Banks and financial institutions. The algorithm is a great technique for an extraction of useful information and remarks from huge dataset.

### Interesting rules for yes rules (Deposit=yes)
•	Customer with outcomes of the previous campaign equal success, who are in the senior age category (50-65 years), married and do not have a housing loan are most likely to subscribe to a term deposit. The client in this category can be targeted for marketing campaign.

•	Customer with outcomes of the previous campaign equal success and were contacted between 10 to 20 minutes are most likely to subscribe to a term deposit. The client in this category can also be targeted for marketing campaign.

### Interesting rules for no rules (Deposit=no)
•	Customer with a secondary education level, contacted between 4 to 6 times during the campaign for less than 10 minutes call duration and with an unknown contact type will likely not subscribe for a term deposit.


•	Customer who is in the middle-age category (36-50 years), with a negative bank balance and was contacted for duration of less than 10 minutes with an unknown contact type, will likely not subscribe for a term deposit.
