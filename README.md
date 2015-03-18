# Algorithms-as-a-Service

Cloud'N'Sci Ltd has created a marketplace for data refining services based on the Algorithms-as-a-Service concept. The [Cloud'N'Sci.fi marketplace](http://cloudnsci.fi "The Cloud'N'Sci.fi marketplace") will be available at [http://cloudnsci.fi](http://cloudnsci.fi "The Cloud'N'Sci.fi marketplace") for anyone who wants to commercialize or utilize smart data refining algorithms, data sets or data refining applications. The marketplace will be opened for public use later in 2015 (currently serving only pilot projects). 

Guides for Cloud'N'Sci.fi marketplace users and developers will be published in this GitHub project at [https://github.com/CloudNSci/cloudnsci-guides](https://github.com/CloudNSci/cloudnsci-guides).

## Data Refinery

Data refining algorithms made available via the [Cloud'N'Sci.fi marketplace](http://cloudnsci.fi "The Cloud'N'Sci.fi marketplace") are called "refineries". Each refinery should implement a data refining step with well-specified input and output data. The Cloud'N'Sci.fi marketplace provides means for making data refining requests to refinery modules and charging for their usage. Requests can be made manually from the marketplace web user interface or by external applications via the marketplace Application Programming Interface (API). Moreover, the marketplace makes it possible to build larger "algorithm architectures" wherein one refinery can make subrequests to another. The Cloud'N'Sci.fi marketplace takes care of hosting refinery processes and delivering input/output data to/from refineries.

One of the key design principles of the Cloud'N'Sci.fi marketplace has been that developers must be able use their favorite programming language and software libraries when implementing new refineries. Therefore, refineries are integrated to the marketplace as independent processes without any marketplace related source code dependencies. Refinery processes are executed in secure sandboxes that are completely isolated from the rest of the world (unless explicitly granted access to some external resources). Refineries can communicate only with the Cloud'N'Sci.fi marketplace by using the [Cloud'N'Sci Refinery Protocol (CNS-RP)](CloudNSci-Refinery-Protocol-1_0.md "The Cloud'N'Sci.fi refinery protocol version 1.0") described in document [CloudNSci-Refinery-Protocol-1_0.md](CloudNSci-Refinery-Protocol-1_0.md). 

(... more guides will be published later ...)

----------
  
Copyright (c) Cloud'N'Sci Ltd 2015, [http://cloudnsci.fi](http://cloudnsci.fi).
