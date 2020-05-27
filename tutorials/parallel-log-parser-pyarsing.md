# Parallel log parser with PyParsing and multiprocessing

## Table of contents

- Introduction
- Creating a virtual environment
- Log file parser with PyParsing
- Parallel log file parser with `multiprocessing`
- Comparing runtime with `time` command
- Load the log file in chunk
- Conclusion
- Further reading

## Introduction

In the [another tutorial](https://studiawan.github.io/tutorials/log-parser-pyparsing), I have described a simple log parser with PyParsing. The code only uses one processor at one time and takes time when processing a large log file. It works fine with a small log file. However, in reality, we often get a large log file. Therefore, it needs more speed to parse log entries. Fortunately, Python provides a native package called `multiprocessing`, which supports parallel processing. We will use this package to make a parallel log parser. This tutorial includes three full codes and you can jump into the codes to try and get the results immediately:
- Parsing with serial processing [(code)](https://gist.github.com/studiawan/33845032e0a15e5d2a100626033f37ef)
- Parsing with parallel processing [(code)](https://gist.github.com/studiawan/60910b0a6e6ac926a93044b4ff48e533)
- Parsing with parallel processing and read the file in chunk [(code)](https://gist.github.com/studiawan/c9995b5620f57ed9617fea6fd67ef16a)

## Creating a virtual environment

It is recommended that we use a virtual environment to run a Python project. The reason is that we can contain the packages needed that sometimes have a different version with other projects. With a virtual environment, we can install various packages without broke other projects' packages. We use Anaconda for creating this environment:

`conda create --name parallel-log-parser python=3.6`

We can specify the Python version to be used in this project. If we do not specify the version, it will use the default version from the operating system. Next, we need to activate the newly created environment:

`conda activate parallel-log-parser`

It should show the environment name `(parallel-log-parser)` at the beginning of the terminal or your command prompt. I recommend this tutorial [Python Virtual Environments: A Primer](https://realpython.com/python-virtual-environments-a-primer/) to understand more about the virtual environment. After that, install the `pyparsing` package:

`pip install pyparsing`

## Log file parser with PyParsing

We have created a log file parser with PyParsing in another [tutorial](https://studiawan.github.io/tutorials/log-parser-pyparsing). In short, the parser process will extract each field in a log entry, such as timestamp, application name, and log message. However, the code in that tutorial is running in serial mode and only using one processor at a time. We can modify the code to run in parallel by using the `multiprocessing` package. The full code of serial execution is available in my [GitHub Gist](https://gist.github.com/studiawan/33845032e0a15e5d2a100626033f37ef). While the code for parallel one is also available [here](https://gist.github.com/studiawan/60910b0a6e6ac926a93044b4ff48e533).

## Parallel log file parser with `multiprocessing`

To use the package, we need to import it at the beginning of the code: `import multiprocessing`. We modify the existing log parser code by adding the method `__call__`:

```
    def __call__(self, log_line):
        parsed_log = self.__get_fields(log_line)
        
        return parsed_log
```

`__call__` is a built-in method and it will be executed when we call the `parse_authlog` containing `map` function from `multiprocessing`. We then modify the original `parse_authlog` by using `multiprocessing` so that we can parse all lines in different processor core:

```
    def parse_authlog(self):
        # read log file
        try:
            with open(self.log_file, 'r') as f:
                log_lines = f.readlines()
        
        except FileNotFoundError:
            print('File not found.')
            sys.exit(1)
        
        # run parser with multiprocessing
        total_cpu = multiprocessing.cpu_count()
        pool = multiprocessing.Pool(processes=total_cpu)
        parsed_logs = pool.map(self, log_lines)
        pool.close()
        pool.join()
        
        self.__save_csv(parsed_logs)
```

There are several key objects and methods used from `multiprocessing`:
- `cpu_count()`: It returns the count of all available processors from the system. We can set the number of processors for `multiprocessing`. For example, we only want half of the total processors. Therefore, we can define `total_cpu = multiprocessing.cpu_count() / 2`. Or we can just set `total_cpu = 3`.
- `Pool`: An object that supports data parallelism and controls the jobs for each processor. The argument is `processes` which means the number of worker processes distributed to the designated processor.
- `map`: A parallel implementation of the regular `map` function. It applies a function to every item in the provided list. In this case, each item is a log line and the function is the parser using PyParsing as defined in the method `__call__` (it executes `self.__get_fields` method).
- `close`: This method restricts more jobs from being submitted to the pool. 
- `join`: This method waits for all processes to exit.

In the above code, `map` has two arguments: `self`, and `log_lines`. When `map` is executed, it will call `__call__` because the first argument is `self`. In another approach, we can call a function and replace the `self` argument. `log_lines` is all lines from the input log file. Once we call `pool.map`, it will distribute the data through all available processors. The results will be saved in a list named `parsed_logs`. 

Next, we can save the parsing results to a CSV file by adding this method `__save_csv`:

```
    def __save_csv(self, parsed_logs):
        # open csv file
        f = open(self.log_file + '.csv', 'wt')
        writer = csv.writer(f)
        writer.writerow(['timestamp', 'hostname', 'application', 'message'])
        
        for result in parsed_logs:
            writer.writerow([result['timestamp'], result['hostname'], result['application'], result['message']])
        
        f.close()
```

Saving to the CSV file is pretty straight-forward. We loop through each element in parsing results `parsed_logs` and then write it to the file with `writer.writerow()`. Note that each object to be written to a CSV file should be formatted as a list. We do not install any additional package because `csv` is a native package from Python installation.

An additional explanation of concurrency and parallelism in Python is available in a tutorial from Real Python: [Speed Up Your Python Program With Concurrency](https://realpython.com/python-concurrency/). 

## Comparing runtime with `time`

One simple way to measure execution time is to use `time` command. We will compare the serial and parallel code to see the effectiveness of `multiprocessing`. We can easily call this from the terminal and executing the script at the same time:

```
$ time -p python parallel-log-parser.py auth.log
real 8.14
user 29.05
sys 0.21
```

```
$ time -p python log-parser-pyparsing.py auth.log
real 14.77
user 14.73
sys 0.03
```

Where option `-p` will show execution time in the second unit. We use `real` because it shows real time used by the process. Using parallel code, we only need 8.14 seconds, while using the serial code requires 14.77 seconds. Note that `user` time in parallel is 29.05 because I use all four processors available on my laptop. 

## Load the log file in chunk

Sometimes we get a large log file to analyze. Using `readlines()` will load all log entries into memory. In the case of a large file, it is not efficient. However, we can load a large log file in a chunk, so we do not save all log entries in the memory. In this section, we will parse the log file with `multiprocessing` and read the log file in the chunk. We add method `__grouper` to read the file in chunk:

```
    def __grouper(self, n, iterable, padvalue=None):
        return zip_longest(*[iter(iterable)]*n, fillvalue=padvalue)
```

We need to add an import statement `from itertools import zip_longest` to use `zip_longest` which provides ability to read in chunk size of `n` from an `iterable`. In our case, the iterable is a file handler. In the last chunk, if the number is less than `n`, it will be padded with `None` value. The method `parse_authlog` is now modified into `parse_authlog_chunk`:

```
    def parse_authlog_chunk(self):
        # open log file
        f_log = open(self.log_file, 'r')
        chunk_size = 1000
        
        # open csv file
        f_csv = open(self.log_file + '.csv', 'wt')
        writer = csv.writer(f_csv)
        writer.writerow(['timestamp', 'hostname', 'application', 'message'])

        # create pool for multiprocessing
        cpu_total = multiprocessing.cpu_count()
        pool = multiprocessing.Pool(cpu_total)

        # process in chunk
        for chunk in self.__grouper(chunk_size, f_log):            
            results = pool.map(self, chunk)            
            
            # write to csv file
            for result in results:
                if result is not None:
                    writer.writerow([result['timestamp'], result['hostname'], result['application'], result['message']])
           
        pool.close()
        pool.join()
        f_log.close()
        f_csv.close()

```

First, we open the log file and the CSV file to save the parsing results. The pool is then initiated and we read the log file in a chunk with `for chunk in self.__grouper():`. For each chunk, it will be processed for parsing, as shown in `results = pool.map(self, chunk)`. The parsing results for each line are then saved into the CSV file with `writer.writerow()`. Finally, we close all file handlers and the pool instance. The full code for parsing log file in a chunk is [available here](https://gist.github.com/studiawan/c9995b5620f57ed9617fea6fd67ef16a).

## Conclusion

In this tutorial, we discuss building a simple parallel program to parse a log file. The runtime analysis with `time` clearly shows that using `multiprocessing` can speed up the parsing compared to the serial code. By using `multiprocessing`, we can write a simple but powerful code that runs in a parallel. We only build a parser for auth.log file and you can extend the grammar of log parser to apply the parallel parser to you own log file. 

## Further reading

- multiprocessing — Process-based parallelism [https://docs.python.org/3.6/library/multiprocessing.html](https://docs.python.org/3.6/library/multiprocessing.html)
- PyParsing documentation [https://pyparsing-docs.readthedocs.io/en/latest/](https://pyparsing-docs.readthedocs.io/en/latest/)
- [Parallel Processing With multiprocessing: Overview](https://realpython.com/lessons/parallel-processing-multiprocessing-overview/)
