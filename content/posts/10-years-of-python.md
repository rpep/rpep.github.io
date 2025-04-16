---
title: "10 years of Python - a personal retrospective from 2015 to 2025"
date: 2025-04-16T20:35:58+00:00
draft: false
featured_image: '/images/python-rainbow.png'
---

I first started touching Python while coming towards the end of a Master's project, about ten years ago. Much has changed over that period, and looking back, it's kind of amazing how far we've come. Python has it's fair share of critics out there on the internet, and obviously no tool exists in a vacuum, so many of the comparisons and criticisms are warranted as other languages and tooling have come along, but personally, I still find it incredibly powerful.

I thought it might be a nice exercise to show how much things have changed for the better over that period by picking out some specific evolutions that have happened over that time period.

## A scientific/data language to THE scientific/data language

At the time I started picking up Python, I was relatively comfortable with MATLAB, had picked up Mathematica for my master's degree project to do some gnarly analysis of counts of nearest neighbours in lattices and was pretty comfortable working on C++ code, all of which I'd picked up during my Physics degree. I'd managed to get a student license for Mathematica, but it wasn't cheap, and I'd been accepted for a PhD place where I knew the lab mostly used Python, so I started picking it up after my final University exams were finished. For me, it was almost love at first sight. It had much of the mathematical stuff that MATLAB did, symbolic maths with SymPy like Mathematica (although SymPy did lag behind in performance and features at the time), you could make it fast by combining it with C. Still, it was not exactly universally accepted - most of the researchers in the Engineering department I joined still used MATLAB aside from my small research group, lots of undergraduate teaching was still in MATLAB, the statistics course I did during my PhD was in R. It was one choice of several.

Nowadays, I think it would be pretty difficult to argue against the fact that Python has totally come to dominate this space. To try and attempt the same sorts of projects today as I worked on then in other languages is really hard to justify - it's certainly not impossible, but gaps in tooling and libraries are hard, and the mindshare of development of methods is happening today in Python. Library authors have been incredibly novel in working around shortcomings in the core language and CPython interpreter design around performance for scientific and data problems, with:

* Numba providing JIT compilation of numerical Python code to LLVM under the hood, often delivering near-C speeds without leaving Python syntax, often just by adding annotations.
* TensorFlow, PyTorch, and JAX all use lazy evaluation and static/dynamic graph compilation to shift real work onto highly optimized backends.
* Cython, PyBind11, and increasingly Rust via PyO3 allow developers to write performance-critical components outside Python and expose them cleanly to the high-level API.
* Dask and Ray provide distributed or parallel abstractions without requiring users to change how they write Python.
* Libraries as a result are getting faster and in some cases have inspired more performant alternatives - for e.g. in the case of Pandas and Polars.

As a result, the library ecosystem in the language has just got better and better in both functionality and performance terms. Python is now just part of the furniture.

## ... but great for glue code, less so for other things

On the flipside of the above, while Python has become really popular for data, it does feel to me that Python’s share in development of general-purpose backend services appears relatively flat compared to trends in Go, Rust, or Node over the same period. Flask and FastAPI work really well for exposing scientific code and ML models which are difficult to write in other languages, but my feeling is that as a general purpose backend language, it's not really grown that much, largely due to performance and scaling concerns. Async support has helped but not fixed performance issues, particularly because support is mixed and it requires a rethinking and rearchitecting of libraries to be asynchronous.

Personally, I do think this is a shame - I don't think I've ever been as productive writing APIs with other tooling as with Django. I'm interested to see whether ongoing performance improvements in the CPython interpreter (e.g. via the Faster CPython initiative) can reduce this gap and encourage broader adoption.

## Packaging

![packaging](/images/packaging-struggle.png)

Everyone loves to hate on Python's story around packaging, and it certainly isn't faultless today, but things have really come a long way. When I first started using the language, the default assumption was still really that you were running on Linux, you had installed the interpreter through a system package manager, and many packages, particularly in the scientific computing ecosystem, were really really difficult to install.This architectural assumption led to issues with reproducibility, version skew, and cross-platform portability. - package manager versions of things lagged a long way behind the latest releases. The reason for this was largely that compiled external C and Fortran libraries were required as dependencies. Largely, the story looked like how many packages in the R ecosystem still are today - bundled source code, and the expectation that external libraries were provided by your operating system or yourself, and were in the appropriate paths for the compiler to pick up. This was far from a seamless experience. If you were on Windows, there was basically no hope.

As a result, various bundled sets of working packages had been created. The first I used was Enthought Canopy, but Conda became more popular after and eventually won out, especially once Conda Forge allowed user contributed packages. I don't think that it's fair to say that it "won" the packaging war in Python - the commercial licensing made that impossble -  but it forced a lot of growth in the core Python packaging ecosystem, with many popular libraries now bundling compiled binary versions of dependencies.

