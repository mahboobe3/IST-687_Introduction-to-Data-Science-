# IST-687_Introduction-to-Data-Science-


---
title: "IST 687 Group 5 Final Project"
author: "Mahboobeh Hosseini,Elissa Carroll, Katie Pierce,  Sok Leng Chan"
date: "12/15/2020"
output: html_document
---

```{r chunck, include=FALSE}
knitr::opts_chunk$set(echo = FALSE, message = FALSE, warning = FALSE)
```

__Project Data__

Excel files: [link](https://www.kaggle.com/arslanali4343/top-personality-dataset?select=2018_ratings.csv)
Research Paper: [link](https://link.springer.com/article/10.1007/s10796-017-9782-y)

__Project Introduction__  
We chose a dataset from Kaggle that is ascertaining the effect of personality types on movie choices. The authors of this study asked the question: What drives people’s choices in the movies they watch?
To answer this question, they chose to look at the effects of personality traits on the likelihood that users would rate movies highly or not. 
Each user took a personality test and was assigned a score of 1-7 for each of five different personality traits. 
They were then given 12 movies to watch and based on their preference for those movies were assigned one of four metrics: Serendipity, Popularity, Diversity, Default. 
For each of these metrics they were also evaluated on how strongly they aligned with that metric. Each user was given an Assigned Condition of High, Medium, or Low depending on this. 
The assigned metric and assigned condition were combined to determine the list of movies that was given to each user

__Definitions__  
Openness: an assessment score (from 1 to 7) assessing user tendency to prefer new experience. 1 means the user has tendency NOT to prefer new experience, 7 means the user has tendency to prefer new experience.

Agreeableness: an assessment score (from 1 to 7) assessing user tendency to be compassionate and cooperative rather than suspicious and antagonistic towards others. 1 means the user has tendency to NOT be compassionate and cooperative. 7 means the user has tendency to be compassionate and cooperative.

Emotional Stability: an assessment score (from 1 to 7) assessing user tendency to have psychological stress. 1 means the user has tendency to have psychological stress, and 7 means the user has tendency to NOT have psychological stress.

Conscientiousness: an assessment score (from 1 to 7) assessing user tendency to be organized and dependable, and show self-discipline. 1 means the user does not have such a tendency, and 7 means the user has such tendency.

Extraversion: an assessment score (from 1 to 7) assessing user tendency to be outgoing. 1 means the user does not have such a tendency, and 7 means the user has such a tendency.

Assigned Metric: one of the follows (serendipity, popularity, diversity, default). Each user, besides being assessed their personality, was evaluated their preferences for a list of 12 movies manipulated with serendipity, popularity, diversity value or none (default option).

Assigned Condition: one of the follows (high, medium, low). Based on the assigned metric, and this assigned condition, the list of movies was generated for the users. For example: if the assigned metric is serendipity and the assigned condition is high, the movies in the list are highly serendipitous. 

Predictedratingx (x is from 1 to 12): the predicted rating of the corresponding movie_x for the user.

Is_Personalized: The response of the user to the question This list is personalized for me. Users answered on the 5-point Likert scale. (1: Strongly Disagree, 5: Strongly Agree).

Enjoy_watching: The response of the user to the question This list contains movies I think I enjoyed watching. Users answered on the 5-point Likert scale. (1: Strongly Disagree, 5: Strongly Agree)

__Load all packages necessary for the project__  
```{r library, include=FALSE}
require(tidyverse)
require(moments)
require(ggplot2)
require(modeest)
require(gridExtra)
require(ggpubr)
require(reshape2)
require(Metrics)
require(kernlab)
```

## Load and Clean Data

1. Load in the Personality Data and the ratings data.  
```{r load, include=FALSE}
#Personality data
Org_Personality <-read.csv("C:/Users/carro/Desktop/Fall 2020 Quarter/IST 687/Project/2018PersonalityData.csv", header = TRUE)

#Make another copy of the personality data for manipulating
Personality <- read.csv("C:/Users/carro/Desktop/Fall 2020 Quarter/IST 687/Project/2018PersonalityData.csv", header = TRUE)

#Ratings data
ratings <-read.csv("C:/Users/carro/Desktop/Fall 2020 Quarter/IST 687/Project/2018_ratings.csv", header = TRUE)

# Summary information about project data
summary(Org_Personality) #All variables other than userid, assigned.condition, and assigned.metric are numeric.
str(Org_Personality)

# Summary information about ratings data
summary(ratings) #timestamp and useri are character data.
str(ratings)
```

2. Edit column names & remove duplicate rows and spaces.  
```{r clm, include=FALSE}
# rename useri to userid in ratings data to match personality data column names
ratings <- ratings %>% rename(userid = useri)
colnames(ratings)

# Remove duplicate records in personality data
Personality <- Personality %>%
    select(userid:assigned.condition, is_personalized:enjoy_watching) %>%
    group_by(userid) %>%
    mutate(row_num = row_number()) %>%      
    filter(row_num==1) %>%
    select(-row_num)

# filter duplicate userid from original personality dataset
Org_Personality <- Org_Personality %>%
    group_by(userid) %>%
    mutate(row_num = row_number()) %>%
    filter(row_num==1) %>%
    select(-row_num)

#Remove spaces between words
Personality$assigned.metric <- gsub(" ", "", Personality$assigned.metric)
Personality$assigned.condition <- gsub(" ", "", Personality$assigned.condition)
Org_Personality$assigned.metric <- gsub(" ", "", Org_Personality$assigned.metric)
Org_Personality$assigned.condition <- gsub(" ", "", Org_Personality$assigned.condition)

#Look at each dataset
head(Personality)
head(Org_Personality)
head(ratings)
```

3. Join Personality data with users' actual ratings data (For analysis of predicted ratings accuracy).   
```{r join1, include=FALSE}
# join the ratings dataset with the Personality data
RatingPerson <- left_join(ratings,
          Personality,
          by = c("userid"="userid"))

#RatingPerson is a dataset that contains all of the information from both of the original datasets. All Personality traits, actual ratings, predicted ratings, assigned medtrics and assigned conditions, predicted ratings, personalization score, and enjoyment score
```

4. Reshape Org_Personality so that all of the movies for each user are in a single column labeled movie_id.  
```{r long, include=FALSE}
# Transform wide to long and assign a new name
PersonPredicted <- bind_rows(Org_Personality %>% select(userid, movie_id = movie_1, predicted_rating = predicted_rating_1),
          Org_Personality %>% select(userid, movie_id = movie_2, predicted_rating = predicted_rating_2), 
          Org_Personality %>% select(userid, movie_id = movie_3, predicted_rating = predicted_rating_3),
          Org_Personality %>% select(userid, movie_id = movie_4, predicted_rating = predicted_rating_4),
          Org_Personality %>% select(userid, movie_id = movie_5, predicted_rating = predicted_rating_5),
          Org_Personality %>% select(userid, movie_id = movie_6, predicted_rating = predicted_rating_6),
          Org_Personality %>% select(userid, movie_id = movie_7, predicted_rating = predicted_rating_7),
          Org_Personality %>% select(userid, movie_id = movie_8, predicted_rating = predicted_rating_8),
          Org_Personality %>% select(userid, movie_id = movie_9, predicted_rating = predicted_rating_9),
          Org_Personality %>% select(userid, movie_id = movie_10, predicted_rating = predicted_rating_10),
          Org_Personality %>% select(userid, movie_id = movie_11, predicted_rating = predicted_rating_11),
          Org_Personality %>% select(userid, movie_id = movie_12, predicted_rating = predicted_rating_12))
```

5. Join PersonPredicted with actual ratings.  
```{r join2, include=FALSE}
PredictedAndActual <-right_join(PersonPredicted, ratings, c("userid"="userid", "movie_id"="movie_id")) %>% group_by(userid)

head(PredictedAndActual)
#PredictedAndActual is a dataframe with just userid, movie id, predicted rating, tstamp, and rating.

PredictedActualPersonality <-right_join(Personality, PredictedAndActual, by=("userid"="userid")) %>% group_by(userid)

head(PredictedActualPersonality) #It is important to note that PredictedActualPersonality has NA's in the predicted_rating variable. This is EXPECTED as the actual ratings are for every movie the user reviewed and the predicted_rating is only for 12 movies per user as a sample.
```

6. Remove NA's from PredictedActualPersonality for use in later Analysis.   
```{r NAs, include=FALSE}
##remove the missing value in Personality 
PredAct_Personality <- PredictedActualPersonality %>%select(userid:extraversion,predicted_rating:rating,movie_id)%>% 
                          group_by(userid)%>% mutate(row_num = row_number()) %>% filter(row_num==1) %>% select(-row_num)

##remove NA in the Predicated rating which wasn't rated by the users
PredAct_Personality <- PredAct_Personality[complete.cases(PredAct_Personality[7]),]

# view the structure 
head(PredAct_Personality,10)
```

7. View dataframes we will use:  
```{r head, include=FALSE}
print("Personality")
head(Personality)
print("Org_Personality")
head(Org_Personality)
print("ratings")
head(ratings)
print("RatingPerson")
head(RatingPerson)
print("PredictedAndActual")
head(PredictedAndActual)
print("PredictedActualPersonality")
head(PredictedActualPersonality)
print("PredAct_Personality")
head(PredAct_Personality)
(PredAct_Personality)
```
  
## Personality Trait Distributions

We started our exploratory analysis by looking at the range of users in the data set by their personality types. For this, we used histograms.   
In the below graphs, the y axis represents the number of users, the x axis is the score for that personality type and the fill represents the assigned metric for each user. As you can see, each of the personality traits has a different distribution and none are an exact normal distribution.  

```{r Personality Histograms}
#Openness
Openness <- ggplot(Personality, aes(x=openness, fill=assigned.metric)) + geom_histogram() + ylab("Number of Users") +xlab("Openness Score")


#agreeableness
Agreeableness <- ggplot(Personality, aes(x=agreeableness, fill=assigned.metric)) + geom_histogram()+ ylab("Number of Users") +xlab("Agreeableness Score")

#emotional stability
Emotional_Stability<-ggplot(Personality, aes(x=emotional_stability, fill=assigned.metric)) + geom_histogram()+ ylab("Number of Users") +xlab("Emotional Stability Score")

#Conscientiousness
Conscientiousness<- ggplot(Personality, aes(x=conscientiousness, fill=assigned.metric)) + geom_histogram()+ ylab("Number of Users") +xlab("Conscientiousness Score")

#Extraversion
Extraversion <- ggplot(Personality, aes(x=extraversion, fill=assigned.metric)) + geom_histogram()+ ylab("Number of Users") +xlab("Extraversion Score")

#Put all of the plots together in two images

grid.arrange(Openness, Agreeableness, Emotional_Stability, ncol = 1)
grid.arrange(Conscientiousness, Extraversion, ncol=1)
```

### Descriptive Statistics  

We wanted to take the visualizations that were displayed in the histograms and look at values for central tendency and dispersion for each of the personality traits. 
```{r DescriptiveStats}
#Look at measures of central tendency

print("Mean of Personality Traits")
round(colMeans(PredAct_Personality[,2:6]), digits = 1)            #compute the mean of a variable
#While all fairly similar, Openness has the greatest average score (5.37)

print("Median of Personality Traits")
round(sapply(PredAct_Personality[,2:6], median), digits = 1)    #compute the median of a variable
#Openness has the highest central divide (5.5)

print("Mode of Personality Traits")
round(sapply(PredAct_Personality[,2:6], mfv), digits = 1)     #compute the mode of a variable
#Openness has the highest regularly occurring score (5)

                                                        #round function : rounds off the values


#Look at measures of dispersion
print("Standard Deviation of Personality Traits")
round(sapply(PredAct_Personality[,2:6], sd), digits = 1)        #compute the standard deviation  of a variable
#Openness has the tightest sd

print("Range of Personality Traits")
round(sapply(PredAct_Personality[,2:6], range), digits = 1)     #compute the range of a variable
#All have the same range

print("Skew of Personality Traits")
round(sapply(PredAct_Personality[,2:6], skewness), digits=2)

#Opennes, emotional stability, and conscientiousness all have negative skewness which indicates that their mean is less than their median and they are left-skewed.
#Agreeableness and Extraversion have positive skewness which indicates that their mean is greater than their median and they are right-skewed. 
#The greatest skew is for openness at -0.57 and Extraversion at 0.31

print("Kurtosis of Personality Traits")
round(sapply(Personality[,2:6], kurtosis), digits = 2)
#	The kurtosis of a normal distribution is 3; any kurtosis <3 is playkurtic (tends to produce fewer and less extreme outliers than the normal distribution); any kurtosis >3 is thought to produce more outliers than the normal distribution.

# The kurtosis of all personality traits except for Openness are less than 3 indicating that they contain fewer outliers and extreme values than a normal distribution. 



#How to Calculate Skewness & Kurtosis in R       (notes for skewness)
#A negative skew indicates that the tail is on the left side of the distribution, which extends towards more negative values.
#A positive skew indicates that the tail is on the right side of the distribution, which extends towards more positive values.
#A value of zero indicates that there is no skewness in the distribution at all, meaning the distribution is perfectly symmetrical.
```

#### 1. **Central Tendency:**
  + While all Personality traits had fairly similar measures, Openness has the highest average score at 5.37, the highest median at 5.5, and the highest mode at 5.    
  
#### 2. **Dispersion:** 
  + All ranges are the same as each trait has a max score of 7 and a minimum score of 1.   
  + Standard Deviations range from 1.0 to 1.5 with openness having the tightest standard deviation.
  + Positive skew: Agreeableness and Extraversion have a positive skew which indicates that their mean is greater than their median.   
  + Negative skew: Opennes, Emotional Stability, and Conscientiousness have negative skew which indicates that their mean is less than their median.  
  + The greatest skew is for openness at -0.57 and Extraversion at 0.31.  	
  + The kurtosis of a normal distribution is 3; any kurtosis <3 is playkurtic (tends to produce fewer and less extreme outliers than the normal distribution); any kurtosis >3 is thought to produce more outliers than the normal distribution.  
  + Each of the Personality traits, excpet Openness, has a kurtosis of less than three indicating that they produce fewer outliers and extreme values than a standard normal distribution.  


## How do the numbers of users in each of the traits differ?  

We next wanted to look at the numbers of users in each trait. Since each user is given a rating for every personality trait, we chose to break up these personality traits into zones. These zones are high scorers (score of 6-7), medium scorers (score of 3-5), and low scorers (score of 1-2).   

#### 1. Filter each of the personality traits based on our described "zones" of high, medium, or low.  
  + **Number of Users with Personality Trait Scores of 6-7**  
```{r HighTrait}
cat("Openness",435) 
cat("Agreeableness",110)
cat("Emotional Stability", 251)
cat("Conscientiousness",227) 
cat("Extraversion", 90)
```
  
  + **Number of Users with Personality Trait Scores of 3-5.5** 
```{r MediumTrait}
cat("Openness",602) 
cat("Agreeableness",830)
cat("Emotional Stability", 685)
cat("Conscientiousness",748) 
cat("Extraversion", 591)
```
 
 + **Number of Users with Personality Trait Scores of 1-2.5** 
```{r LowTrait}
cat("Openness",15) 
cat("Agreeableness",112)
cat("Emotional Stability", 116)
cat("Conscientiousness",77) 
cat("Extraversion", 371)
```  
  
  
```{r include=FALSE}
#High ratings (value > 6)
PredAct_Personality %>% filter(openness > 5.5) %>% nrow()
#There are 435 users with a rating between 6 -7 in openness
PredAct_Personality %>% filter(agreeableness > 5.5) %>% nrow()
#only 110
PredAct_Personality %>% filter(emotional_stability > 5.5) %>% nrow()
#251
PredAct_Personality %>% filter(conscientiousness > 5.5) %>% nrow()
#227
PredAct_Personality %>% filter(extraversion > 5.5) %>% nrow()
#90

#Medium Ratings (2.5 >value <6)   ## nrow : count the number of rows 
PredAct_Personality %>% filter(openness > 2.5) %>% filter(openness < 6) %>% nrow()
#602
PredAct_Personality %>% filter(agreeableness > 2.5) %>% filter(agreeableness < 6) %>% nrow()
#830
PredAct_Personality %>% filter(emotional_stability > 2.5) %>% filter(emotional_stability < 6) %>% nrow()
#685
PredAct_Personality %>% filter(conscientiousness > 2.5) %>% filter(conscientiousness < 6) %>% nrow()
#748
PredAct_Personality %>% filter(extraversion > 2.5) %>% filter(extraversion < 6) %>% nrow()
#591

print("Number of Users with Personality Trait Scores of 1-2")
#Low Ratings ( less 3 )
PredAct_Personality %>% filter(openness < 3) %>% nrow()
#15
PredAct_Personality %>% filter(agreeableness <3) %>% nrow()
#112
PredAct_Personality %>% filter(emotional_stability <3) %>% nrow()
#116
PredAct_Personality %>% filter(conscientiousness <3) %>% nrow()
#77
PredAct_Personality %>% filter(extraversion <3) %>% nrow()
#371
```
    
  + **The trait with the highest number of users with low scores is Extraversion, Agreeableness has the most users with a medium score, and Openness has the most users with high scores. All personality traits have more users in a medium scoring category than in high or low categories.**  
  + **At this point, we feel we have looked into the differences in assigned score for each of the Personality traits sufficiently. There are some clear differences in the numbers of users with high, medium, or low scores; as well as differences in the central tendency and dispersion. These can all be predominantly visualized in the histograms that we started this section with.**  
  + _We will now switch our focus to looking at the Assigned Conditions given to each User as well as User satisfaction in their assigned movie list._   

## Assigned Condition and User Satisfaction  

The assigned metric was used to curate lists of films for users to rate.  Users received values of "diversity," "popularity," "serendipity," or a default of "all."  In addition to the assigned metric, an assigned condition was used to scale the assigned metric.  For example, users with an assigned metric of serendipity and an assigned condition of "high" would receive highly serendipitious movies to rate.  

We expect to see the assigned metric provided some degree of customization in the list of movies sent to users, but also wanted to see if the addition of the assigned condition improved user satisfaction with the list of films. 
#### 1. What is the distribution of the assigned metric?  
```{r assCon1}
print("Number of Users in Each Assigned Metric")
tapply(Personality$userid,Personality$assigned.metric, length)
```
  + **Results: Very few people have no assigned metric and most were given a list of "popular" movies to watch and rate.**   

#### 2. What is the average rating by each assigned metric?  
```{r MetricPrint}
print("Average Rating of all users grouped together:")
round(mean(ratings$rating), digits=2)

print("Average Rating by Each Assigned Metric")
round(tapply(RatingPerson$rating, RatingPerson$assigned.metric, mean), digits = 2)
```
  + **Average ratings are very similar across assigned metrics and assigned metric category may not have had any influence in customizing the list of movies given to each User.**   

#### 3. Number of Users in each assigned condition by assigned metric:   
```{r AssMetAll , include=FALSE}
print("Number of Users with assigned metric of 'all' by assigned condition")
no_metric_users <- Personality %>%
  filter(assigned.metric == "all")
tapply(no_metric_users$userid, no_metric_users$assigned.condition, length)
```

```{r AssigMet}
print("Number of Users with assigned metric of 'diversity' by assigned condition")
diversity_users <- Personality %>%
  filter(assigned.metric == "diversity")
tapply(diversity_users$userid, diversity_users$assigned.condition, length)

print("Number of Users with assigned metric of 'popularity' by assigned condition")
popularity_users <- Personality %>%
  filter(assigned.metric == "popularity")
tapply(popularity_users$userid, popularity_users$assigned.condition, length)

print("Number of Users with assigned metric of 'popularity' by assigned condition")
serendipity_users <- Personality %>%
  filter(assigned.metric == "serendipity")
tapply(serendipity_users$userid, serendipity_users$assigned.condition, length)
```

  + **For each assigned metric, the distribution of assigned condition appears to be uniform.  This means as we examine the affect of assigned condition on ratings and user satisfaction our analysis will not be skewed due to a high percentage of users with a specific condition.**   

#### 4. Distribution of ratings by each assigned metric:  
```{r MetDist1, include=FALSE}
head(RatingPerson)

metric_all <- RatingPerson %>%
    filter(assigned.metric == "all")
all_by_condition <- ggplot(metric_all, aes(x=rating)) + geom_histogram(aes(fill=assigned.condition)) + theme_classic() + ggtitle("User Ratings for All Metric") + xlab("Rating") + ylab("Frequency") + labs(fill = "Assigned Condition")
all_by_condition
```

```{r MetDist2, include=FALSE}
metric_diversity <- RatingPerson %>%
    filter(assigned.metric == "diversity")
diversity_by_condition <- ggplot(metric_diversity, aes(x=rating)) + geom_histogram(aes(fill=assigned.condition)) + theme_classic() + ggtitle("User Ratings for Diversity Metric") + xlab("Rating") + ylab("Frequency") + labs(fill = "Assigned Condition")
diversity_by_condition

metric_popular <- RatingPerson %>%
    filter(assigned.metric == "popularity")
popular_by_condition <- ggplot(metric_popular, aes(x=rating)) + geom_histogram(aes(fill=assigned.condition)) + theme_classic() + ggtitle("User Ratings for Popular Metric") + xlab("Rating") + ylab("Frequency") + labs(fill = "Assigned Condition")
popular_by_condition

metric_serendipity <- RatingPerson %>%
    filter(assigned.metric == "serendipity")
serendipity_by_condition <- ggplot(metric_serendipity, aes(x=rating)) + geom_histogram(aes(fill=assigned.condition)) + theme_classic() + ggtitle("User Ratings for Serendipity Metric") + xlab("Rating") + ylab("Frequency") + labs(fill = "Assigned Condition")
serendipity_by_condition
```

```{r MetDist3, include=TRUE}
grid.arrange(diversity_by_condition, popular_by_condition, serendipity_by_condition, nrow=2)
```

  + **From the distributions of ratings for each assigned metric, it does not appear that any assigned condition resulted in higher ratings. See the values for each below.**      

```{r MetDist4, include=TRUE}
print("Average ratings for diversity metric:")
tapply(metric_diversity$rating, metric_diversity$assigned.condition, mean)
print("Average ratings for popularity metric:")
tapply(metric_popular$rating, metric_popular$assigned.condition, mean)
print("Average ratings for serendipity metric:")
tapply(metric_serendipity$rating, metric_serendipity$assigned.condition, mean)
```

  + **The percentage of ratings for high, low, and medium assigned conditions were consistent throughout the distributions of ratings.**     
  + **The customization with the assigned metric and condition did not affect user ratings.  We then investigated what effected User film satisfaction.**  

## User Satisfaction Analysis

Since customization with the assigned metric and condition did not seem to impact user ratings, we then investigated any possible relationship between user enjoyment and their list of films.  

#### 1. What is the distribution of satisfaction?     
```{r UserSat1}
Personality <- Personality %>%
   add_count(userid)

enjoy_watching_assigned_metric <- ggplot(Personality,aes(x=enjoy_watching, y=n, fill=assigned.metric)) + geom_bar(stat="identity") + theme_classic() + ggtitle("User Enjoyment by Assigned Metric") + xlab("Enjoyed Watching") + ylab("Users") + labs(fill = "Assigned Metric")

enjoy_watching_assigned_metric
```

**Overall, the vast majority of users enjoyed the list of films they were assigned to rate. However, it does not appear that the assigned metric influenced whether users enjoyed their list of movies. The proportion of satisfied users is consistent for each of the enjoyment responses.**    


  + **We next examined the distribution of user satisfaction for each assigned metric by the assigned condition.**       
```{r UserSat2, include=FALSE}
# for each assigned metric did the assigned condition improve user satisfaction?
# create Personality dataframes for each of the assigned metrics to plot user satisfaction vs the assigned condition

person_diversity <- Personality %>%
    filter(assigned.metric == "diversity") %>%
    distinct(userid, enjoy_watching, assigned.condition) %>%
    add_count(userid)

person_popularity<- Personality %>%
    filter(assigned.metric == "popularity") %>%
    distinct(userid, enjoy_watching, assigned.condition) %>%
    add_count(userid)

person_serendipity <- Personality %>%
    filter(assigned.metric == "serendipity") %>%
    distinct(userid, enjoy_watching, assigned.condition) %>%
    add_count(userid)

# bar plot of satisfaction color coded by assigned condition

# satisfaction for users with an assigned metric of diversity
diversity_enjoy_bar <- ggplot(person_diversity,aes(x=enjoy_watching, y=n, fill=assigned.condition)) + geom_bar(stat="identity") + theme_classic() + ggtitle("Diversity Metric") + xlab("Enjoyed Watching") + ylab("Users") + theme(legend.position="none")

# satisfaction for users with an assigned metric of popularity
popularity_enjoy_bar <- ggplot(person_popularity,aes(x=enjoy_watching, y=n, fill=assigned.condition)) + geom_bar(stat="identity") + theme_classic() + ggtitle("Popularity Metric") + xlab("Enjoyed Watching") + ylab("Users") + theme(legend.position="none")

# satisfaction for users with an assigned metric of serendipity
serendipity_enjoy_bar <- ggplot(person_serendipity, aes(x=enjoy_watching, y=n, fill=assigned.condition)) + geom_bar(stat="identity") + theme_classic() + ggtitle("Serendipity Metric") + xlab("Enjoyed Watching") + ylab("Users") + theme(legend.position = "none")
```

```{r UserSat3, include=TRUE}
grid.arrange(diversity_enjoy_bar, popularity_enjoy_bar, serendipity_enjoy_bar, nrow=2)
```

  + **Again, it does not appear that the assigned conditions had any influence in whether users enjoyed their movies.**    
  + **While the effects by Assigned condition are not significant, they are playing some role in enjoy watching score. The enjoy watching score is highest when the diversity condition is medium, the popularity condition is low and the serendipity is either low or high in the high enjoy watching score.**  
  + **If we compare the rating vs enjoy watching score by the assigned condition we see that When diversity is in a medium condition, it has the highest average rating. For popularity, the average rating is highest when it is in the high condition and for serendipity the average rating is highest in the medium condition.**  
  + **These results tell us that if people highly rate a movie it does not necessarily mean that they enjoyed it the most.**  
  + **As an example, people might think that if a movie is not as popular, it should be rated higher because it might not have received enough attention. However, they might enjoy watching a popular movie more due to various reasons such as being able to watch it with more people because everyone likes that type of movie.   + After looking at assigned condition, assigned metric, and their effect upon user enjoyment of movies, we next want to turn our attention to the ratings that each user gave.**   

## Personality and User Rating 

#### 1. Generate graphs of average user rating vs user's personality scores.    
```{r PUser1, include=FALSE}
#If we include the below bar charts, explain in a bullet point before them what the X axis is!!!!!

# Average rating vs openness scores
openness_scores <- names(tapply(RatingPerson$rating, RatingPerson$openness, mean))

open_bar <- barplot(tapply(RatingPerson$rating, RatingPerson$openness, mean), xlab="Openness Scores", ylab="Average Rating",ylim = c(0, 4), col="#69b3a2", xaxt="n")

# Average rating vs agreeableness scores
agree_scores <- names(tapply(RatingPerson$rating, RatingPerson$agreeableness, mean))

agree_bar <- barplot(tapply(RatingPerson$rating, RatingPerson$agreeableness, mean), xlab="Agreeableness Scores", ylab="Average Rating",ylim = c(0, 4), col="#69b3a2",xaxt="n")


# Average rating vs emotional stability scores
stability_scores <- names(tapply(RatingPerson$rating, RatingPerson$emotional_stability, mean))

stab_bar<- barplot(tapply(RatingPerson$rating, RatingPerson$emotional_stability, mean), xlab="Emotional Stability Scores", ylab="Average Rating",ylim = c(0, 4), col="#69b3a2",xaxt="n")

# Average rating vs conscientiousness scores
conscientious_scores <- names(tapply(RatingPerson$rating, RatingPerson$conscientiousness, mean))

conscientious_bar<-barplot(tapply(RatingPerson$rating, RatingPerson$conscientiousness, mean), xlab="Conscientiousness Scores", ylab="Average Rating",ylim = c(0, 4), col="#69b3a2",xaxt="n")

# Average rating vs extraversion scores
extraversion_scores <- names(tapply(RatingPerson$rating, RatingPerson$extraversion, mean))

extra_bar<- barplot(tapply(RatingPerson$rating, RatingPerson$extraversion, mean), xlab="Extraversion Scores", ylab="Average Rating",ylim = c(0, 4), col="#69b3a2",xaxt="n")
```

```{r PUserSok, include=TRUE}
#get the average for each type of personality
avg_o<-mean(RatingPerson$openness)
avg_a<-mean(RatingPerson$agreeableness)
avg_e<-mean(RatingPerson$emotional_stability)
avg_c<-mean(RatingPerson$conscientiousness)
avg_ex<-mean(RatingPerson$extraversion)
#store avg of each personality in avg
avg <- rbind.data.frame(avg_o,avg_a,avg_e,avg_c,avg_ex)
#rename all row name
row.names(avg)<-c("AVG_openess","AVG_agreeable","AVG_emotionalStability","AVG_conscientious","AVG_extraversion")
#change row to column
avg$AVG<-row.names(avg)
#rename column name
colnames(avg)<-c("AVG","TYPE")
ggplot(avg,aes(x=TYPE,y=AVG,fill=AVG))+ geom_bar(position ="stack",stat="identity")+labs(title="Avg of each type of personality")
```

  + **Results: Users with high or low scores in each personality trait had the same average rating.**  
 
#### 2. It is possible that there are patterns that we are not seeing due to looking at too broad of group classifications. To delve into this, we decided to look just at users who had given a rating of five to 100 or more movies in their list. We utilized bar graphs to visualize any patterns.      

```{r PUser2, include=FALSE}
# create a table that filters users with more than 100 5 ratings and return personality scores
HighRatingPerson2 <- RatingPerson %>% 
    filter(rating >= 5) %>% 
    add_count(userid) %>%
    filter(n >= 100) %>%
    distinct(userid, openness, agreeableness, emotional_stability, conscientiousness, extraversion)
```


```{r PUserBar, include=FALSE}
# bargraphs of personality traits for users with high ratings

# openness scores for users with high ratings
HighRatingOpenness <- HighRatingPerson2 %>%
       distinct(userid, openness) %>%
       count(openness, userid)

# bar graph of users with high ratings by openness scores
high_rating_openness <- ggplot(HighRatingOpenness, aes(x=openness, y=n)) + geom_bar(stat="identity", fill="deepskyblue") + theme_classic() + ggtitle("High Rating Users") + xlab("Openness") + ylab("Users")

# agreeableness scores for users with high ratings
HighRatingAgree <- HighRatingPerson2 %>%
       distinct(userid, agreeableness) %>%
       count(agreeableness, userid)

# bar graph of users with high ratings by agreeableness scores
high_rating_agree <- ggplot(HighRatingAgree, aes(x=agreeableness, y=n)) + geom_bar(stat="identity", fill="magenta3") + theme_classic() + ggtitle("High Rating Users") + xlab("Agreeableness") + ylab("Users")

# emotional_stability scores for users with high ratings
HighRatingEmotion <- HighRatingPerson2 %>%
       distinct(userid, emotional_stability) %>%
       count(emotional_stability, userid)

# bar graph of users with high ratings by emotional stability scores
high_rating_emotion <- ggplot(HighRatingEmotion, aes(x=emotional_stability, y=n)) + geom_bar(stat="identity", fill="slate blue") + theme_classic() + ggtitle("High Rating Users") + xlab("Emotional Stability") + ylab("Users")


# conscientiousness scores for users with high ratings
HighRatingConscientious <- HighRatingPerson2 %>%
       distinct(userid, conscientiousness) %>%
       count(conscientiousness, userid)

# bar graph of users with high ratings by conscientiousness scores
high_rating_conscientious <- ggplot(HighRatingConscientious, aes(x=conscientiousness, y=n)) + geom_bar(stat="identity", fill= "light green") + theme_classic() + ggtitle("High Rating Users") + xlab("Conscientiousness") + ylab("Users")

# extraversion scores for useres with high ratings
HighRatingExtraversion <- HighRatingPerson2 %>%
       distinct(userid, extraversion) %>%
       count(extraversion, userid)

# bar graph of users with high ratings by extraversion scores
high_rating_extraversion <- ggplot(HighRatingExtraversion, aes(x=extraversion, y=n)) + geom_bar(stat="identity", fill= "pink") + theme_classic() + ggtitle("High Rating Users") + xlab("Extraversion") + ylab("Users")
```

```{r PUserBar2, include=TRUE}
grid.arrange(high_rating_openness, high_rating_agree, high_rating_emotion, high_rating_conscientious, high_rating_extraversion, nrow=2)
```
  
  + **While these distributions may look different from each other, when compared to the distribution of users within each score, no new patterns emerged. **
    
  + **We also wanted to see if there were noticeable differences when we grouped the user by filtering Personality traits by high or low score buckets.**  

#### 1. Filter users by high/low scores in personality traits (High = score > average, Low = score < average).   
#### 2. Create density diagram for predicted vs. actual rate base on the different types of personality. 
```{r PUserDnesity1, include=FALSE}
#Separate the types of personality
Openness <- PredAct_Personality %>% select(userid,openness,predicted_rating:movie_id)  %>% group_by(userid)
agreeableness<- PredAct_Personality %>% select(userid,agreeableness,predicted_rating:movie_id)  %>% group_by(userid)
emotional_stability<- PredAct_Personality %>% select(userid,emotional_stability,predicted_rating:movie_id) %>% group_by(userid)
conscientiousness<- PredAct_Personality %>% select(userid,conscientiousness,predicted_rating:movie_id) %>% group_by(userid)
extraversion<- PredAct_Personality %>% select(userid,extraversion,predicted_rating:movie_id) %>% group_by(userid)
#get the average of openness and store in Avg_Openness
Avg_Openness <- mean(Openness$openness)
Avg_agreeable <- mean(agreeableness$agreeableness)
Avg_emotion <- mean(emotional_stability$emotional_stability)
Avg_conscientious <- mean(conscientiousness$conscientiousness)
Avg_extraversion <- mean(extraversion$extraversion)

# filter by high/low scores in personality traits? 
##  Categorize the data higher or lower than average 

cat_openness <- Openness %>% select(userid:movie_id) %>% mutate(cat =ifelse(openness>=Avg_Openness ,yes="High",no="Low")) 
cat_agreeableness <- agreeableness%>% select(userid:movie_id) %>% mutate(cat =ifelse(agreeableness>=Avg_agreeable ,yes="High",no="Low"))
cat_emotion <- emotional_stability%>% select(userid:movie_id) %>% mutate(cat =ifelse(emotional_stability>=Avg_emotion ,yes="High",no="Low"))
cat_conscientious <- conscientiousness%>% select(userid:movie_id) %>% mutate(cat =ifelse(conscientiousness>=Avg_conscientious ,yes="High",no="Low"))
cat_extraversion <- extraversion%>% select(userid:movie_id) %>% mutate(cat =ifelse(extraversion>=Avg_extraversion ,yes="High",no="Low"))
 
## change the Categorize to be factor 
cat_openness$cat <- as.factor(cat_openness$cat)
cat_agreeableness$cat <- as.factor(cat_agreeableness$cat)
cat_emotion$cat <- as.factor(cat_emotion$cat)
cat_conscientious$cat <- as.factor(cat_conscientious$cat)
cat_extraversion$cat <- as.factor(cat_extraversion$cat)

#count the number of each group (high and low) by class
cat_openness %>% group_by(cat) %>% summarise(openness_num_class = n()) %>% arrange(desc(openness_num_class))
cat_agreeableness%>% group_by(cat) %>% summarise(agreeable_num_class = n()) %>% arrange(desc(agreeable_num_class))
cat_emotion %>% group_by(cat) %>% summarise(emotion_num_class = n()) %>% arrange(desc(emotion_num_class))
cat_conscientious %>% group_by(cat) %>% summarise(concientious_num_class = n()) %>% arrange(desc(concientious_num_class))
cat_extraversion %>% group_by(cat) %>% summarise(extraversion_num_class = n()) %>% arrange(desc(extraversion_num_class))
```

#### 3. Clarify the difference between Predicted vs. actual rate based on the different types of personality.  
```{r PUserDemsity2, include=FALSE}
##actual rate as x
##create density diagram to clarify the difference for different type of personality  (based on the average of openness, High > avg , Low <age)
density_openness <- ggdensity(cat_openness, x = "rating",add = "mean",rug = T ,color = "cat", 
                              fill = "cat",palette = "Set2", title="Openness Actual Rate")


density_agreealbe <- ggdensity(cat_agreeableness, x = "rating",add = "mean",rug = T ,color = "cat", 
                               fill = "cat",palette = "Set2", title="Agreeableness Actual Rate")

density_emotion <- ggdensity(cat_emotion, x = "rating",add = "mean",rug = T ,color = "cat", 
                             fill = "cat",palette = "Set2", title="Emotional Stability Actual Rate")

density_conscientious <- ggdensity(cat_conscientious, x = "rating",add = "mean",rug = T ,color = "cat", 
                                   fill = "cat",palette = "Set2", title="Conscientious Actual Rate")

density_extraversion <- ggdensity(cat_extraversion, x = "rating",add = "mean",rug = T ,color = "cat",
                                  fill = "cat",palette = "Set2", title="Extraversion Actual Rate")

##Predictive rate as x
##create density diagram to clarify the difference for different type of personality  (based on the average of openness, High > avg , Low <age)

P_density_openness <- ggdensity(cat_openness, x = "predicted_rating",add = "mean",rug = T ,color = "cat",
                                fill = "cat",palette = " Jco", title="Openness Predicted Rate")

P_density_agreealbe <- ggdensity(cat_agreeableness, x = "predicted_rating",add = "mean",rug = T ,color = "cat",
                                 fill = "cat",palette = "Jco", title="Agreeableness Predicted Rate")

P_density_emotion <- ggdensity(cat_emotion, x = "predicted_rating",add = "mean",rug = T ,color = "cat", 
                               fill = "cat",palette = "Jco", title="Emotional Stability Predicted Rate")

P_density_conscientious <- ggdensity(cat_conscientious, x = "predicted_rating",add = "mean",rug = T ,color = "cat", 
                                 fill = "cat",palette = "Jco", title="Conscientious Predicted Rate")

P_density_extraversion <- ggdensity(cat_extraversion, x = "predicted_rating",add = "mean",rug = T ,color = "cat", 
                             fill = "cat",palette = "Jco", title="Extraversion Predicted Rate")
``` 
  + **In the below density plots, the vertical dashed lines represent the mean for each of the personality traits depicted. We are looking at density plots of high and low scoring users ratings.**  
  
```{r PUserDensity3, include=TRUE}
## combine the actual and predicted diagram in one  (base on the different type of personality)
ggarrange( density_openness ,P_density_openness ,ncol = 2, nrow = 1,hjust=30,vjust =5,widths = c(1,1),align="h")
ggarrange( density_agreealbe ,P_density_agreealbe ,ncol = 2, nrow = 1,hjust=25,vjust =5,widths = c(1,1),align="h")
ggarrange( density_emotion ,P_density_emotion ,ncol = 2, nrow = 1,hjust=25,vjust =5,widths = c(1,1),align="h")
ggarrange( density_conscientious,P_density_conscientious ,ncol = 2, nrow = 1,hjust=25,vjust =5,widths = c(1,1),align="h")
ggarrange( density_extraversion,P_density_extraversion ,ncol = 2, nrow = 1,hjust=25,vjust =5,widths = c(1,1),align="h")
```

  + **These density plots show that people with an extreme high level of openness scored above Average compared to the other 4 types of personality (agreeable, emotional stability, conscientious, and extraversion). Their predictive rates for their movies match best with their actual ratings.**  
 
 
  + **Now that we have explored the relationship between personality traits, assigned metrics, and assigned conditions with ratings and user satisfaction we will next explore the Predicted and Actual ratings.**   
  
## Predicted and Actual Ratings  

Each user was given over six thousand films to rate; this data was captured as their Actual Rating. Additionally, for each user, a random sample of 12 of their rated movies was chosen for the researchers to attempt to predict the ratings. We wanted to look at the differences between actual and predicted ratings to pull out any patterns that might occur.  
We found that the best way to truly see patterns was to use the "Bins" we created earlier of high, medium, and low scores for each of the personality traits.  
  
#### 1. Separate the type of personality  and making the diagram separately 
```{r PredAct1, include=FALSE}
# separate the type of personality 
Openness <- PredAct_Personality %>% select(userid,openness,predicted_rating:movie_id)  %>% group_by(userid)
agreeableness<- PredAct_Personality %>% select(userid,agreeableness,predicted_rating:movie_id)  %>% group_by(userid)
emotional_stability<- PredAct_Personality %>% select(userid,emotional_stability,predicted_rating:movie_id) %>% group_by(userid)
conscientiousness<- PredAct_Personality %>% select(userid,conscientiousness,predicted_rating:movie_id) %>% group_by(userid)
extraversion<- PredAct_Personality %>% select(userid,extraversion,predicted_rating:movie_id) %>% group_by(userid)
```
  
#### 2. Categorize the data by high , medium and low utilizing ifelse statements.  
```{r PredAct2, include=FALSE}
##  Categorize the data by high , medium and low 
HML_Openness<- Openness %>% select(userid:movie_id)%>% mutate(CLASS=ifelse(openness>5.5,yes="High",
                                                       no=ifelse(openness <3 ,yes="Low", no="Medium"))) 

HML_agreeable <-  agreeableness %>% select(userid:movie_id) %>%  mutate(CLASS=ifelse(agreeableness>5.5,yes="High",
                                                                no=ifelse(agreeableness<3,yes="Low", no="Medium")))

HML_emotion <-  emotional_stability %>% select(userid:movie_id) %>%  mutate(CLASS=ifelse(emotional_stability>5.5,yes="High",
                                                                  no=ifelse(emotional_stability<3,yes="Low", no="Medium")))

HML_concientious <-  conscientiousness %>% select(userid:movie_id) %>% mutate(CLASS=ifelse(conscientiousness>5.5,yes="High",no=ifelse(conscientiousness<3,yes="Low", no="Medium")))

HML_extraversion<-  extraversion %>% select(userid:movie_id) %>%  mutate(CLASS=ifelse(extraversion>5.5,yes="High",
                                                                         no=ifelse(extraversion<3,yes="Low", no="Medium")))
```

#### 3. Set boundaries for predicted ratings.  
```{r PredAct3, include=FALSE}
#Boundaries are necessary for us to establish in order to use bar charts to analyze this data. When comparing diagrams if we don't set boundaries for predicted rate, it is difficult to read histogram  for actual rate. And if  we use a bar chart for actual rate, it can perform well, but it is has trouble for predicted rate if we don't set boundaries. 

# check the min and max value  in order to set the boundaries(range2.5 -5.5)
max(HML_extraversion$predicted_rating)
min(HML_extraversion$predicted_rating)


##set the class boundaries for predicted rate in order to present the diagram 
#(the range of each group should be 2.5-3,3-3.5,3.5-4,4-4.5, 4.5-5,5-5.5)
New_HML_Openness<- HML_Openness  %>% select(userid:CLASS) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

New_HML_agreeable<- HML_agreeable %>% select(userid:CLASS) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

New_HML_emotion<- HML_emotion  %>% select(userid:CLASS) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

New_HML_concientious <- HML_concientious %>% select(userid:CLASS) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

New_HML_extraversion <- HML_extraversion  %>% select(userid:CLASS) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

## change the class and the boundaries to be factor 
New_HML_Openness$CLASS<- as.factor(New_HML_Openness$CLASS)
New_HML_agreeable$CLASS<- as.factor(New_HML_agreeable$CLASS)
New_HML_emotion$CLASS  <- as.factor(New_HML_emotion$CLASS)
New_HML_concientious$CLASS <- as.factor(New_HML_concientious$CLASS)
New_HML_extraversion$CLASS <- as.factor(New_HML_extraversion$CLASS)
New_HML_Openness$boundaries<- as.factor(New_HML_Openness$boundaries)
New_HML_agreeable$boundaries<- as.factor(New_HML_agreeable$boundaries)
New_HML_emotion$boundaries  <- as.factor(New_HML_emotion$boundaries)
New_HML_concientious$boundaries <- as.factor(New_HML_concientious$boundaries)
New_HML_extraversion$boundaries <- as.factor(New_HML_extraversion$boundaries)

#arrange the order by boundaries (Ascending )
New_HML_Openness<- New_HML_Openness[order(New_HML_Openness$boundaries),]
New_HML_agreeable<- New_HML_agreeable[order(New_HML_agreeable$boundaries),]
New_HML_emotion<- New_HML_emotion[order(New_HML_emotion$boundaries),]
New_HML_concientious<- New_HML_concientious[order(New_HML_concientious$boundaries),]
New_HML_extraversion<- New_HML_extraversion[order(New_HML_extraversion$boundaries),]
```
  
#### 4. Number of Users in each Class    
  
```{r PredAct4, include=TRUE}
# show the number of  each group by class
New_HML_Openness %>% group_by(CLASS) %>% summarise(openness_num_class = n()) %>% arrange(desc(openness_num_class))
New_HML_agreeable %>% group_by(CLASS) %>% summarise(agreeable_num_class = n()) %>% arrange(desc(agreeable_num_class))
New_HML_emotion %>% group_by(CLASS) %>% summarise(emotion_num_class = n()) %>% arrange(desc(emotion_num_class))
New_HML_concientious %>% group_by(CLASS) %>% summarise(concientious_num_class = n()) %>% arrange(desc(concientious_num_class))
New_HML_extraversion %>% group_by(CLASS) %>% summarise(extraversion_num_class = n()) %>% arrange(desc(extraversion_num_class))
```
  
  
#### 5. Create bar charts to visualize the distribution of number of Users in relation to the score they received being low, medium, or high.  
```{r PredActBar, include=FALSE}
# Bar plot for each personality (x=actual rating)
Barchar_openness <-   ggplot(New_HML_Openness, aes(rating,fill=CLASS))+ geom_bar(position ="stack")+
                      facet_wrap(~CLASS)+scale_fill_brewer(palette="Set2")+theme_minimal()+
                      labs( y="number of Users",title ="The allocation of Users is sorted by High,Medium and Low Score of Openness")
                      

Barchar_agreeable <-  ggplot(New_HML_agreeable,aes(rating,fill=CLASS))+ geom_bar(position ="stack")+
                      facet_wrap(~CLASS)+scale_fill_brewer(palette="Set2")+theme_minimal()+
                      labs( y="number of Users",title= "The allocation of Users is sorted by High,Medium and Low Score of agreeable") 
                    

Barchar_emotion <-   ggplot(New_HML_emotion,aes(rating,fill=CLASS))+ geom_bar(position ="stack")+
                      facet_wrap(~CLASS)+scale_fill_brewer(palette="Set2")+theme_minimal()+
                      labs( y="number of Users",title= "The allocation of Users is sorted by High,Medium and Low Score of Emotional stable") 
                    
Barchar_conscientious<- ggplot(New_HML_concientious,aes(rating,fill=CLASS))+ geom_bar(position ="stack")+
                        facet_wrap(~CLASS)+scale_fill_brewer(palette="Set2")+theme_minimal()+
                        labs( y="number of Users",title= "The allocation of Users is sorted by High,Medium and Low Score of conscientious")
                        

Barchar_extraversion<-  ggplot(New_HML_extraversion,aes(rating,fill=CLASS))+ geom_bar(position ="stack")+
                        facet_wrap(~CLASS)+scale_fill_brewer(palette="Set2")+theme_minimal()+
                        labs( y="number of Users",title= "The allocation of Users is sorted by High,Medium and Low Score of extraversion") 
                      

# Bar plot for each personality (x=Predicted rating)
P_Barchar_openness <-   ggplot(New_HML_Openness,aes(boundaries, fill=CLASS)) + geom_bar(position = "stack") + facet_wrap(~CLASS) + 
                        scale_fill_brewer(palette="Set2") + theme_minimal()+labs( y="number of Users", x="Predicted rating",
                        title =  "The allocation of Users is sorted by High,Medium and Low Score of Openness") 

P_Barchar_agreeable <-  ggplot(New_HML_agreeable,aes(boundaries, fill=CLASS)) + geom_bar(position = "stack") + facet_wrap(~CLASS) + 
                        scale_fill_brewer(palette="Set2") + theme_minimal()+
                        labs( y="number of Users",title= "The allocation of Users is sorted by High,Medium and Low Score of agreeable")
                       

P_Barchar_emotion <-   ggplot(New_HML_emotion,aes(boundaries, fill=CLASS)) + geom_bar(position = "stack") + facet_wrap(~CLASS) + 
                        scale_fill_brewer(palette="Set2") + theme_minimal()+ labs( y="number of Users", x="Predicted rating",
                        title= "The allocation of Users is sorted by High,Medium and Low Score of Emotional Stablity") 
                        

P_Barchar_conscientious<- ggplot(New_HML_concientious,aes(boundaries, fill=CLASS)) + geom_bar(position = "stack") + facet_wrap(~CLASS) + 
                          scale_fill_brewer(palette="Set2") + theme_minimal()+ labs( y="number of Users", x="Predicted rating",
                          title= "The allocation of Users is sorted by High,Medium and Low Score of conscientious") 
                        

P_Barchar_extraversion<-  ggplot(New_HML_extraversion,aes(boundaries, fill=CLASS)) + geom_bar(position = "stack") + facet_wrap(~CLASS) + 
                          scale_fill_brewer(palette="Set2") + theme_minimal()+ labs( y="number of Users", x="Predicted rating",
                          title= "The allocation of Users is sorted by High,Medium and Low Score of extraversion")
```
  
```{r PredActBar2, include=TRUE}
## combine the actual and predicted diagram in one  (base on the different type of personality)
ggarrange( Barchar_openness ,P_Barchar_openness ,ncol = 1, nrow = 2,hjust=25,vjust =5)
ggarrange( Barchar_agreeable ,P_Barchar_agreeable ,ncol = 1, nrow = 2,hjust=25,vjust =5)
ggarrange( Barchar_emotion ,P_Barchar_emotion ,ncol = 1, nrow = 2,hjust=25,vjust =5)
ggarrange( Barchar_conscientious,P_Barchar_conscientious ,ncol = 1, nrow = 2,hjust=25,vjust =5)
ggarrange( Barchar_extraversion,P_Barchar_extraversion ,ncol = 1, nrow = 2,hjust=25,vjust =5)
```
  
  
  + **When users are grouped into categories of low, medium, and high by their personality traits we can see that the predicted rating pattern is similar to the actual ratings given by the users.**  
  + **More specifically, if someone were to score highly in any one of the personality traits, the likelihood of the predicted rating mirroring the actual rating increases.**

  

```{r , include=FALSE}
# How similar are the predicted and actual ratings generated by the reasearchers?  

#1. We ascertained the normal range of difference between predicted and actual ratings.
#  + This difference for all users fell between 0 and 1.  
# + We chose to look at users whose predicted and actual ratings differed by no more than +/-0.05 points. 
# + There were 200 users in the dataset who fit these criteria. 
#find the variance between predicted and actual rating and store in variance
variance <- PredAct_Personality$predicted_rating - PredAct_Personality$rating

# add new column "variance into PredAct_Personality
PredAct_Personality$varaiance <- variance

##find the range of variance 
r <- range(PredAct_Personality$varaiance)

##print the range of variance of predicated vs actual rating
paste("Range of predicated vs actual rating : min var: ",r[1],"max var:",r[2])

## select the users which have the least variance in the dataset.(select the variance is within 0.05 ) 
leastvar<-PredAct_Personality%>% filter(0.05 >varaiance)%>% filter (varaiance> -0.05)

## select the users which have no variance in the dataset, Top10_Users for accuracy of predicated ratings
novar <- PredAct_Personality%>% filter(varaiance==0)
Top10User_Accur <- novar

##Print Top10 Users who rated more accuracy in the records. 
paste("Top10 User whose predicted and actual ratings were close :",Top10User_Accur$userid)

# Create the function to find mode
getmode <- function(v) {
   uniqv <- unique(v)
   uniqv[which.max(tabulate(match(v, uniqv)))]
}

## get the mode from least variance dataset and see which type of users rated more accurate  
Mode_openness <- getmode(leastvar$openness)
Mode_agreeableness <- getmode(leastvar$agreeableness)
Mode_emotional_stability <- getmode(leastvar$emotional_stability)
Mode_conscientiousness <- getmode(leastvar$conscientiousness)
Mode_extraversion <- getmode(leastvar$extraversion)

##print the mode for each personality of least variance 
paste("In the least variance, Mode_openness:",Mode_openness,"Mode_agreeableness:",Mode_agreeableness,"Mode_conscientiousness:",Mode_conscientiousness,"Mode_emotional_stability:",Mode_emotional_stability,"Mode_extraversion:",Mode_extraversion)

##Around 200 users who rated the film more accuracy (least variance), Set the mode as standard. 
## For the openness trait, the users are with a extreme high level of openness (score 6.5)
## For the agreeableness trait, the Users are moderate (score 4)
## For the conscientiousness, the user are Average (score 5)
## For the emotional_stability, the user are Average (Score 5)
## For the extroversion, the user are moderate (Score 4)
```

  + For the agreeableness trait, the users are moderate (score 4). For the conscientious, the user is average (score 5). For the emotional stability, the users are average. And for the extraversion, the user is moderate (score4). For the extraversion, the users are moderate. 

  + After analyzing the differences between predicted and actual ratings in a few different ways, we wanted to move on to creating our own prediction models to see if we could generate results as close as their were. 
  
## Explanatory Regression Models  

While the original researchers for this study were able to generate a model for predicting user ratings from the personality traits, they came to the conclusion that their "prediction" was unable to explain the differences in User ratings, enjoyment and satisfaction.   
We wanted to attempted to create our own models starting with linear and multiple regression. While we expect to see similar results to theirs (our models will not explain what drivers user ratings with any confidence), we felt it was worth the effort to try.  

#### 1. Linear Regression Models 
```{r LinearRegression, include=FALSE}
# regression models for each of 5 personality traits and summaries
lm_openness <- lm(formula=rating ~ openness, data=RatingPerson)
summary(lm_openness)
#R2 of 6.491 x 10^5

lm_agree <- lm(formula = rating ~ agreeableness, data = RatingPerson)
summary(lm_agree)
#R2 of 0.003928

lm_extra <- lm(formula = rating ~ extraversion, data = RatingPerson)
summary(lm_extra)
#R2 of 0.002633

lm_emotion <- lm(formula = rating ~ emotional_stability, data = RatingPerson)
summary(lm_emotion)
#R2 od 0.001219

lm_conscientious <- lm(formula = rating ~ conscientiousness, data = RatingPerson)
summary(lm_conscientious)
#R2 of 0.000141
```

  + **Our dependent variable for all of the regression models is user ratings. We want to see if there is good indication of any of the Personality Trait Scores being the driver for these ratings. **  
  + **We chose to first look at each of the Personality traits individually. Agreeableness, then extroversion and emotional stability are the traits that are more likely to predict movie ratings. However, none of the R ^2^ values are high enough to suggest a these are good explanatory variables. In fact, the highest R ^2^ value was 0.003928. This was surprisingly low to us.**
  + **Next, we wanted to see if combining any of the personality traits would lead to better explanatory power.**  
  
#### 2. Multiple Regression Models   
```{r MultipleRegression, include=FALSE}
# multiple regression models
lm_agree_extra <- lm(formula = rating ~ agreeableness + extraversion, data = RatingPerson)
summary(lm_agree_extra)
# the inclusion of the extraversion trait made the R^2 value slightly better
#R2 of 0.005956

# add columns for predicted values and residuals to dataframe to create plots
RatingPerson$agree_extra_predict <- predict(lm_agree_extra)
RatingPerson$agree_extra_residuals <- residuals(lm_agree_extra)
RatingPerson$agree_abs_residuals <- abs(RatingPerson$agree_extra_residuals)

lm_agree_extra_emotion <- lm(formula = rating ~ agreeableness + extraversion + emotional_stability, data = RatingPerson)
summary(lm_agree_extra_emotion)
#R2 of 0.006369

RatingPerson$agree_extra_emo_predict <- predict(lm_agree_extra_emotion)
RatingPerson$agree_extra_emo_residuals <- residuals(lm_agree_extra_emotion)
RatingPerson$extra_abs_residuals <- abs(RatingPerson$agree_extra_emo_residuals)

lm_all <- lm(formula = rating ~ agreeableness + extraversion + emotional_stability + conscientiousness + openness, data = RatingPerson)
summary(lm_all)
#R2 of 0.007123

RatingPerson$all_predict <- predict(lm_all)
RatingPerson$all_residuals <- residuals(lm_all)
RatingPerson$all_abs_residuals <- abs(RatingPerson$all_residuals)
```

1. We looked at three separate multiple regression models: 
  + A) Agreeableness + Extraversion:  
  + **R ^2^ of 0.005956**    
  + B) Agreeableness + Extraversion + Emotional Stability:  
  + **R ^2^ of 0.006369**  
  + C) Used all five of the personality traits to explain ratings:  
  + **R ^2^ of 0.007123**  
  + **See the best regression that was produced:**  
  
