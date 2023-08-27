---
layout: post
title: Wrapping it up! | GSoC'23 @ CERN-HSF
author: Yash Solanki
categories: Tech
permalink: /gsoc-submission
---

### Overview:

RooFit - Pythonic interaction with the RooWorkspace

# Table of Contents
1. [Warming up - Fixing bugs for non-existent objects](#warming-up---fixing-bugs-for-non-existent-objects)
2. [Creating objects using Strings](#creating-objects-using-strings)
3. [Creating objects using Python Dictionaries](#creating-objects-using-python-dictionaries)
4. [Refactoring code to C++ backend](#refactoring-code-to-c-backend)
5. [Avoiding unnevessary imports of I/O keys!](#avoiding-unnevessary-imports-of-io-keys)
6. [Importing datasets, modelconfigs and new tutorial!](#importing-datasets-modelconfigs-and-new-tutorial)
7. [A summary - What's new for the users?](#a-summary---whats-new-for-the-users)
8. [Future goals - tutorials, multi-channel data, operators and more!](#future-goals---tutorials-multi-channel-data-operators-and-more)

# <u> Warming up - Fixing bugs for non-existent objects </u> 

Earlier, accessing objects from the RooWorkspace such as P.D.F.s, variables and datasets that were not already present in the `RooWorkspace` only printed an error, however, the program continued to run. This might lead to segmentation faults and confusing errors down the lane. 

I fixed this by throwing an appropriate `std::runtime_error`. Now, if you make access invalid objects, the program terminates!

[Link to Pull Request - #12860](https://github.com/root-project/root/pull/12860)

# <u> Creating objects using Strings </u> 

Root has the functionality to create objects using a `factory expression`, which is a C++-like string. Using such strings in Python is not very intuitive for the users.

As a first step for the Pythonization, I added a backend to create the factory expressions with a key-value, which has the object name as its key and an initialization string as its value.

So, to make a Gaussian p.d.f., the code

```python
import ROOT

ws = ROOT.RooWorkspace("ws")
"Gaussian::gauss(x[0.0, 10.0], mu[5.0], sigma[2.0, 0.01, 10.0])"
```

gets replaced by - 

```python
import ROOT

ws = ROOT.RooWorkspace("ws")

ws["gauss"] = "Gaussian(x[0.0, 10.0], mu[5.0], sigma[2.0, 0.01, 10.0])"
```

[Link to Pull Request - #12911](https://github.com/root-project/root/pull/12911)

# <u> Creating objects using Python Dictionaries </u> 

```python
import ROOT

ws = ROOT.RooWorkspace("ws")

ws["m1"] = dict({ "max": 5, "min": -5, "value": 0 })
ws["m2"] = dict({ "max": 5, "min": -5, "value": 1 })
ws["mean"] = dict({ "type":"sum", "summands":["m1", "m1"] })
```

[Link to Pull Request - #12994](https://github.com/root-project/root/pull/12994)

# <u> Refactoring code to C++ backend </u> 

[Link to Pull Request - #13150](https://github.com/root-project/root/pull/13150)

# <u> Avoiding unnevessary imports of I/O keys! </u> 

[Link to Pull Request - #13152](https://github.com/root-project/root/pull/13152)

# <u> Importing datasets, modelconfigs and new tutorial! </u> 

[Link to Pull Request - #12860](https://github.com/root-project/root/pull/12860)

# <u> A summary - What's new for the users? </u> 



# <u>  Future goals - tutorials, multi-channel data, operators and more! </u> 


