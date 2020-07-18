---
title: Working with R session interactively in Python
top: false
cover: false
toc: true
mathjax: false
date: 2020-07-12 12:14:43
password:
summary:
tags:
  - IO
  - Subprocess
categories: Python
---

This post introduces a feasible way to work with *R* session interactively in *Python* with the help of `Subprocess`. Through `PIPE` of `stdin` and `stdout`, data and commands can be passed between the two process.

### Toy Example

Input an integer in *Python* and return that integer + 1 from *R*.

```r
#!/usr/bin/env Rscript
while (TRUE) {
  var = readLines("stdin", 1)
  Sys.sleep(3)
  write(as.integer(var)+1, stdout())
}
```

```python
#!/usr/bin/env python3
from subprocess import Popen, PIPE

rsession = Popen(['Rscript', '/path/to/script.R'], stdin=PIPE, stdout=PIPE, stderr=PIPE)

rsession.stdin.write(b'20\n')
rsession.stdin.flush()
print(rsession.stdout.readline())
```
Explanations line by line:

1. *R*

   * `readLines("stdin", 1)`
     * read the input from python.
     * as R handles scalar and vector of length 1 softly, no other operations are needed.
     * `readline()` does not seem to work this way.
   * `write(as.integer(var)+1, stdout())`
     * write strings in R to `stdout()`.
     * implicitly transform input to *bytes*.

2. *Python*

   * `rsession = Popen(['Rscript', '/path/to/script.R'], stdin=PIPE, stdout=PIPE, stderr=PIPE)`
     * start *R* subprocess.
     * connect to the subprocess by `stdin`, `stdout` and `stderr`(optional) through *PIPE*.
   * `rsession.stdin.write(b'20\n')`
     * only accept *bytes* input.
     * ends with `'\n'` is a MUST.
   * `rsession.stdin.flush()`
     * flush the contents in `stdin`
   * `rsession.stdout.readline()`
     * read the next **ONE** line
     * returned type is *bytes* as well

### Elaborate Example

Pass arguments(parameters) to *R* session in *Python* and retrieve the resulting data frame.


```r
#!/usr/bin/env Rscript
library(tidyverse)

# For windows users, this line might solve the problem with non-English locale
# the stem of the potential problem is that windows does not seem to support locales like 'ja_JP.UTF-8'
# Sys.setlocale(locale = "english")


# Toy dataframe
df_raw <- tibble(
  x = c('いろは','やちよ','鶴乃'),
  y = 1:3,
  z = c('うい', 'みふゆ', '万々歳')
)

# function to output the dataframe to stdout
write_df <- function(df) {
  # Begin Of DataFrame, works as an indicator for following dataframe lines
  write("BODF", stdout())
  
  df %>%
    format_tsv() %>%
    enc2utf8() %>%
    write(., stdout())
}

# act like interactively
while (TRUE) {
  arg_input = readLines("stdin", 1)
  
  # filter rows
  df_temp <- df_raw %>%
    filter(x==arg_input)
  
  write_df(df_temp)
}
```


```python
#!/usr/bin/env python3
import pandas as pd

from subprocess import Popen, PIPE
from io import BytesIO

def read_df(word, sess):
  '''
  Arguments:
    word: can be arbitrary parameters passed to R session through stdin pipe
    sess: R session running as a subprocess
  '''
  word += '\n' # add end of line 
  sess.stdin.write(word.encode('utf-8')) # to bytes
  sess.stdin.flush()
  
  # skip irrelevant lines
  # b'BODF\r\n' is defined in the R script
  line = 'initializer'
  while line!=b'BODF\n':
    line = sess.stdout.readline()
    
  # write all lines of dataframe to a buffer
  # data frame in lines of stdout from R session shall end with b'\r\n'
  buffer = BytesIO()
  while line!=b'\n':
    line = sess.stdout.readline()
    buffer.write(line)
    
  # reset the pointer of the buffer and read
  buffer.seek(0)
  df = pd.read_csv(buffer, sep='\t', encoding='utf-8')
  return df


if __name__=='__main__':
  rsession = Popen(['Rscript', '/path/to/script.R'], stdin=PIPE, stdout=PIPE, stderr=PIPE)
  try:
    print(read_df('いろは', rsession))
    print(read_df('やちよ', rsession))
  except Exception as e:
    print(e)
  finally:
    rsession.kill()
```