```{r MultipleRegressionBest, include=TRUE}
summary(lm_all)
```
  
  + **Conclusion: In every model, the R ^2^ values are extremely low, even though the p-values for each of the traits is significant. This tells us that while we can be confident that the inputs are impacting out output, they can explain very little of the variability in User ratings.**    


#### 3. **Look at Scatter Plots of Residuals of the Multiple Regression Models**  
  + **We are including only the plot of the residuals from our regression using all of the personality traits to explain user ratings as that model had the best R ^2^.**  
```{r MRScatter, include=FALSE}
# we plotted our residuals against the Agreeableness trait because it had the (marginally) highest correlation coefficient

regression_agree_extra_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=agree_extra_residuals)) + geom_point() + theme_classic() + ggtitle("Agreeableness & Extraversion Regression Model") + xlab("Agreeableness") + ylab("Residuals")
regression_agree_extra_residuals

regression_abs_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=abs_residuals)) + geom_point() + theme_classic() + ggtitle("Agreeableness & Extraversion Regression Model") + xlab("Agreeableness") + ylab("Absolute Residuals")

regression_agree_extra_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=agree_extra_residuals)) + geom_point() + theme_classic() + ggtitle("Agree, Extra, Emot Regression Model") + xlab("Agreeableness") + ylab("Residuals")
regression_agree_extra_residuals

extra_abs_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=extra_abs_residuals)) + geom_point() + theme_classic() + ggtitle("Absolute Residuals") + xlab("Agreeableness") + ylab("Absolute Residuals")

all_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=all_residuals)) + geom_point() + theme_classic() + ggtitle("Total Regression Model") + xlab("Agreeableness") + ylab("Residuals")
regression_agree_extra_residuals

all_abs_residuals <- ggplot(RatingPerson, aes(x=agreeableness, y=abs_residuals)) + geom_point() + theme_classic() + ggtitle("Absolute Residuals") + xlab("Agreeableness") + ylab("Absolute Residuals")
```

  
  + **Results: As expected, since the R ^2^ values for all are models are so low, plots of the residuals yield no discernible trends.**
  + **Results: All of our R ^2^ values for these models were less than 0.01. When looking at the plots of the residuals, we feel that the values are so random that no reliable information can be drawn from them.**   
  + **Next, we are generating our own prediction models.**   

