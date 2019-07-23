# RTutorSAGI: Analyse Submissions to Grade and Improve Problem Sets
 
**Author: Sebastian Kranz, Ulm University** 

This is a companion package to [RTutor](https://github.com/skranz/RTutor/). RTutor allows to create interactive R problem sets that students can solve at home or in the cloud (e.g. via https://rstudio.cloud/). 

Students can create submission files of their solutions (either by calling `make.submission` in the RStudio version, or by pressing the corresponding button in the shiny web version). 

This package helps teachers to handle the submissions, in particular for two tasks:

1. Grading: Combine the submissions of several students and several problem sets and automatically create csv files that show the total points achieved.

2. Problem Set Improvement: By default a submission file contains a log that contains the code from all attempts of a user to solve the tasks. This helps to understand where students systematically got stuck when solving the problem set and how they misunderstood particular tasks. One can use the insights to improve the problem set by adapting the hints or by changing problematic tasks. RTutorSAGI has functions that facilitate this process.

First some important notes.

## DISCLAIMER: NO WARRANTY
I wrote this R package for my own usage. While everybody is free to use it, I provide no warranty whatsoever.
While best to my knowledge the software works fine, I cannot rule out with 100% certainty that there may be a bug in the code such that points are incorrectly counted or incorrectly added up in some circumstances.

## Security

For security reasons the functions described below don't evaluate the code that students have submitted. For grading they rely on the statistics created on students' computers when `make.submission` was called. Nevertheless, the submission files from all students are loaded. While I don't know how (and did not find any specific warnings on Google) this potentially may open some attack vector. To be on the safe side, you can analyse the submissions inside an [RStudio docker container](https://hub.docker.com/r/rocker/rstudio/) with restricted permissions.

## Installation

My packages are not hosted on CRAN (while CRAN is great, it takes a lot of time to maintain several packages there). You can install `RTutorSAGI` with all dependencies from my own Github based R repository as following:

```r
install.packages("RTutorSAGI",repos = c("https://skranz-repo.github.io/drat/",getOption("repos")))
```
Alternatively, you can directly install from Github using `devtools::install_github("skranz/RTutorSAGI")`.


## Loading submission

We first have to load all relevant submission. A submission file that a student has created with `make.submission` has the file type `.sub`.

You can load a single submission file with the function `load.sub` (which is basically a wrapper around base R `load`):

```r
sub = load.sub("Max_Mustermann__Intro.sub")
```
The variable `sub` is an environment with several elements For example
```r
sub$rmd.code
```
contains the content of the student's solution Rmd file.

We typically don't want to analyse single submissions but bulk load all submission. For this purpose, you should put all submitted submission files into some directory and then use `load.subs` with a correct `stud.name.fun` as explained below. The directory can have subdirectories, e.g. for different problem sets. 

### Example using Moodle

At my university, we use [Moodle](https://moodle.org/) as course management system. My students solve the problem sets at home and then shall upload their submission files on Moodle. Moodle then allows me to bulk download for each problem set a big ZIP file with all submission as small zip files. 

The following code uses a simple `RTutorSAGI` convenience function to unpack all ZIPs from Moodle, which I stored in subdirectory `moodle_zip`

```r
# Make sure you are in the right working directory
getwd()
# Possible adapt via setwd()
unpack.moodle.sub.zips(zip.dir = "moodle_zip", sub.dir="sub")
```
The subdirectory `sub` then contains all submission files.

We now load all submissions into R:
```r
sub.li = load.subs(sub.dir = "sub", stud.name.fun=moodle.stud.name.fun)
```
When loading the submissions, we also want to assign the correct student name for each submission. While students shall specify a user name in the Rmd file that contains their solution, these sepcified user names may not be completely reliable. Instead, I want to use the student names from Moodle. Moodle encodes the student's name in the file name of the submission. Currently (Summer 2019) the format has the structure:

`Upload_MoodleTaskName----StudentName--OriginalFileName.sub`

The argument `stud.name.function` of `load.subs` is a function that takes the file name of the submission file and its loaded content `sub` as arguments and must return the student's name. For example, to extract the sudent name from the Moodle file format above, we could use the function:
```r
my.stud.name.fun = function(file, sub, ...) {
  stud.name = stringtools::str.between(basename(file),"----","--")
  stud.name
}
```
The function `moodle.stud.name.fun` does a little bit more in order to deal with UTF-8 encoding problems.

If no `stud.name.fun` is provided, we use by default the user.name specified in the problem set. But as mentioned, that might be more unreliable.

Having loaded a list of all submissions `sub.li`, we can then procceed with grading or analysing the logs.

### Alternatives to Moodle

Of course, there are many other ways how you can create a directory with all submission files. There is absolutely no need to use Moodle. You can use any other course management system were students can upload files and you can download them. You could also write and host your own shiny app where students can make their submissions.

## Grading

If `sub.li` contains a loaded list of submissions, we can summarize total points by calling:

```r
grade.subs(sub.li=sub.li, grade.dir="grades")
```

It then generates in the `grade.dir` directory a csv file with total points of every student and another file with point per problem set. The tables also contain information about how many hints were asked. Hints have no impact on the points, however.

If submissions also contain log files, there is also a csv file that estimates work time on the problem sets.

What you do with the total points is up to you. You may set a minimal point requirement to be allowed to participate in the exam. Or you may make the points a small part of the total grade.

From my experience, even some small relevance for final grades provides nice incentives for students to work on the problem sets. I would not recommend to make the points from RTutor problem sets count much more than 10% of the total grade, however. I don't see an effective way how you can rule out cheating from students who copy the solutions from other students. The main reason for a grade optimizing student to work himself on the problem sets should be that it provides good preparation for the final exam.


Loading submissions and grading creates and appends to a log file `grading_log.txt` in your working directory. The file can give information about irregularities. Here is an example:
```
grade_log.txt

The following submissions have not the extension .sub and will be ignored:

 Upload_PS_Intro----Max_Mustermann--Max_Mustermann_Intro.Rmd
```

In my courses, it happens sometime that student upload the wrong file, like their Rmd file instead of the submission file created with `make.submission`. This is noted in the log. If you are lenient, you could then write the student an email and ask the student to upload the correct file and then repeat the whole process.

## Analyse logs in order to improve problem sets

While I put quite some effort to create helpful automatic hints and failure messages, they may not always be enough. Sometimes students get stuck badly. While it is good if students have to think about a problem and solving requires some effort, sometimes a task may have just be too complicated. Alternatively, RTutor may expect a particular solution while a solution of the student that is seemingly also correct is rejected without approbriate feedback for the student why.

When designing a problem set, it is hard to predict all the ways how students can get stuck and it is also quite tedious to cover all possible cases in advance. Better to have an iterative process, where we improve a problem set after seeing in detail how student got stuck. This information is stored in the logs that are by default part of students' submission files.

To convert the information into a more convenient format call:

```r
sub.li = load.subs(sub.dir="sub", rps.dir="org_ps")
write.chunk.logs(sub.li, logs.dir = "chunk_logs")
```
Make sure that you store in the directory specified by `rps.dir` all rps files of the original versions of your problem sets.

This call creates and fills the directory specified in `logs.dir` with subdirectories for each problem set. Each subdirectory contains an .R file for each chunk of the problem set that requires the user to enter code.

An example file name is

`3_e (e 88 u 13 h 11).R`

This file contains a log of all failed solution attempts of chunk `3 e` in the problem set. The information in brackets of the file name tells us that `e=88` times the check failed. There were `u=13` users who at least once entered some wrong code. Also in total `h=11` times a user asked for a hint. So by scanning this info in the file names, you get some first idea of which chunks of a problem set might be problematic.

If you want to analyse these error and hint statistics in a nice R data frame, you can run
```r
res = analyse.subs(sub.li,rps.dir="org_ps")
sum.df = res$sum.df
sum.df
```

Consider the following following task from Excercise 3 e) of my Intro to R problem set.

> e) Let z be a variable that shows the first 100 square numbers, i.e. 1,4,9,... Then show z.