1. *R*
   * `format_tsv()`
     * transform the data frame to a string
     * other similar functions can be found in [official documents](https://www.rdocumentation.org/packages/readr/versions/1.3.1/topics/format_delim)
   * `enc2utf8()`
     * change encoding of the string to `utf-8`
     * this line might be necessary under certain conditions

2. *Python*

   * `buffer = BytesIO()`
     * create a file-like object to save the *bytes* input from `stdout`
   * `buffer.write(line)`
     * save `stdout` line by line to buffer
   * `buffer.seek(0)`
     * reset the pointer to the beginning of buffer
     * otherwise nothing can be read as the pointer is unidirectional
   * `df = pd.read_csv(buffer, sep='\t', encoding='utf-8')`
     * `sep='\t'` is corresponding to `format_tsv()` in *R*
     * `encoding='utf-8'` is necessary.


### Monitor `stdout` from R by `Thread`

To avoid the *deadlock* of the main thread, using another thread to listen to `stdout` is a good solution

*R* code does not change from previous section.


```python
#!/usr/bin/env python3
import pandas as pd

from subprocess import Popen, PIPE, call
from threading import Thread
from queue import Queue
from io import BytesIO

def read_stdout(sess):
  flag_df = False
  for line in iter(sess.stdout.readline, b''):
    if flag_df:
      if line==b'\n':
        buffer.seek(0)
        df = pd.read_csv(buffer, sep='\t', encoding='utf-8')
        queue.put(df, False)
        flag_df = False
      else:
        buffer.write(line)
    elif line==b'BODF\n':
      flag_df = True
      buffer = BytesIO()
      
def read_df(word, sess):
  word += '\n' # add end of line 
  sess.stdin.write(word.encode('utf-8')) # to bytes
  sess.stdin.flush()
  
  df = queue.get(timeout=10)
  return df

if __name__=='__main__':
  rsession = Popen(['Rscript', '/path/to/script.R'], stdin=PIPE, stdout=PIPE, stderr=PIPE)

  # global queue
  queue = Queue()

  # start thread to listen to rsession.stdout
  t = Thread(target=read_stdout, args=(rsession, ))
  t.start()
  try:
    print(read_df('いろは', rsession))
    print(read_df('やちよ', rsession))
  except Exception as e:
    print(e)
  finally:
    rsession.terminate()
    # On windows try this.
    # call(['taskkill', '/F', '/T', '/PID', str(rsession.pid)])

  # can only join the thread when rsession is terminated
  # as stdout.readline() only returns b'' when the process is terminated.
  t.join()
```

This enables more flexible control over the `stdout` stream and abstract the functionalities of each part better. Though more lines to handle exceptions are needed, the basic flow might be a good practice.


### Class version of the previous approach using `Thread`

```python
#!/usr/bin/env python3
from subprocess import Popen, PIPE, call
from threading import Thread
from queue import Queue, Empty
from io import BytesIO
from functools import wraps

# if you are using Jupyter to test
from IPython.display import display

import pandas as pd

def threaded(func):
  @wraps(func)
  def wrapper(*args, **kwargs):
    thread = Thread(target=func, args=args, kwargs=kwargs)
    thread.start()
    return thread
  return wrapper

class Rsession:
  def __init__(self, path):
    self.sess = Popen(['Rscript', path], stdin=PIPE, stdout=PIPE, stderr=PIPE)
    self.queue = Queue()
    
    # start thread
    self.thread = self.read_stdout()
    
  @threaded
  def read_stdout(self):
    flag_df = False
    for line in iter(self.sess.stdout.readline, b''):
      if flag_df:
        if line==b'\n':
          buffer.seek(0)
          df = pd.read_csv(buffer, sep='\t', encoding='utf-8')
          self.queue.put(df, False)
          flag_df = False
        else:
          buffer.write(line)
      elif line==b'BODF\n':
        flag_df = True
        buffer = BytesIO()
      
  def read_df(self, word):
    word += '\n' # add end of line 
    self.sess.stdin.write(word.encode('utf-8')) # to bytes
    self.sess.stdin.flush()

    df = self.queue.get(timeout=10)
    return df
  
  def __enter__(self):
    return self
  
  def __exit__(self, exc_type, exc_val, exc_tb):
    self.sess.terminate()
    # call(['taskkill', '/F', '/T', '/PID', str(self.sess.pid)])
    self.thread.join()

if __name__=='__main__':
  with Rsession('/path/to/script.R') as sess:
    display(sess.read_df('いろは'), sess.read_df('やちよ'))
```



### Tips

* Be careful of **deadlock**

The following lines will cause a deadlock if the execution of the process is not finished, e.g. using `while(TRUE)` to interact with *R* session. In that case, *Python* is waiting for `stdout` and *R* is waiting for `stdin`, and neither is really moving a single step forward.

```python
for line in sess.stdout.readlines():
  pass
```

In real world applications, `callbacks` are definitely necessary to avoid those situations. A practical strategy will be always reading `stdout` line by line and working with `callbacks` from *R* session to take the proper actions.

* Windows

There are many subtleties in Windows. 

  1. But the *encoding/decoding* is the most complicated one among them. Like in my case, `stdin` and `stdout` are handled properly in `utf-8`, whereas the `stderr` is `ShiftJIS` inheriting settings from powershell. If only Windows were to support locales like `ja_JP.UTF-8`...

  2. `rsession.terminate()` does not work... Try `subprocess.call(['taskkill', '/F', '/T', '/PID', str(rsession.pid)])` instead. Even after killing the subprocess, one should run either `rsession.terminate()`/`rsession.kill()` or `rsession.wait()` to let the process instance know it is terminated. Otherwise, those methods of checking the status of the process would not work properly.

  3. `End Of Line` from `stdout` on Windows is *'\r\n'* rather than *'\n'*.

Developing on a unix system would solve them naturally...