## Create prediction models to test the accurancy of predicted rate.

#### 1. Generate the test and train groups for our data. 
```{r Prediction1, include=FALSE}
#set the boundaries for predicted rate and add one column is called "boundaries"
Personalitygroup <-PredAct_Personality %>%  select(userid:movie_id) %>% mutate(boundaries =ifelse(predicted_rating<3,yes="2.5-3",
                   no=ifelse( predicted_rating >3 & predicted_rating < 3.5 ,yes="3-3.5", 
                   no=ifelse(predicted_rating > 3.5 & predicted_rating <4,  yes="3.5-4",
                   no= ifelse(predicted_rating<4.5 & predicted_rating <4 , yes="4-4.5", 
                   no = ifelse(predicted_rating>5,yes="5-5.5",no="4.5-5"))))))

#set the sample seed and create a train and test data sets using the PredAct_Personality
set.seed(5680)

# count the number of rows of PredAct_Personality and store in Countrow
Countrow <-NROW(Personalitygroup)  

# take 80% from the rows of AirQu to be the sample set , and round the answer to be integer 
PersonalitySample <- round(0.8*Countrow)

#create the random sample to set up the train and test dataset
RandomSam <- sample(1:Countrow,size=PersonalitySample ,replace=F)

#create the train data set
Personality_train <- Personalitygroup[RandomSam, ]
#create the test data set
Personality_test <- Personalitygroup[-RandomSam, ]

#check the database structure 
str(Personality_train)
str(Personality_test)
```

