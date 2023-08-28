---
layout: post
title: Wrapping it up! | GSoC'23 @ CERN-HSF 
author: Yash Solanki
categories: Tech
permalink: /gsoc-submission
---

## <u> Table of Contents </u>
1. [Overview](#-overview-)
2. [Warming up - Fixing bugs for non-existent objects](#-warming-up---fixing-bug-for-non-existent-objects-)
3. [Creating objects using Strings](#-creating-objects-using-strings-)
4. [Creating objects using Python Dictionaries](#-creating-objects-using-python-dictionaries-)
5. [Refactoring code to C++ backend](#-refactoring-code-to-c-backend-)
6. [Making the JSON I/O key management automatic](#-making-the-json-io-key-management-automatic-)
7. [Importing datasets, modelconfigs and a new tutorial!](#-importing-datasets-modelconfigs-and-a-new-tutorial-)
8. [Future goals - tutorials, multi-channel data, operators and more!](#--future-goals---tutorials-multi-channel-data-operators-and-more-)
9. [Conclusion, difficulties, and learning outcomes](#-conclusion-difficulties-and-learning-outcomes-)

## <u> Overview </u>

CERN's ROOT is an open source data anaysis framework that is used for High Engergy Physics applications. PyROOT, the ROOT's python interface, allows users to access all functions of ROOT using a cppyy backend. However, to interact with RooFit, the users needed to use C++-like strings even in Python, hence, it needed 'Pythonizations' to make the experience more intuitive and easier. 

As a part of my GSoC project- "RooFit - Pythonic interaction with RooWorkspace"- I had to write Pythonizations to by writing C++ and Python plugins, so that RooWorkspace objects can be handled using Pythonic syntax using a JSON I/O backend called HS3. 

# <u> Warming up - Fixing bug for non-existent objects </u> 

Earlier, accessing objects from the RooWorkspace such as P.D.F.s, variables and datasets that were not already present in the `RooWorkspace` only printed an error, however, the program continued to run. This might lead to segmentation faults and confusing errors down the lane. 

I fixed this by throwing an appropriate `std::runtime_error`. Now, if you access invalid objects, the program terminates too.

[Link to Pull Request - #12860](https://github.com/root-project/root/pull/12860)

# <u> Creating objects using Strings </u> 

Root has the functionality to create objects using a `factory expression`, which is a C++-like string. Using such strings in Python is not very intuitive for the users.

As a first step for the Pythonization, I added a backend to create the factory expressions with a key-value, which has the object name as its key and an initialization string as its value.

So, to make a Gaussian p.d.f., the code

```python
import ROOT

ws = ROOT.RooWorkspace("ws")
ws.factory("Gaussian::gauss(x[0.0, 10.0], mu[5.0], sigma[2.0, 0.01, 10.0])")
```

gets replaced by 

```python
import ROOT

ws = ROOT.RooWorkspace("ws")

ws["gauss"] = "Gaussian(x[0.0, 10.0], mu[5.0], sigma[2.0, 0.01, 10.0])"
```

[Link to Pull Request - #12911](https://github.com/root-project/root/pull/12911)

# <u> Creating objects using Python Dictionaries </u> 

The next step in Pythonization of RooWorkspace was to create object using dictionaries. RooFit already had a JSON I/O backend called HS3. Hence,I had to create parse the key-value pair in correct format from Python and pass it to the HS3 backend in C++. This was followed by creation of a JSON tree which was used to import the objects into the workspace.

For example -

```python
import ROOT

ws = ROOT.RooWorkspace("ws")

ws["m1"] = dict({ "max": 5, "min": -5, "value": 0 })
ws["m2"] = dict({ "max": 5, "min": -5, "value": 1 })
ws["mean"] = dict({ "type":"sum", "summands":["m1", "m1"] })
```

Creating of JSON dictionary was still done on the Python side, which was fixed in the next PR.

[Link to Pull Request - #12994](https://github.com/root-project/root/pull/12994)

# <u> Refactoring code to C++ backend </u> 

In the previous pull request, most of the operations with string were done on the Python side. However, we wanted the Python objects to be a wrapper around the C++ functions. So, the logic to process string in the correct format was passed to the C++ side, where the `JSONTree` of HS3 was used to create a JSON object that follows the HS3 standard.

[Link to Pull Request - #13150](https://github.com/root-project/root/pull/13150)

# <u> Making the JSON I/O key management automatic </u> 

Creating new objects in the JSON objects is done in RooWorkspace with `factory expressions`. However, these expressions were not automatically imported to the RooWorkspace when it gets created. This was fixed by setting up default factory expressions whenever a new Workspace is created.

[Link to Pull Request - #13152](https://github.com/root-project/root/pull/13152)

# <u> Importing datasets, modelconfigs and a new tutorial! </u> 

The earlier pythonizations allowed the creation of objects, however, importing datasets was still done using JSON Strings. This pull request added the feature to import both `binned` and `unbinned` datasets to a RooWorkspace using a dictionary. 

For example - 

```python
ws["asimovData_channel1"] = {
    "axes": [{"max": 2.0, "min": 1.0, "name": "obs_x_channel1", "nbins": 2}],
    "contents": [120, 110],
    "type": "binned",
}
```

Along with this, pythonizations for making and fitting models directly from the workspace(instaed of creating a `Measurement` object) and importing `modelConfig` for specifying properties of fitting was added.

With all these changes, we can now can now make models and fit to the data in a completely Pythonic way! I also made a new tutorial to demonstrate this for single channel models.

<details>
<summary>An example of Pythonic workflow using RooWorkspace</summary>

<pre>
<code data-lang="python">
import ROOT

ws = ROOT.RooWorkspace()

ws["Lumi"] = dict({"max": 10.0, "min": 0.0, "value": 1.0})
ws["nominalLumi"] = dict({"max": 2.0, "min": 0.0, "value": 1.0})

ws["lumiConstraint"] = {
    "mean": "nominalLumi",
    "sigma": 0.1,
    "type": "gaussian_dist",
    "x": "Lumi",
}

ws["obsData_channel1"] = {
    "axes": [{"max": 2.0, "min": 1.0, "name": "obs_x_channel1", "nbins": 2}],
    "contents": [122, 112],
    "type": "binned",
}

ws["asimovData_channel1"] = {
    "axes": [{"max": 2.0, "min": 1.0, "name": "obs_x_channel1", "nbins": 2}],
    "contents": [120, 110],
    "type": "binned",
}

ws["model_channel1"] = {
    "axes": [{"max": 2.0, "min": 1.0, "name": "obs_x_channel1", "nbins": 2}],
    "samples": [
        {
            "data": {"contents": [100, 0], "errors": [5, 0]},
            "modifiers": [
                {
                    "constraint_name": "lumiConstraint",
                    "name": "Lumi",
                    "parameter": "Lumi",
                    "type": "normfactor",
                },
                {
                    "constraint": "Gauss",
                    "data": {"hi": 1.05, "lo": 0.95},
                    "name": "syst2",
                    "parameter": "alpha_syst2",
                    "type": "normsys",
                },
                {"constraint": "Poisson", "name": "staterror", "type": "staterror"},
            ],
            "name": "background1",
        },
        {
            "data": {"contents": [0, 100], "errors": [0, 10]},
            "modifiers": [
                {
                    "constraint_name": "lumiConstraint",
                    "name": "Lumi",
                    "parameter": "Lumi",
                    "type": "normfactor",
                },
                {
                    "constraint": "Gauss",
                    "data": {"hi": 1.05, "lo": 0.95},
                    "name": "syst3",
                    "parameter": "alpha_syst3",
                    "type": "normsys",
                },
                {"constraint": "Poisson", "name": "staterror", "type": "staterror"},
            ],
            "name": "background2",
        },
        {
            "data": {"contents": [20, 10]},
            "modifiers": [
                {
                    "constraint_name": "lumiConstraint",
                    "name": "Lumi",
                    "parameter": "Lumi",
                    "type": "normfactor",
                },
                {
                    "name": "SigXsecOverSM",
                    "parameter": "SigXsecOverSM",
                    "type": "normfactor",
                },
                {
                    "constraint": "Gauss",
                    "data": {"hi": 1.05, "lo": 0.95},
                    "name": "syst1",
                    "parameter": "alpha_syst1",
                    "type": "normsys",
                },
            ],
            "name": "signal",
        },
    ],
    "type": "histfactory_dist",
}

ws["simPdf_asimovData"] = {"pdfName": "model_channel1", "poi": "SigXsecOverSM"}

ws.Print()

ROOT.RooStats.HistFactory.FitModelAndPlot("meas", "tut", "obsData_channel1", ws)
</code>
</pre>

</details>


[Link to Pull Request - #13553](https://github.com/root-project/root/pull/13553) 

# <u>  Future goals - tutorials, multi-channel data, operators and more! </u>

There were a few more milestones mentioned in my propsal that I couldn't deliver because more additions to the `RooJSONFactoryWSTool` were required. Currently, the support for multi-channel datasets and simultaneous p.d.f.s needs to be pythonized. I'll do these in the next few weeks and create another tutorial for this use case.

Other than that, we also want to support operators for the objects of RooWorkspace. For example, to add p.d.f.s we could use '+', and for convolution we could use '**'. 

# <u> Conclusion, difficulties, and learning outcomes </u>

It was a great experience to work for the ROOT team. The major challenge that I faced was understanding and debugging issues in a large codebase. I learned a lot of advanced C++ while writing plugins for the ROOT. After completing the pending goals, I would like to continue contributing to ROOT project.

I would like to thank my mentors **Jonas Rembser** and **Lorenzo Moneta** for supporting me through the GSoC journey. They were very helpful and polite and whenever I got stuck they helped me navigate through possible solutions.  

Until next time!

Yash 