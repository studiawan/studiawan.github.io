# Writing log file parser with Python and PyParsing

## Table of contents
* Log file and its structure
* Creating a virtual environment
* Installing PyParsing
* Building a parser for auth.log
* Saving parser results to a CSV file
* Conclusion
* Further reading

## Log file and its structure

A log file records various events from users and the operating system. A system administrator needs to inspect log files for suspicious activities or regular monitoring. The first step of this analysis is to parse log files and break them into fields or entities. Below is an example of a portion of an authentication log file from a Linux operating system. This sample file can be downloaded from [SecRepo](http://www.secrepo.com/auth.log/auth.log.gz) (a public repository for security-related files). 

```
Nov 30 06:39:00 ip-172-31-27-153 CRON[21882]: pam_unix(cron:session): session closed for user root
Nov 30 06:47:01 ip-172-31-27-153 CRON[22087]: pam_unix(cron:session): session opened for user root by (uid=0)
Nov 30 06:47:03 ip-172-31-27-153 CRON[22087]: pam_unix(cron:session): session closed for user root
```

As shown above, the typical fields in a log file are timestamp, host name, service name and its process identifier, and the main message. The parsing procedure aims to split each field in each log entry. Usually, we use a regular expression (regex) to parse the log file. However, regex sometimes hard to read and maintain. Therefore, we will use more human-readable parser using PyParsing in this tutorial.

## Creating a virtual environment

Before writing the codes, it is better to create a Python virtual environment, so that our libraries in this project do not pollute other projects. I recommend Anaconda to manage our environment:

`conda create --name log-parser python=3.6`

Wait for a moment because Anaconda is downloading the necessary basic libraries. As the command implied, we create an environment named `log-parser` with Python version 3.6. Next, we need to activate this environment:

`conda activate log-parser`

If the environment is activated successfully at the beginning of the terminal, we can see `(log-parser)`. For a further tutorial about virtual environment, you can read [An Effective Python Environment: Making Yourself at Home](https://realpython.com/effective-python-environment/) and [Python Virtual Environments: A Primer](https://realpython.com/python-virtual-environments-a-primer/) from Real Python website. Make sure that we use this environment when we execute the code that we will discuss below.

## Installing PyParsing

[PyParsing](https://github.com/pyparsing/pyparsing) is a human-readable parser and its main advantages are easy to write and read. We can install PyParsing using `pip` in the activated environment `log-parser`: 

`pip install pyparsing`

To test whether or not PyParsing is installed correctly, we can import it in a Python prompt:

`import pyparsing`

If the screen shows no error messages, it means PyParsing is successfully installed and we can start writing the log parser.

## Building a parser for auth.log

To build a parser, we need two functions. The first one is for grammar or structure definition and the second one is to get the parsing results based on the defined grammar. In this tutorial, we make a parser for auth.log that usually found in a typical Linux operating system. If you want to run the full source code, it is available on my [GitHub Gist](https://gist.github.com/studiawan/33845032e0a15e5d2a100626033f37ef). The definition of class for auth.log parser `AuthLogParser` is shown below:

```
import sys
import csv
from pyparsing import Word, alphas, Suppress, Combine, string, nums, Optional, Regex


class AuthLogParser(object):
	def __init__(self, log_file):
		self.log_file = log_file
		self.authlog_grammar = self.__get_authlog_grammar()
```

First, we import the necessary packages, specifically `sys` and several functions from the PyParsing module. We will save the parsing results into a CSV file, so we also import package `csv`. In the `__init__` method, we define the log file name and call `__get_authlog_grammar()` so that we do not call this method each time we parse a line from the log file. In the next step, we define the grammar of an auth.log file:

```
	@staticmethod
	def __get_authlog_grammar():
		ints = Word(nums)

		# timestamp
		month = Word(string.ascii_uppercase, string.ascii_lowercase, exact=3)
		day = ints
		hour = Combine(ints + ":" + ints + ":" + ints)
		timestamp = month + day + hour

		# hostname, service name, message
		hostname_or_ip = Word(alphas + nums + "_" + "-" + ".")
		appname = Word(alphas + "/" + "-" + "_" + ".") + Optional(Suppress("[") + ints + Suppress("]")) + Suppress(":")
		message = Regex(".*")

		# auth log grammar
		authlog_grammar = timestamp.setResultsName('timestamp') + hostname_or_ip.setResultsName('hostname') + \
						  appname.setResultsName('application') + message.setResultsName('message')

		return authlog_grammar
```

There are several functions we used in the grammar:
- Word: one or more characters.
- Combine: joins all matched tokens into a single string.
- Optional: optional element for a match.
- Suppress: removes matched tokens.
- Regex: a regular expression to be matched at the current parse position.

`alphas` denotes letter and `nums` represents digits. PyParsing parses each word separated by space character by default. `Word(string.ascii_uppercase, string.ascii_lowercase, exact=3)` means that a word can contain upper and lower case with the exact length of three characters. While `Regex(".*")` means all words until the end of the line.

The log grammar is a concatenation of each definition with its correct order: timestamp, host name, application or service name, and the main message of a log entry. `setResultsName` aims to give an identifier for each field that we parse. This will make it easy for us to retrieve each field by writing its identifier, as shown in the following method, `__get_fields`. 

After that, we can write another method to parse each line based on the defined grammar. We use `parseString` to parse a particular line. Subsequently, we declare a dictionary to save the parsing results. Note that we need to convert `timestamp` and `application` to list with `asList()` because they are a combination of several entities from the log entry. For example, `timestamp` is a combination of `month`, `day`, and `hour`. This method returns the parser results.

```
	def __get_fields(self, log_line): 
		# parsing
		parsed = self.authlog_grammar.parseString(log_line)
		
		# get each field
		parsed_log = dict()		
		parsed_log['timestamp'] = ' '.join(parsed.timestamp.asList())
		parsed_log['hostname'] = parsed.hostname
		parsed_log['application'] = ' '.join(parsed.application.asList()) 
		parsed_log['message'] = parsed.message
		
		return parsed_log
```

The previous two methods are private methods identified by `__` at the beginning of the name. We need one more method named `parse_authlog` to read each line in the given log file and then call the `__get_fields` methods. We add an error-checking with `FileNotFoundError` exception just in case the user input the wrong file name.

```
	def parse_authlog(self):
		try:
			with open(self.log_file, 'r') as f:
				for line in f:
					result = self.__get_fields(line)
					print(result)
		
		except FileNotFoundError:
			print('File not found.')
			sys.exit(1)
```
The last piece of code is to call the class `AuthLogParser` that we have written before. To execute the parsing process, we call the method `parse_authlog`. The user can write the input file and we simply catch it with `sys.argv`. If the user does not provide any file, it will print an error message and the program will exit.

```
if __name__ == '__main__':	
	if len(sys.argv) == 2:
		file_name = sys.argv[1]
		parser = AuthLogParser(file_name)
		parser.parse_authlog()
	
	else:
		print('Please type a correct log file name.')
		sys.exit(1)
```

The full source code is available [here](https://gist.github.com/studiawan/33845032e0a15e5d2a100626033f37ef). To run the code, you run the command:

`python log-parser.py auth.log`

Make sure that the path of the log file is correct. The sample output is shown below. The output is presented in a dictionary where the key is the field name, so the user can easily identify each field for further analysis of the log file. Note that we use `dict` data type, which means the field is not ordered based on the first it appears. If you want the ordered value in a dictionary, there is `OrderedDict` in Python. For more information about arguments, you can check [Python Command Line Arguments](https://realpython.com/python-command-line-arguments/) tutorial from Real Python.

```
{'hostname': 'ip-172-31-27-153', 'application': 'CRON 21882', 'timestamp': 'Nov 30 06:39:00', 'message': 'pam_unix(cron:session): session closed for user root'}
{'hostname': 'ip-172-31-27-153', 'application': 'CRON 22087', 'timestamp': 'Nov 30 06:47:01', 'message': 'pam_unix(cron:session): session opened for user root by (uid=0)'}
{'hostname': 'ip-172-31-27-153', 'application': 'CRON 22087', 'timestamp': 'Nov 30 06:47:03', 'message': 'pam_unix(cron:session): session closed for user root'}
```

## Saving parser results to a CSV file

We can save the parsing results to a CSV file for further analysis. We need to modify `parse_authlog` method to save the results:

```
	def parse_authlog(self):
		try:
			# open csv file
			f = open(self.log_file + '.csv', 'wt')
			writer = csv.writer(f)
			writer.writerow(['timestamp', 'hostname', 'application', 'message'])
			
			with open(self.log_file, 'r') as f:
				# read each line
				for line in f:
					# parse a line
					result = self.__get_fields(line)
					
					# write to csv file and print
					writer.writerow([result['timestamp'], result['hostname'], result['application'], result['message']])
					print(result)
			
			# close csv file
			f.close()
		
		except FileNotFoundError:
			print('File not found.')
			sys.exit(1)
```

First, we prepare a CSV file to save the parsing results. The first row we write is a header label for each field. After getting the parsing results, we save to CSV file with `writer.writerow()`. Note that we need to convert parsing results into a list. Finally, we close the CSV file. We can run the code with the same command: `python log-parser.py auth.log`.

## Conclusion

In this tutorial, we learn to write a parser for a log file, especially auth.log file using PyParsing. It offers more readability of the parser. We can extend this parser to another type of log files, such as syslog, web server logs, and so on. PyParsing is definitely easier to read than regex. However, in terms of runtime, it is a little bit behind regex because more time is needed to interpret the grammar. If you want to write a code that easier to maintain, then PyParsing is a good choice. However, if the performance is more critical than readability, regex can provide faster execution. 

## Further reading

* PyParsing GitHub repository [https://github.com/pyparsing/pyparsing](https://github.com/pyparsing/pyparsing)
* PyParsing documentation [https://pyparsing-docs.readthedocs.io/en/latest/](https://pyparsing-docs.readthedocs.io/en/latest/)
