---
layout:     post
title:      Implementing a simple file watcher using Bash/Crontab
date:       2020-07-10 19:22
summary:    This is a simple exploration of cron, bash and how to create a file watcher.
categories: software
---
> Watch for modified files in specific directories at regular interval

Ever wondered how to subscribe to new files? We will explore a very basic functional way to solve this issue in bash in this blog post. The situation is as follows:some processes write files with a consistent prefix (e.g: revenue_tableYYYYMMdd.csv) and we want to catch the new written files and do something with them, for instance copy them to another server.  We assume we only have very basic software infrastructure, a server running bash. We only have to write 2 files, the bash script and the config file. Then we will run the bash script in a cron job to run everyday. The structure will look like the following:
```
subscriber
|    schedule_transfer.sh
|    subscribed_files_info.csv
```
The bash script will parse the csv and verify whether the mentioned files have been modified/created/updated. If so, we will copy them to another server.

#### Config file
Let's start by writing the config file `subscribed_files_info.csv`. This file is a simple csv with no header -- it's simpler not to write header -- with 2 columns. The first column is the path and the second column is the prefix of the tables inside of the specific directory.

```python
df.head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>path</th>
      <th>file_prefix</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>/home/jackson/Documents/test_dir</td>
      <td>testtable</td>
    </tr>
    <tr>
      <th>1</th>
      <td>/home/jackson/Documents</td>
      <td>tablejoke</td>
    </tr>
  </tbody>
</table>
</div>



### Bash script 
Great, now the goal is to subscribe to these two paths. In other words, we want to fetch all new files with the path `/home/jackson/Documents/test_dir/testtable*` and `/home/jackson/Documents/tablejoke*`. 

Let's say that we are only interested in the files updated in the last day. That way, if we schedule our script to run everyday at the same hour, we should capture most new files. Obviously, there are many cases which will cause potential issues, but here we assume that if the server is offile and the job does not run and a file modified in the last day is not captures, some operators will fix the situation.


```sh
%%sh
#! /usr/bin/env bash

while IFS=, read -r path prefix
do
    echo "I got:$path/$prefix"
    files=$(find $path -iname "$prefix*" -mtime -1 -type f)
    echo ${files}
done < subscribed_files_info.csv

```

    I got:/home/jackson/Documents/test_dir/testtable
    /home/jackson/Documents/test_dir/testtable220.csv /home/jackson/Documents/test_dir/testtable2019.csv
    I got:/home/jackson/Documents/tablejoke
    /home/jackson/Documents/tablejoke202006.csv /home/jackson/Documents/tablejoke202005.csv


This is the most important piece of code here. It loops over all rows of the csv and extracts the 2 variables. Then, we use the `find` command not gather all the files beginning with our prefix. Notice the `-mtime -1` argument which filters the files older than 1 day. The `-type f` argument filters the search to only files (no directories).  
Next, we will swap the `echo ${files}` to an actual function which scp the files to another server.

```python
%%handle
%%sh

function process_files {
    for filename in "${@}"
    do
        echo "Processing file ${filename}"
        # scp ${filename} jackson@10.0.4.100:/ingestion/import_zone/
    done
}
;
```

Excellent, now that we have a function to handle our programming logic. We simply have to swap echo for process files:

```python
%%handle
%%sh
#! /usr/bin/env bash
while IFS=, read -r path prefix
do
    echo "I got:${path}/${prefix}"
    files=$(find ${path} -iname "${prefix}*" -mtime -1 -type f)
    process_files ${files}
done < subscribed_files_info.csv
```

To automate our process, we need the script to run everyday. An efficient way of doing so is setting up a cron job. Cron is a system daemon used to execute desired tasks (in the background) at designated times. To create a new cron job for our user, hit:

```sh
%%sh
crontab -e
```

I suggest you use [this website](https://crontab.guru/) to help you out with the cron schedule.  

For instance, everyday at 16:35 is `33 16 * * *`. In this case, we only have to add the command 
`33 16 * * * /bin/bash -c "<path_of_your_script>` at the very end of the crontab file (from the editor opened by crontab -e).