#### 2. Build a Model using KSVM & visualize the results. 
```{r Prediction2, include=FALSE}
# create the model using ksvm by openness
Model <- ksvm(rating~openness,data=Personality_train,kernel="vanilladot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by agreeableness
Model_a <- ksvm(rating~agreeableness,data=Personality_train,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by emotional stability
Model_e <- ksvm(rating~emotional_stability,data=Personality_train,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by conscientiousness
Model_c <- ksvm(rating~conscientiousness ,data=Personality_train,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by extraversion
Model_ex <- ksvm(rating~extraversion ,data=Personality_train,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by openness and agreeable 
Model2 <- ksvm(rating~openness+agreeableness,data=Personality_train,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
# create the model using ksvm by openness, agreeable, emotional stability, conscientious and extravision 
Model3 <- ksvm(rating~openness+agreeableness+emotional_stability+conscientiousness+extraversion ,
               data= Personality_train ,kernel="rbfdot",kpar="automatic",C=15,cross=1, prob.model=T)
#view SVM of class  
Model
Model_a
Model_e
Model_c
Model_ex
Model2
Model3

# predict each personliaty type by using Personality test
predModel1 <- predict(Model,Personality_test)
PredModela <- predict(Model_a,Personality_test)
PredModele <- predict(Model_e,Personality_test)
PredModelc <- predict(Model_c,Personality_test)
PredModelex <- predict(Model_ex, Personality_test)
predModel2 <- predict(Model2,Personality_test)
predModel3 <- predict(Model3,Personality_test)

#store the variables in CompModel 
compModel1 <- table(predModel1,Personality_test$rating)
compModela <- table(PredModela,Personality_test$rating)
compModele <- table(PredModele,Personality_test$rating)
compModelc <- table(PredModelc,Personality_test$rating)
compModelex <- table(PredModelex,Personality_test$rating)
compModel2 <- table(predModel2,Personality_test$rating)
compModel3 <- table(predModel3,Personality_test$rating)

# check the accuracy of prediction , 0 = NO , 1 = Yes
Model1result <- table(compModel1)
Modelaresult <- table(compModela)
Modeleresult <- table(compModele)
Modelcresult <- table(compModelc)
Modelexresult <- table(compModelex)
Model2result <- table(compModel2)
Model3result <- table(compModel3)

#compute the Root Mean Squared Error
error_pred <- round(rmse(Personality_test$rating,predModel1), digits=4)
error_preda <- round(rmse(Personality_test$rating,PredModela), digits = 4)
error_prede <- round(rmse(Personality_test$rating,PredModele), digits = 4)
error_predc <- round(rmse(Personality_test$rating,PredModelc), digits = 4)
error_predex <- round(rmse(Personality_test$rating,PredModelex), digits = 4)
error_pred2 <- round(rmse(Personality_test$rating,predModel2), digits = 4)
error_pred3 <- round(rmse(Personality_test$rating,predModel3), digits = 4)

#print the result
paste ("As the error value of actual rate for predication(openness) :",error_pred)
paste ("As the error value of actual rate for predication(agreeable) :",error_preda)
paste ("As the error value of actual rate for predication(emotional stablity) :",error_prede)
paste ("As the error value of actual rate for predication(conscientiousness) :",error_predc)
paste ("As the error value of actual rate for predication(extraversion) :",error_predex)
paste("As the error value of actual rate for predication2(openness+agreeable) :",error_pred2)
paste("As the error value of actual rate for predication3 :",error_pred3)

##according to the result of error value of predicating rate for predication, 
##we can know the accuracy of predication should be around 63% (1-0.37)
```
 
  + **We made seven different prediction models using eps-svr regression in the KSVM type model. We calculated the root mean square error of each and saw that all were close to 1.0. See an example below**   