Here is the log of one user from `3_e (e 88 u 13 h 11).R`:

```r
# NEW USER **********************************


z=(seq(1:10))^2
z


# *** 26 secs later...  asked for hint.

# *** 58 secs later... 

z=1:10*1:10
z


# *** 65 secs later... 

z=(1:10)*(1:10)
z


# *** 23 secs later...  asked for hint.

# *** 32 secs later... 

z=(1:10)^2
z


# *** 22 secs later... solved succesfully!
```

Looking at the whole log file, I found that several students assigned `z=(1:10)^2` instead of `z=(1:100)^2`.

The sample solution was `z = 1:100 * 1:100`. The automatic hint for such a formula would have looked like this:
```
You have to assign a correct formula to the variable 'z'. Here is a scrambled version of my solution with some characters being hidden by ?:

 z = 1??00 * 1:?0?
```

I originally was of the opinion that the automatic hint gave away too much information here and therefore specified a custom hint. That just showed the message:
```
There is a simple one-line formula to compute the 100 first square numbers. Just combine what you have learned in exercise 2 f) and in exercise 3 b).")
```

Based on the log files, I have updatet to an adaptive hint that provides more detailed information for this common mistake. In the solution file the chunk now has the following code:
```r
z = 1:100 * 1:100
#< hint
if (true(identical(z,1:10*1:10))) {
  cat("Huh, you made a common mistake. Read the task precisely. You shall assign to z the first 100 square numbers, not only all square numbers that are less or equal to 100")
} else if (true(length(z)!=100)) {
  cat("Your variable z must have 100 elements, but in your solution z has", length(z),"elements.")  
} else {
  cat("There is a simple one-line formula to compute the first 100 square numbers. Just combine what you have learned in exercise 2 f) and in exercise 3 b).
")
}
#>
z
```

The code in the hint block will be evaluated in an environment in which all variables defined in earlier solved chunks are known. Also the student's current code that chunk has been run in a parent environment. 

The function `true` is a robust version of `isTRUE` that never throws an error, but simply returns `FALSE` if the expression cannot be evaluated. This is useful, because 
whether `z` exists in the hint environment depends on whether the user has defined it in her solution for the chunk or not. A normal call to `isTRUE` would throw an error if `z` does not exist.

Note that you and your students need at least version 2019.07.22 (yes my version numbers are just dates) of RTutor for those adaptive hints to work.

Side remark: The new RTutor version also has improved behavior of the automatic tests. Working through the logs I found some systematic cases were RTutor failed to accept a seemingly correct solution without helpful message to the students why.
