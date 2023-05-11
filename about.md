## About This Guide

**Task Based:** TAP exists of dozens of individual components playing together to build a great developer experience. While much of the information available on the internet goes into great detail on individual components it is often hard to instructions to solve a certain problem or task across the entire platform. This guide will always assume a certain task and provide step-by-step instructions to execute that across all of TAP.

**Single Page:** Instead of modularizing guides, we strive to provide all instructions top to bottom on a single page. This comes at the expense of duplication for us, but with a huge benefit for the reader especially for newcomers to TAP.

**Blind Flight:** All guides strive to allow blind flight. This means, they provide all instructions in a way that allows simple copy & paste from top to bottom and will complete successfully. This way, readers don't have to understand what is happening before they do it but get results quickly.

**Opinionation:** The guide provide instructions for very specific configurations and architectures we think will commonly be found. We do not try to document every possible combination situations. Example: When installing TAP on GCP, we will use GCP's container registry (GCR) and assume open connectivity to the internet. Using on onsite Artifactory and traffic tunneled through an internal proxy, will not be considered as the architecture is considered untypical atm.

**Validation:** Especially as a newcomer to any complex system, not knowing what to expect after running through a guide, tutorial or howto is a big problem. You execute the steps but how do you validate it works as it should at this stage? We finish every guide with a "Validation" section that shows you what it is that should work and what that looks like.