```{r Prediction3, include=TRUE}
paste ("Model 1 error value of actual rate for predication :",error_pred)
```
  
  + **Below we show scatter plots of the variance between Predicted and Actual ratings for each personality type as generated by our KSVM models.**    
  
```{r Prediction4, include=FALSE}
# error between actual predicated rate and Predicted rate
error <- abs(Personality_test$rating - predModel1)
errora<- abs(Personality_test$rating - PredModela)
errore <-abs(Personality_test$rating - PredModele)
errorc <-abs(Personality_test$rating - PredModelc)
errorex <-abs(Personality_test$rating - PredModelex)
error2 <-abs(Personality_test$rating - predModel2)
error3 <-abs(Personality_test$rating - predModel3)

# select the different type of personality and error btw actual rate and predicted rate in ksvmdata
ksvmdata <- data.frame(Personality_test$openness,Personality_test$rating,error)
ksvmdata_a <- data.frame(Personality_test$agreeableness,Personality_test$rating,errora)
ksvmdata_e <- data.frame(Personality_test$emotional_stability,Personality_test$rating,errore)
ksvmdata_c <- data.frame(Personality_test$conscientiousness,Personality_test$rating,errorc)
ksvmdata_ex <- data.frame(Personality_test$extraversion,Personality_test$rating,errorex)
ksvmdata2 <- data.frame(Personality_test$openness,Personality_test$agreeableness,error2)
ksvmdata3 <- data.frame(Personality_test$openness,Personality_test$agreeableness,Personality_test$emotional_stability,Personality_test$conscientiousness,Personality_test$extraversion,error3,Personality_test$rating)

#rename the column name
colnames(ksvmdata) <- c("openness","acutal_rate","variance")
colnames(ksvmdata_a)<- c("agreeable","actual_rate","variance")
colnames(ksvmdata_e)<- c("emotionalstablity","actual_rate","variance")
colnames(ksvmdata_c)<- c("conscientious","actual_rate","variance")
colnames(ksvmdata_ex)<-c("extraversion","actual_rate","variance")
colnames(ksvmdata2) <- c("openness","agreeableness","variance")
colnames(ksvmdata3) <- c("openness","agreeableness","emotional_stablility","conscientious","extraversion","error","actual_rate")

#combine all personality type to be one row and retaining "movie_id" and"userid" , and then store in combine_PT 
combine_PT<- melt(data=ksvmdata3, id.vars = c("actual_rate","error"),measure.vars = c("openness","agreeableness","emotional_stablility","conscientious","extraversion"))
#rename combine_PT columns
colnames(combine_PT) <- c("actual_rate","variance","Personality_type","P_Score")

#plot the scatter diagram for PreModel 
Ksvm_PredModel1 <- ggplot(ksvmdata,aes(x=openness, y= acutal_rate)) +   ## set x = openness , y= actual rate 
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="Openness Variance")
Ksvm_PredModela <- ggplot(ksvmdata_a,aes(x=agreeable, y= actual_rate)) +   ## set x = agreeableness , y= actual rate 
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="Agreeableness Variance")
Ksvm_PredModele <- ggplot(ksvmdata_e,aes(x=emotionalstablity, y= actual_rate)) +   ## set x = emotional stable, y= actual rate 
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="EmotionalStablity Variance")
Ksvm_PredModelc <- ggplot(ksvmdata_c,aes(x=conscientious, y= actual_rate)) +   ## set x = conscientious , y= actual rate 
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="Conscientious Variance")
Ksvm_PredModelex<-ggplot(ksvmdata_ex,aes(x=extraversion, y= actual_rate)) +   ## set x = extraversion , y= actual rate 
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="Extraversion Variance")
Ksvm_PredModel2 <- ggplot(ksvmdata2,aes(x=openness, y= agreeableness)) +   ## set x = openness , y= agreeableness
  geom_point(aes(col=variance,size=variance)) +  ## add points with colors based on error
  labs(title="Openness + Agreeable Variance")
Ksvm_PredModel3 <- ggplot(combine_PT,aes(x=P_Score, y= actual_rate)) +   ## set x = openness , y= agreeableness
  geom_point(aes(col=Personality_type,size=variance)) +    ##  colors based on personality type and size based on error  
  labs(title="The variance btw Acutal and Personality type by Ksvm predicted model",x="PersonalityScore")
```

