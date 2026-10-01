# AI Use Disclosure

**Tools used:** 
Codex

**What I used them for:**
I used Codex to get started with an initial implementation of the code for data collection/cleaning, sentiment sorting, return construction, charting, and one-lag Granger tests. I also used it to troubleshoot API, data parsing, and notebook execution errors.

**What I wrote myself:**
I implemented and set up the Python code in the notebook for outputting the required tables and figures. I also reviewed the code and associated outputs that Codex wrote in the notebook to ensure that assignment requirements were met, and I wrote out the interpretations of the tables and figures in the report. 

**Anything the model got wrong that I had to correct:**
The initial report values only reflected a prior execution with 14 aligned trading observations, so they had to be replaced with updated 16-observations results. In addition, 
there were some other public social sources that the model thought had available data (i.e. Bluesky) but in reality did not have data available (return HTTP 403 errors) so those implementations could not be included.  
