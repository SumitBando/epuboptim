# epuboptim - Optimize/compress EPUB files

`epuboptim` is a CLI tool to optimize existing EPUB files. It does so by utilizing 3rd party tools like `jpegoptim`, `pngoptim` etc.

# Why?

EPUBs are often very large, and handheld readers have limited ersources (disk, CPU, RAM). EPUB files are just zip files, so recompressing
them is not too hard.

# Features?

* Configurable recompression tools
* E.g. by mapping file extensions to tools
* Rule based syntax, rules are processed one after the other.
  * .jpg, .jpeg => "jpegtran -copy none -optimize -progressive -maxmemory 100M -outfile {{file}}"
* configuration in a JSON file (yes, just plain json...)
* Highly parallel, task based
  * A "find" task finds epub files, creates tasks to unzip them
  * unzip task uncomresses zip, creates tasks based on the configurable rules (jpg recompressions, PNG, ...)

# How to paralleize?

The goal should be 
* Always use all CPU resources
  * For single epub files (recompress the individual files in parallel, biggest files first)
  * process multiple epub files in parallel (but try to finish one book before working on the next)

How?
* Let's keep it simple. Create a single queue of tasks, probably a `queue.LifoQueue` https://docs.python.org/3/library/queue.html
* Find all epubs, and push a "unzip" task to the queue.
* Workers take an item from the queue, process it, and create one, zero, or multiple work items they push back onto the queue.
  * "unzip" task unzips, sorts files by size as a heuristic, then creates "recompress" work items of each file, pushes these to the queue
  * "recompress" tasks have a shared atomic counter, so once the last "recompress" task of a book is done, it creates a new task to "rezip" the book.
  * "zip" task compresses the minimized files into a new .epub file and replaces the old one.