[PEP513](https://peps.python.org/pep-0513/) which introduced the 'manylinux' environment in particular was really important, since it gave a 'base' ecosystem for package authors to target when providing binary wheels on Linux. Before this, you couldn't pick a wheel up and expect it to work on any particular Linux distribution, and as such no binary wheels existed on PyPi. 

On top of this, we've seen big changes in the way you write packaging in the last few years. It used to be the case that many packages had a bespoke `setup.py` script utilising distutils or later setuptools which ran arbitrary code at install time. These were often fragile, relied on injected parameters from environment variables, and frequently didn't work on other people's systems. First `setup.cfg` introduced declarative package metadata, and `pyproject.toml` decoupled the build from the metadata, and made it much easier to package up libraries. 

Today, I think it's fair to say that there's been a bit of an explosion of competing tooling in the space, with `poetry` and `uv` becoming swiftly the most popular for both managing source code and package dependencies when writing a new library, and  as tooling for working with Python in general. Still, Python packaging remains a really complex and story. Compared to other ecosystems — Rust with Cargo, Go with modules, or Node —Python’s dependency model is almost uniquely hard because of the areas it's popular in. The reasons are structural:
* Python wraps C, C++, Fortran, and now increasingly Rust.
* Scientific libraries often depend on platform-specific binaries or external system packages.
* GPU acceleration (CUDA, ROCm, etc.), and common use of the language on other platforms (x86_64, ARM, even POWER9 in one of my roles) introduces a hardware axis to dependency resolution which package authors have to deal with when publishing their code.
* Additionally, the Python ecosystem lacks a centralized, opinionated build system (like Cargo in Rust), leading to fragmentation across tools like setuptools, poetry, flit, and uv.

I do tend to think that first three of these constraints make comparisons with other ecosystems somewhat unfair, even though I've had my fair share of dealing with packaging and installation problems. With all that said, the experience to day is a world away from where it was in 2015, which has made it a much better experience for developers, particularly if you're on Windows.

![DEVELOPERS](/images/steve-ballmer.gif)

## Language convergence - the Python 2 to 3 transition

When I started writing Python, it was all Python 2. It wasn't that Python 3 hadn't been released 10 years ago - it was nearly 7 years old at that point. The issue was that many packages just didn't work with it. In April 2014 this had largely been recognised, and the end of life date for Python 2 got pushed back by five years to 2020.

The painful problem I experienced was basically the chain of dependencies you used needing to do the migration before you could. Many library authors took a decent amount of time refactoring to make things work in Python 3, and often this also caused interface changes and new major version releases. And so it really became a multifaceted problem for any complex piece of software - one day, you decide all the dependencies you used had migrated, and if you were lucky you didn't have major changes to make around binary/str handling, but you still had to deal with all of your libraries being upgraded, all at once. On top of that, you still ended up having to support users with Python 2 until the EOL came around, and this meant the use of libraries like `six` and `future` for maintaining compatibility, which later had to be cleaned up.

Like with Perl 4 to Perl 5, I think that the Python 2 to 3 transition is a cautionary tale in how-not-to-do a backwards-incompatible evolution of programming languages.

# IPython to Jupyter

![ipython to jupyter](/images/ipython-to-jupyter.png)

Notebook interfaces weren't alien to me having come from using Mathematica, but because of that software's mathematical focus, they hadn't achieved mainstream usage outside specific academic domains. Coming to Python, I thought IPython notebooks were a great extension of the ideas Mathematica had - though at the time there were at that time still major gaps between IPython's capabilities, particularly around interactivity and widgets for things like plotting.

In the 10 years since, there's been a huge leap forwards - first from seperating the frontend of Jupyter from the language independent kernels, of which there are many. Then subsequently, the introduction of Jupyter Lab makes the browser a full IDE experience. Notebooks are now de-facto the exchange format for machine learning practitioners, and are commonly used for pedagogical material like lecture notes and even books.

## Typing - unfinished business?

The introduction of typing to Python seems to divide people into two camps - those who don't like it and feel like it defeats the point of using the language, and those that really feel it needs it. I probably fall more towards the latter camp, but I certainly don't think that it's sufficient if you're a library author since you still need to guard against bad inputs. Part of the problem is that because it's not totally accepted across Python users and developers, and there is lack of support in many major libraries (Django, which uses lots of introspection at runtime, being the largest popular library without first party support). It is therefore difficult on many projects to go 'all in', which leads to useful type annotations being sprinkled in, but often not enforced by tooling. If you do try and utilise the type checker, you can often find yourself in a bit of a battle with the type checker.

At the same time, some popular libraries have really driven forwards the typing story, though more in the general Python development sphere than on the scientific/ML side. Pydantic and FastAPI especially are strongly reliant on type annotations and provide an elegant way of writing APIs in Python as a result. Nonetheless, it does feel to me like this is one area where we have come a long way, but there is still a story to be written.

## Conclusion

This isn't a totally comprehensive summary of the changes - it's mostly been borne out by what I've seen in the areas I've worked - particularly in scientific computing, API development and in data pipelines. It feels to me that Python has gone from being a bit of a plucky upstart to become a clear mainstream first choice for many when starting a new project, even in spite of some rough edges, particularly around packaging. After 10 years, I still find Python a fun language to use, and while I've picked up new languages like Go in that time as well, for anything personal, I'd still largely jump to it since I find it expressive, capable and easy to work with, with great libraries.
