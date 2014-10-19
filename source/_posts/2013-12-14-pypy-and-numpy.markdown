---
layout: post
title: "pypy and numpy"
date: 2013-12-14 4:48:53 -0500
comments: true
categories: Python, CS
---
Pypy and numpy have always been trying to get along well. On one hand, numpy is one of the best scientific libraries in the Python world. And on the other hand, pypy improves the efficiency of the execution significantly.  

I happened to start to switch from naive python interpreter to pypy due to some efficiency issue, and the performance boosting is tremendous. However, it took me some time to figure out how to make numpy work with pypy. Turns out the pypy dev team has just removed the numpypy(which is one of the numpy version in pypy) out of the code base. So we have to install the numpy as an individual module from the source provided by pypy.   

If you are using virtualenv, which is a good practice:  
```
    pip install git+https://bitbucket.org/pypy/numpy.git
```

Or directly download the fork and install:  
```    
    git clone https://bitbucket.org/pypy/numpy.git  
```  

```
    cd numpy  
```  

```
    pypy setup.py install  
```

Note that the official numpy module cannot work with pypy, so you have to use the fork by pypy dev team.  
You can always check <http://buildbot.pypy.org/numpy-status/latest.html> to see which functions and models can be used within pypy. Great applause to pypy dev team. They are really making big progresses.