```{r Prediction5, include=TRUE}
#view the diagram
## combine each type of personality with actual rate in one page (base on the different type of personality)
ggarrange(Ksvm_PredModel1,Ksvm_PredModela,Ksvm_PredModele,Ksvm_PredModelc,Ksvm_PredModelex,ncol = 2, nrow = 3,hjust=15,vjust =5,widths = c(1,1),align="h")
```


  + **For the predicted model of each type of personality, we can see the larger variance of allocation is below average of actual rate. the less variance was spread on the top fo diagram.For example: openness, it indicates users who openness score between 4 and 7 , their rating is closer to actual rate and less difference.**
  + **In the predicted model 2, I set x as openness and y as agreeable, we can see there is a few large point on the diagram. E.g. when the openness scored 5.5 and agreeable scored 5, the is a obvious variance between them. **
  +**In the predicted model 3, I set x as Personality Score and y as actual rate. As you see, Extraversion is the majority variance and spread out each corner on the diagram. Second one is emotional stability and conscientious. **
  
  
## Conclusions
**1. The Personality test that was administered to Users was able to separate Users into distinct groups.**
**2. Assigned Metric and Assigned Condition were unable to explain variability in User Satisfaction.**  
**3. None of the Personality Traits, Assigned Metrics, or Assigned Conditions were able to explain differences in the User' ratings.**  
**4. The researchers were able to predict ratings within around 0.5 points of the actual user ratings.**
**5. Our regression models were unable to explain variability in User ratings.**
**6. Our Predictive models were unable to reliably predict the ratings given by users.**
**7. We wish that we had more time to try other predictive models to see if we could get the provided personality scores to better predict the data.**

#### **8. In conclusion, we do think that having users take a personality test is an interesting approach to being able to better predict user film choices. We do not think that these types of metrics will be able to predict the values of user ratings. We also think that having aditional data, such as user geographical data, would provide more insight for businesses considering using this type of information. 
